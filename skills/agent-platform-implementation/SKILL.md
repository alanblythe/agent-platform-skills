---
name: agent-platform-implementation
description: >
  This skill should be used when a multi-agent build on Gemini Enterprise Agent
  Platform runs without erroring but is quietly wrong: "the agent invented data
  the tool returned correctly", "it works with curl but not in Gemini
  Enterprise", "my sub-agent returned 200 and the caller still made something
  up", "a tool argument is fabricated", "the orchestrator dropped half the
  list", "/run_sse returns 404", "how do I read a sub-agent's output", "how do I
  stop the model inventing tool arguments", "my eval score did not change",
  "eval grade scored the wrong run", "the metric scores 0 on a working agent",
  or "my multi-turn eval case tests something that never happened". Covers what
  actually crosses an agent-to-agent boundary, guards that a model cannot game,
  and the verification traps that report a score unrelated to your agent. Use
  agent-platform-architecture to choose a host or transport,
  agent-platform-a2a for calls failing with a status code, and
  google-agents-cli-eval for eval methodology, dataset schema and metric choice.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.1.0
---

# Implementing and verifying a multi-agent deployment

Covers the layer between a working architecture and a correct one: what one
agent can actually see of another, how to constrain a model that is filling a
gap, and why the signals you check after a change routinely measure something
other than the change.

For choosing a host, transport or topology, see `agent-platform-architecture`.
For calls failing with a status code, see `agent-platform-a2a`. For eval
methodology — the flywheel, dataset schema, choosing metrics — see the
`google-agents-cli` skills; this covers only the traps where the harness
reports a number that is not about your agent.

Client-side claims describe ADK 2.6.x and `agents-cli` 1.2–1.3.

## Quick answers

| Question | Answer |
| :--- | :--- |
| A sub-agent returned the right data and the caller still invented some — why? | Only its **rendered text** crossed. Tool results stay in the sub-agent's own session (I3) |
| How does a caller read a sub-agent's output? | From session events filtered by `author` and `invocation_id`. `RemoteA2aAgent` has no `output_key` (I5) |
| My fix works with curl but not in Gemini Enterprise | You wrapped one of ADK's **two** executors. GE negotiates the other one (I1) |
| `/run_sse` 404s but the agent card resolves | ADK REST uses the **agent directory**; the card path uses the ADK `App` name (I2) |
| Can I validate a tool argument against the model's own output? | **No.** It will satisfy you by degrading the other side (I6) |
| Will more thinking budget stop the invention? | Only if the information was present. It cannot supply what never arrived (I7) |
| My score is identical after a real change | `eval grade` scored the **previous** trace; nothing warns you (I8) |
| A metric scores 0 on an agent that demonstrably works | You are grading the wrong transport (I10) |
| Can I test "cancel what turn 1 booked" in a dataset? | **Not through `agent_data.turns`** — earlier turns are narrated, not run (I9) |

## Forces

