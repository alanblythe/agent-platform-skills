---
name: agent-platform-state
description: >
  This skill should be used when deciding where data lives on Gemini Enterprise
  Agent Platform, across Agent Platform Sessions and Agent Platform Memory Bank:
  "how do I persist X", "does this survive a restart", "should I use Sessions or
  Memory Bank", "can agents share state across users", "how do I share memories
  across users", "why is my state empty in a new conversation", "what does app:
  or user: prefixed state do", "how do I list every memory in a scope", "why did
  my tool write to the wrong scope", or when a value written by a tool is missing
  on the next turn or the next instance. Names each managed service, gives its
  scope and lifetime, and maps it to the ADK client that binds to it. Use
  agent-platform-architecture for choosing a host or identity mode,
  agent-platform-a2a for auth failures.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.9.0
---

# State and memory on Agent Platform

Covers the two managed state services — **Agent Platform Sessions** and
**Agent Platform Memory Bank** — and the ADK clients that bind to them.
For **Agent Runtime** and **Agent Gateway**, see `agent-platform-architecture`.

Client-side claims describe ADK 2.6.x. API-resource claims come from the
generated Vertex SDK types, which are produced from the service schema.

## The services, by name

Agent Platform exposes two managed state services. They are separate products
with separate APIs, separate scoping rules, and separate lifetimes.

| Managed service | Stores | Scoped by | Bound from ADK by |
| :--- | :--- | :--- | :--- |
| **Agent Platform Sessions** | conversation history, events, and one state blob per session | a single session | `VertexAiSessionService` |
| **Agent Platform Memory Bank** | extracted or supplied facts, and structured profiles | an arbitrary developer-defined scope map | `VertexAiMemoryBankService` |

Everything else ADK offers is **not** an Agent Platform service. These run in
your own process or your own database, and the distinction is the single most
common source of confusion in this area:

| ADK class | Backed by | Managed by Agent Platform |
| :--- | :--- | :---: |
| `VertexAiSessionService` | Agent Platform Sessions | ✅ |
| `VertexAiMemoryBankService` | Agent Platform Memory Bank | ✅ |
| `InMemorySessionService` | the agent's own process | ❌ |
| `SqliteSessionService` | a file on the instance | ❌ |
| `DatabaseSessionService` | a database you operate | ❌ |
| `InMemoryMemoryService` | the agent's own process | ❌ |

Naming matters here because the two managed services have **opposite**
capabilities from the self-hosted ones (S6), so advice that is correct for
`DatabaseSessionService` is wrong for Agent Platform Sessions.

## Quick answers

| Question | Answer |
| :--- | :--- |
| Does a file written by a tool persist? | **No.** Container filesystems are ephemeral and per-instance on every managed host (S12) |
| Does `app:`-prefixed state work on Agent Platform Sessions? | **No.** `VertexAiSessionService` never splits prefixes; the key is stored literally inside one session (S2) |
| Is that an ADK gap? | **No — the service.** The Sessions resource has one state field, `session_state`, documented "Session specific" (S3) |
| Which session backends honour `app:` / `user:`? | The self-hosted ones only: Database, SQLite, InMemory. **Not** Agent Platform Sessions (S6) |
| Do Agent Platform Sessions live forever? | **No.** `expire_time` is always returned; minimum TTL 24 hours (S5) |
| Can Agent Platform Memory Bank share memories across users? | **Yes.** `scope` is an arbitrary map and need not contain a user key (S7) |
| How do I get a shared scope through ADK? | Pin `user_id` to a constant — ADK hardcodes it into every scope (S9) |
| Can I list every memory in a scope? | **Yes — `retrieve(simple_retrieval_params=…)`.** But not through ADK: `search_memory` only ever sends similarity params (S10) |
| Can a tool pick its own memory scope? | **No.** `ToolContext.add_memory` pins the session's ids — under A2A that is whichever agent called you (S16) |
| Will Memory Bank store my text verbatim? | Only via `add_memory`. The session and event paths run LLM extraction (S11) |
| Where do exact, enumerable records belong? | A constant Memory Bank scope reads back in full (S10) — but neither managed service is transactional, so anything needing atomicity or ordering belongs in an external datastore |

## Three stores, three lifetimes

Every persistence question resolves to which of these you are in.

| Store | Scope | Lifetime | Retrieval |
| :--- | :--- | :--- | :--- |
| **Container filesystem** (not a service) | one instance | until that instance is recycled | direct |
| **Agent Platform Sessions** | one session | TTL'd, minimum 24h | exact, by session |
| **Agent Platform Memory Bank** | an arbitrary scope map | until deleted or expired | similarity search, scope-keyed enumeration, or scope-keyed profiles |