Rules, not preferences. `⇒` means the platform or framework gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| I1 | ADK ships **two** A2A executors and selects between them **per request**: a client that negotiates the executor extension gets the newer one, reading `adk_event_converter`; every other client gets the legacy one, reading `event_converter` | ⇒ any `A2aAgentExecutorConfig` customization must wrap **both** ⇒ wrapping one makes `curl` look fixed while every real client turn is unconverted, because the client that negotiates is the one you are not testing with |
| I2 | One deployment answers to **two different names**: the A2A card path carries the ADK `App` name, the ADK REST path (`/run_sse`, `/list-apps`) carries the **agent directory** | ⇒ a script that reuses one name on the other path 404s every request, which reads as a broken deployment rather than a wrong string |
| I3 | Across an agent-to-agent hop the caller receives the sub-agent's **rendered text and nothing else** — tool calls and tool results belong to the sub-agent's own session | ⇒ a fact the sub-agent looked up but did not say **does not exist** to the caller ⇒ the sub-agent's prose *is* its API |
| I4 | A summary written for a human names what is **exceptional**; a caller reconstructing state needs what is **complete** | ⇒ "7 of 8 free (Hal unavailable)" tells a person everything and a caller almost nothing ⇒ the caller fills the gap, and a fluent model fills it with invention |
| I5 | `RemoteA2aAgent` extends `BaseAgent`, not `LlmAgent`, and `output_key` is an `LlmAgent` field | ⇒ a remote sub-agent's output cannot be routed to state declaratively ⇒ read it from `ToolContext.session.events`, filtering on `author` **and** `invocation_id` so an earlier turn's data is not mistaken for this one's |
| I6 | A validation derived from the model's own output constrains nothing: both sides of the comparison are things the model writes | ⇒ it satisfies the check by degrading the *other* side ⇒ ground a guard in another agent's output, a tool result, or a store — never in a second field of the same tool call |
| I7 | Reasoning budget governs how well a model uses its context, not what is in it | ⇒ raising the thinking level cannot repair a missing input; it buys latency and hides the cause ⇒ measured 55–122 s per turn at `MEDIUM` against ~44 s at `LOW` for the same task |
| I8 | `agents-cli eval grade` reads the **newest** trace file and does not check its age | ⇒ a failed or skipped `generate` silently grades the previous run and prints a plausible, unchanged score ⇒ "my score did not move" and "my inference did not run" are indistinguishable |
| I9 | `agents-cli eval generate` **narrates** a case's earlier turns into context rather than executing them | ⇒ the agent reads that it did something no tool call performed ⇒ nothing stateful — cancel what turn 1 created, amend a prior write — is testable through `agent_data.turns` |
| I10 | A2A and ADK REST return **different responses to the same prompt** whenever the executor filters or converts events, and `eval generate --url` drives the REST path | ⇒ a metric describing A2A-only behaviour scores 0 against a working agent ⇒ and an A2A artifact is not an ADK trace, so `eval grade` rejects the file rather than reading its parts |
| I11 | `uv run agents-cli` resolves the version pinned **inside the agent**, which may differ from the globally installed tool | ⇒ flags present in one are absent in the other ⇒ but a metric that imports the agent package needs the in-agent resolution, so the choice is not free |

Verified by observation: I1, I2, I3, I4, I6, I7, I8, I9, I10, I11. Verified by
reading the client: I5.

**I3 and I4 are the same rule seen twice.** I3 is the mechanism, I4 is how it
reaches you: not as a missing-data error but as a caller that answers
confidently and wrongly. The instinct is to constrain the caller. The fix is
almost always to make the sub-agent say more.

**I1 and I10 are the same rule seen twice.** Both are "the path you exercised is
not the path your users are served" — once while fixing, once while measuring.

## What crosses an agent boundary

An orchestrator composing remote sub-agents sees a transcript, not a workspace.
The sub-agent's tools ran in its own process against its own session; what
arrives is whatever it chose to write.

This makes a sub-agent's output format an interface with two incompatible
consumers, and the human one wins by default because it is the one you read
while developing. A gatherer asked for availability will happily answer:

```
1. Friday 12:00-13:00 — 8 of 8 free
2. Tuesday 13:00-14:00 — 7 of 8 free (Hal unavailable)
```

Correct, useful, and it names one member of an eight-person team. A caller that
must render the full set has been handed a count and one exception. It will not
error. It will produce eight plausible names.

The fix is upstream and cheap — require the sub-agent to state the set, in a
form the caller can lift verbatim:

```
Team (8): Ada, Bo, Cyrus, Dara, Elif, Fen, Gita, Hal
```

Then the caller has something to copy, and a guard downstream has something to
check against. Both properties come from the same line.