The common mistake is reaching for the filesystem, discovering it is ephemeral,
and assuming Agent Platform Sessions is the fix. Sessions is durable, but it is
scoped to one conversation — which is a different problem from being ephemeral,
and is not solved by the same change.

## Forces

Rules, not preferences. `⇒` means the platform gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| S1 | ADK's `app:` / `user:` / `temp:` prefixes are a **client-side convention**, applied by `_session_util.extract_state_delta` splitting a flat dict into app/user/session deltas | ⇒ each session backend implements scoping itself; support is per-backend, not a framework guarantee |
| S2 | `VertexAiSessionService` — the Agent Platform Sessions client — imports `_session_util` but never calls `extract_state_delta` | ⇒ `app:`/`user:` keys are stored **literally** in one session's state ⇒ durable but siloed: a new session reads nothing back |
| S3 | The Agent Platform Sessions resource exposes exactly one state field — `session_state`, "Session specific memory which stores key conversation points". There is no `app_state` or `user_state` | ⇒ S2 is a limit of the service, not an ADK oversight; no client can scope session state above the session |
| S4 | `VertexAiSessionService.get_user_state` raises `NotImplementedError`, directing callers to enumerate sessions | ⇒ corroborates S3 — user-scoped state has nowhere to live in Agent Platform Sessions |
| S5 | Agent Platform Sessions carry `ttl` and `expire_time`; `expire_time` is *always* returned, and both have a 24-hour minimum | ⇒ session state is time-limited storage, never a system of record |
| S6 | Prefix support: `DatabaseSessionService` ✅, `SqliteSessionService` ✅, `InMemorySessionService` ✅, `VertexAiSessionService` ❌ | ⇒ the properties invert: the self-hosted backends scope correctly, the managed service is the one that cannot |
| S7 | Agent Platform Memory Bank's `scope` is `dict[str, str]` — "Required. Immutable… Memories are isolated within their scope… cannot contain the wildcard `*`". Documented examples include a scope with **no user key** | ⇒ cross-user shared collections are a supported pattern, not a workaround |
| S8 | Scope matching is exact across all keys, and consolidation happens only within one scope | ⇒ a shared scope is genuinely isolated storage, not a filter over per-user memories |
| S9 | `VertexAiMemoryBankService` builds `scope={'app_name': …, 'user_id': …}` as a literal at every call site, with no override parameter | ⇒ through ADK the **only** lever on scope is `user_id` ⇒ a constant sentinel user id addresses a shared collection; a custom key requires calling the Memory Bank API directly |
| S10 | Memory Bank has **three** read paths on `memories.retrieve`: `similarity_search_params` (semantic top-k), `simple_retrieval_params` — "the parameters for simple (non-similarity search) retrieval", paginated by `page_size`/`page_token` — and `retrieve_profiles`, "a scope-keyed lookup, not a semantic query" | ⇒ a scope **can** be enumerated exactly, without profiles ⇒ but `VertexAiMemoryBankService.search_memory` hardcodes `similarity_search_params`, so the enumerating path exists only outside ADK |
| S16 | `ToolContext.add_memory` passes `app_name=session.app_name, user_id=session.user_id` — "uses this callback's current session identifiers as memory scope" — with no override | ⇒ S9's constant-`user_id` lever is unavailable **from inside a tool** ⇒ and under A2A the session `user_id` is whichever caller invoked you, so a tool's writes scatter one scope per caller |
| S17 | `simple_retrieval_params.page_size` defaults to **3** and is coerced down to a maximum of 100; the response carries no truncation flag | ⇒ an unset page size silently returns 3 records and looks like a complete answer |
| S11 | Memory Bank has two write paths: `add_memory` → `memories.create` **verbatim**; `add_session_to_memory` / `add_events_to_memory` → generate/IngestEvents, which **LLM-extracts** facts. `enable_consolidation` merges server-side | ⇒ the default path for session ingestion paraphrases; only `add_memory` preserves text as written |
| S12 | Container filesystems on managed hosts are ephemeral, per-instance, and count against instance memory | ⇒ a file write is not persistence: lost on recycle, lost on scale-to-zero, invisible to sibling instances ⇒ concurrent readers can observe different values |
| S13 | `GOOGLE_CLOUD_AGENT_ENGINE_ID` is injected by **Agent Runtime**, not by Cloud Run | ⇒ the standard "Agent Platform Sessions if set, in-memory otherwise" factory **silently degrades** on Cloud Run: identical code, no error, ephemeral sessions ⇒ not a dead end: an engine can be created with no code and pointed at, see `agent-platform-architecture` F15 |
| S14 | `DatabaseSessionService` needs the `db` extra (SQLAlchemy) and builds via `create_async_engine`; dialects are `sqlite`, `mysql`, `postgresql` | ⇒ an async driver is required (`postgresql+asyncpg`, `mysql+aiomysql`, `sqlite+aiosqlite`), and the dependency is not present by default |
| S15 | SQLite and in-memory backends live inside the instance | ⇒ they do not survive multi-instance scaling ⇒ "shared across users" implies a network database or a managed service, not a file |