**Diagnostic.** When a caller invents, probe the sub-agent **directly** with the
same prompt the caller sends, and read its raw text. A sub-agent that answers a
pointed question correctly ("list every member") can still omit that data from
its normal reply — so testing the tool proves nothing about the hop. Compare
what the sub-agent *said* with what the caller *needed*, not with what the
sub-agent's tools returned.

### Reading a sub-agent's output in code

`output_key` is unavailable (I5), so a guard reads the session directly:

```python
def _sub_agent_text(tool_context: ToolContext, author: str) -> str:
    """Concatenates what one sub-agent said during this invocation."""
    session = getattr(tool_context, "session", None)
    if session is None:
        return ""
    chunks = []
    for event in session.events or []:
        # invocation_id matters: without it an earlier turn's data is
        # indistinguishable from this turn's.
        if event.author != author or event.invocation_id != tool_context.invocation_id:
            continue
        for part in getattr(event.content, "parts", None) or []:
            if part.text:
                chunks.append(part.text)
    return "\n".join(chunks)
```

The author string is the sub-agent's configured `name`. Renaming the agent
silently disables any guard built on it, so it is worth a named constant next
to the check rather than a literal inside it.

## Guards a model cannot satisfy the wrong way

Structural validation on a tool call — list lengths pairing up, an enum member,
a required field — is cheap and worth having. It does not touch the failure that
matters here, which is a well-formed call carrying invented content.

Where a guard is grounded decides whether it works at all.

| Guard grounded in | Model's escape | Useful |
| :--- | :--- | :--- |
| Another field of the same tool call | Change that field instead | ❌ |
| The model's own prior message | Restate it to match | ❌ |
| Another agent's output this invocation | None — it cannot edit that event | ✅ |
| A tool result or store read | None | ✅ |

A worked failure: a tool received a list of attendees and a set of slot labels
reading `(8 of 8 free)`. Checking the list length against the number in the
label looks like a grounded consistency check. It is not — the model writes
both. Given a rejection it rewrote the *label*, arriving at `(4 of 4 free)`,
then `(0 of 0 free)` after twelve rejections, satisfying the guard each time by
making the output worse.

Rewritten against the sub-agent's roster line the same guard cannot be
negotiated with, because the model cannot edit an event another agent already
emitted:

```python
roster = _roster_from_text(_sub_agent_text(tool_context, SCHEDULING_AGENT))
if roster is not None:
    missing = [n for n in roster if n.casefold() not in {a.casefold() for a in attendees}]
    extra   = [a for a in attendees if a.casefold() not in {n.casefold() for n in roster}]
    if missing or extra:
        raise ValueError(...)   # returned to the model as correctable text
```

Three properties worth copying:

- **Check both directions.** Inventions and omissions are different failures and
  a caller under pressure produces both — including placeholders (`Team Member
  5`) that are neither real nor obviously wrong.
- **Match whole values, not tokens.** An invented "Dara Whitfield" must not pass
  on the strength of a real "Dara".
- **Skip the check when the grounding is absent, rather than failing.** If the
  sub-agent produced no roster line there is nothing to judge against, and
  refusing every call punishes the agent for an upstream gap. Degrade to the
  weaker check — inventions only — so the guard stays useful against a
  sub-agent that has not been updated yet.

Return the failure **to the model as text it can act on**, naming what was wrong
and what to use instead; a raised exception ends the turn, while a correctable
message is usually resolved on the next attempt.

## Verifying a change

The recurring failure is not a wrong number, it is a number that was never about
the thing you changed.

### Grade the run you just made

`eval grade` takes the newest trace and does not care how old it is (I8). A
`generate` that failed, 404'd on the wrong app name (I2), or was never run,
leaves the previous trace in place to be graded again — and the score comes back
plausible and unchanged, which reads exactly like "my change had no effect".

Refuse a trace older than the code that produced it:

```bash
TRACE=$(ls -t artifacts/traces/*.json | head -1)
if find app -name '*.py' -newer "$TRACE" -print -quit | grep -q .; then
  echo "Error: source is newer than $TRACE — inference did not run against this code."
  exit 1
fi
```

### Grade the transport your users are served

Where the executor filters or converts events, A2A and ADK REST answer the same
prompt differently, and `eval generate --url` drives REST (I10). A metric
written for the delivered behaviour then scores 0 against an agent that works —
and the natural reading of a 0 is that the agent is broken.

Two consequences:

- Keep transport-specific metrics in a **separate config**, and say in a comment
  which path each grades. A metric quietly scoring the wrong path is worse than
  a missing one.
- An A2A artifact is not an ADK trace. `eval grade` rejects the file rather than
  reading its parts, so driving A2A means a small harness that sends over A2A
  and scores in the same pass.

### Test stateful behaviour outside the dataset

Earlier turns in a case are narrated, not executed (I9), so an agent asked to
cancel what turn 1 created is reasoning about a record no tool ever wrote. Some
agents comply anyway, which produces a passing case for behaviour that has never
run.

Anything stateful needs turns that really happen — one session id, reused across
sends, against the running agent:

```python
session = json.loads(post(f"/apps/{APP}/users/{USER}/sessions"))["id"]
for text in TURNS:
    post(f"/apps/{APP}/users/{USER}/sessions/{session}", {"new_message": ...})
```

Point it at a **local** store. Run against a deployed agent, a cancellation case
deletes real records — unset whatever variable selects the managed store, and
confirm it is unset rather than assuming.

Expect real conversations to need more turns than the dataset implies: a
confirmation guard that states a count before accepting agreement takes three
turns, not two, because confirming a number the agent has not shown you yet does
not satisfy it. That is the guard working.

## Dead ends

- **Constraining the caller harder when it invents.** More forceful instructions
  on an agent that was never given the data change the *shape* of the
  fabrication, not its presence — placeholders instead of plausible names. Check
  what crossed the boundary first (I3).
- **Raising the thinking level to fix a grounding failure.** It cannot supply an
  absent input, and the latency is real (I7). Rule out missing context before
  spending it.
- **Wrapping one executor and confirming with `curl`.** The client you are
  debugging for negotiates the other one (I1).
- **Deriving a guard from the model's own output.** Satisfied by degrading the
  other side (I6).
- **Trusting an unchanged score.** Identical before and after is the signature of
  a trace that was never regenerated (I8).
- **Testing the sub-agent's tool instead of the hop.** `get_x` returning the
  right data says nothing about whether the sub-agent *said* it (I3).

## Troubleshooting index

| Symptom | Likely cause |
| :--- | :--- |
| Caller invents data a sub-agent holds | Sub-agent's text omits it — I3, I4 |
| Caller emits placeholders (`Team Member 5`) | Same, with a guard already blocking invention — I4 |
| Works via `curl`, wrong in Gemini Enterprise | Only one executor wrapped — I1 |
| `/run_sse` 404s, card resolves fine | Agent directory vs ADK `App` name — I2 |
| Guard passes but output got worse | Guard grounded in the model's own output — I6 |
| Sub-agent output unreadable from a tool | `RemoteA2aAgent` has no `output_key` — I5 |
| Score identical after a real change | Stale trace regraded — I8 |
| Metric scores 0 on a working agent | Grading the wrong transport — I10 |
| Multi-turn case passes for behaviour never run | Earlier turns narrated — I9 |
| CLI flag missing that the docs list | In-agent pin vs global tool — I11 |

## Sources

- ADK 2.6.1: `google.adk.agents.remote_a2a_agent.RemoteA2aAgent` (MRO and model
  fields), `google.adk.a2a.executor.a2a_agent_executor`,
  `google.adk.a2a.converters.event_converter` and `.from_adk_event`.
- `agents-cli` 1.2–1.3: `eval generate`, `eval grade`.
- Remaining claims are from observed behaviour of deployed agents on Agent
  Runtime and Cloud Run; where a claim is a measurement it carries the numbers.