Verified by reading the client and generated SDK types: S1, S2, S4, S6, S7, S9,
S10, S11, S14, S16, S17. From documentation: S3 (field description), S5, S8.
Platform behaviour: S12, S13, S15.

**S2 and S3 are the same rule seen twice.** S2 is usually met as "my `app:` state
is empty in a new conversation" and read as an ADK bug. It is not: Agent
Platform Sessions has nowhere to put app-scoped state, so the client cannot
fabricate it. Reach for Agent Platform Memory Bank, not for a different prefix.

## Where prefixed state actually works

Rows are session backends; columns are the properties a shared store needs.

| Session backend | Managed service | `app:` / `user:` scoping | Survives restart | Shared across instances |
| :--- | :---: | :---: | :---: | :---: |
| `InMemorySessionService` | — | ✅ | ❌ | ❌ |
| `SqliteSessionService` | — | ✅ | ✅ | ❌ |
| `DatabaseSessionService` | — | ✅ | ✅ | ✅ (network DB only, S15) |
| **Agent Platform Sessions** | ✅ | ❌ | ✅ | ✅ |

**No managed option gives both.** Durable *and* app-scoped, within Sessions,
requires a database you operate. This is the central table of this skill:
everything below is a consequence of it. When app-scoped durable state is the
requirement and you want to stay on managed services, the answer is Agent
Platform Memory Bank, not Agent Platform Sessions.

## Agent Platform Sessions

### What the resource holds

Fields are `create_time`, `display_name`, `expire_time`, `labels`, `name`,
`session_state`, `ttl`, `update_time`, `user_id`.

`user_id` is an immutable identifier for filtering and listing — **not** a state
scope. A session belongs to a user; its state does not span that user's other
sessions.

### Silent degradation across hosts

The conventional factory shape:

```python
if agent_engine_id := os.environ.get("GOOGLE_CLOUD_AGENT_ENGINE_ID"):
    return VertexAiSessionService(...)      # Agent Platform Sessions
return InMemorySessionService()             # not a managed service at all
```

Agent Runtime injects that variable; Cloud Run does not. On Cloud Run this
returns the in-memory backend with no warning (S13), so the agent silently stops
using Agent Platform Sessions. The symptom appears later and elsewhere: state
vanishing between turns once more than one instance is serving. Log which branch
was taken at startup — a fallback that never announces itself is
indistinguishable from a configuration that worked.

Set `SESSION_SERVICE_URI` explicitly on any host that does not inject the
variable, or create an engine for the agent and point at it — an engine needs no
deployed code to hold Sessions and Memory Bank (`agent-platform-architecture`
F15).

## Agent Platform Memory Bank

### Scope is yours to define

Scope is an arbitrary immutable string map, not a fixed user pairing (S7).
Documented forms include a user key, a user-and-session pair, and a session key
alone — the last of which demonstrates that no user key is required. A scope
keyed by team, tenant, or topic is therefore a first-class collection.

Access to a scope is controlled with IAM Conditions on the memory-scope
attribute, which is the intended mechanism for governing who reads a shared
collection.

### Reaching a shared scope through ADK

`VertexAiMemoryBankService` exposes no scope parameter (S9). To share one
collection across users:

- **Through ADK** — pass a constant `user_id`. It is injected into every scope,
  so a fixed sentinel value addresses one shared collection.
- **Around ADK** — call the Memory Bank API's `memories.create` /
  `memories.retrieve` directly with the scope you want. Cleaner keys, at the
  cost of leaving the framework.

**The first option is not available from inside a tool.**
`ToolContext.add_memory` takes the scope from the session it is running in
(S16) — there is no argument for it. Pinning `user_id` therefore means
controlling how sessions are created, which an agent invoked over **A2A** does
not: the session's `user_id` is whichever agent called it, so a tool writing
through `ToolContext` scatters one scope per caller instead of accumulating one
shared collection. An agent that is both A2A-invoked and team-scoped has only
the second option.

Scope is **immutable** once a memory is written. Choose the key before writing
anything you intend to keep.

### Retrieval is three different operations

| Operation | Semantics | Reachable through ADK | Use for |
| :--- | :--- | :---: | :--- |
| `similarity_search_params` | similarity search over free-form facts, top-k | ✅ `search_memory` | recall — preferences, traits, context |
| `simple_retrieval_params` | scope-keyed enumeration of free-form facts, paginated | ❌ | listing a collection exactly |
| `retrieve_profiles` | scope-keyed lookup of structured profiles, one per registered schema | ❌ | exact, schema'd reads |

A scope **can** be enumerated exactly, without registering a profile schema —
`memories.retrieve(scope=…, simple_retrieval_params={"page_size": N})` is a
non-semantic listing. What is missing is not the capability but the ADK route to
it: `VertexAiMemoryBankService.search_memory` is the only read method the client
exposes, and it always sends similarity params (S10).

Set `page_size` explicitly. Unset it returns **3** records with nothing marking
the response as truncated (S17), which reads as a complete collection and is the
kind of bug that surfaces as an agent re-proposing something already taken.

### Writing preserves or paraphrases, depending on the path

`add_memory` writes verbatim through `memories.create`. The session and event
ingestion paths run LLM extraction, and consolidation resolves contradictions
and dedupes within the scope (S11). Both behaviours are correct for recall and
wrong for records: text that must survive unaltered goes through `add_memory`
with consolidation off.

### The team-scoped collection pattern

A shared, exactly-enumerable collection — team bookings, tenant config, a roster
— sits outside what the ADK memory client can express, on both halves. Write
with `memories.create` for a verbatim fact (S11), read with
`simple_retrieval_params` for a complete list (S10), and hold the scope as one
constant:

```python
TEAM_SCOPE = {"app_name": "sched_agent", "user_id": "team"}   # immutable: changing
                                                              # it orphans every record
await client.aio.agent_engines.memories.create(
    name=f"reasoningEngines/{engine_id}",
    fact="booking:" + json.dumps(record, sort_keys=True),
    scope=TEAM_SCOPE,
)

pager = await client.aio.agent_engines.memories.retrieve(
    name=f"reasoningEngines/{engine_id}",
    scope=TEAM_SCOPE,
    simple_retrieval_params={"page_size": 100},   # unset returns 3 (S17)
)
```

Prefix the fact so foreign memories in the same scope are skipped rather than
parsed. Resist splitting the halves — writing through ADK and reading around it
means two clients and two spellings of the scope for one collection, and the
scope is the thing that must never drift.

The engine holding this is selected by `GOOGLE_CLOUD_AGENT_ENGINE_ID`, which
also selects the **session** store — so give each agent its own engine rather
than sharing one (S13, `agent-platform-architecture` F15).

## Choosing

| What must persist | Across | Put it in |
| :--- | :--- | :--- |
| Conversation continuity | turns of one conversation | Agent Platform Sessions, or any backend |
| Conversation continuity | restarts | **Agent Platform Sessions** |
| Facts about one user | that user's sessions | **Agent Platform Memory Bank**, scope containing a user key |
| Facts shared by everyone | all users and sessions | **Agent Platform Memory Bank**, constant scope (S9) — around ADK if the agent is A2A-invoked (S16) |
| A shared collection read in full | all users and sessions | **Agent Platform Memory Bank**, constant scope + `simple_retrieval_params` (S10) |
| Structured, exactly-read values | all users and sessions | **Agent Platform Memory Bank** profiles (S10) |
| App-scoped state inside Sessions | sessions | not possible — use Memory Bank, or a database you operate (S2, S3) |
| Records needing atomicity or ordering | anything | external datastore |
| Anything at all | instance recycling | not the filesystem (S12) |

For an exact, enumerable, concurrently-appended ledger, a serverless document
store is the least infrastructure that satisfies it: no instance to provision,
no network path from the runtime, atomic appends, and IAM by role. A managed SQL
instance is the alternative and additionally makes `app:`-prefixed session state
work (S6) — worth it only when durable *sessions* are a goal in their own right,
since it adds a database, a network path, and a connector.

## Dead ends

- **`app:` state in Agent Platform Sessions to share across conversations.**
  Stored, never shared (S2, S3).
- **Moving hosts to fix data loss.** A different host changes which session
  backend is injected; it does not move a filesystem write into a session
  (S12, S13).
- **SQLite for multi-user state.** Per-instance, so scaling produces divergent
  copies that each vanish separately (S15).
- **Agent Platform Sessions as a system of record.** TTL'd with a 24-hour
  minimum (S5).
- **Free-form Memory Bank entries as a ledger.** Enumeration is possible
  (S10), but the ingestion paths rewrite facts and consolidation merges them
  (S11); nothing is transactional. Fine for a small shared collection written
  verbatim, wrong for anything needing atomicity or ordering guarantees.
- **`ToolContext.add_memory` for a shared scope.** Silently writes to the
  caller's scope under A2A (S16).
- **Reading `get_user_state` against Agent Platform Sessions.** Raises
  `NotImplementedError` by design (S4).

## Sources

- Agent Platform Memory Bank documentation: overview, generate memories, set up,
  API quickstart, ADK quickstart, and IAM Conditions
- Generated Vertex SDK resource types for `Session` and `Memory`
- ADK 2.6.x session and memory service implementations
