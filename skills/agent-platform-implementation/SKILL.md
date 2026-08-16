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
  "my multi-turn eval case tests something that never happened", "the A2UI card
  renders but the button does nothing", "which A2UI components does Gemini
  Enterprise actually render", "one component of my surface is missing", "the
  model keeps re-reading its own surface", "how do I build an A2UI tree", or "a
  parallel stage produced no output and no error". Covers what actually crosses
  an agent-to-agent boundary, guards that a model cannot game, building an A2UI
  surface that survives the round trip, and the verification traps that report a
  score unrelated to your agent. Use agent-platform-architecture to choose a host
  or transport — including whether A2UI can render there at all —
  agent-platform-a2a for calls failing with a status code, and
  google-agents-cli-eval for eval methodology, dataset schema and metric choice.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.2.0
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
| The card renders but one component is missing | GE renders a **subset** of the catalog. Being in the standard catalog is not a rendering guarantee (I12) |
| Should the model emit the A2UI JSON? | **No.** Have it call a tool with domain data and build the tree in code (below) |
| Why is my prompt growing every turn? | The emitted surface is replaying as history — ~2,200 prompt tokens per call (I13) |
| The button fires but the handler has nothing to work with | The surface is gone from history; state must travel in `action.context` (I14) |
| A parallel stage returned nothing, with no error | A service the Runner was never given raised, ending the sequence before later stages (I15) |

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
| I12 | Gemini Enterprise renders a **subset** of the A2UI standard catalog, and a component it does not implement is dropped without complaint | ⇒ catalog membership, schema validity and a clean payload check all coexist with a component that never appears ⇒ a component is unproven until seen rendering **in GE**, so design flows on what you have observed, not on what the spec lists |
| I13 | The emitted surface travels as text inside the message, so it lands in the session event and is replayed into the model's context on every later turn | ⇒ measured ~3,400 characters, roughly **2,200 prompt tokens per call**, for the remainder of the conversation ⇒ and it invites the model to imitate markup you instructed it not to emit ⇒ strip it from the request, and strip it **app-wide**: every agent in the app is handed the same history, so a memory or gatherer agent carries the surface too |
| I14 | Stripping the surface (I13) removes the only record of what was rendered, and the client returns an interaction — not the tree | ⇒ whatever a later turn needs (the user's selection, the menu, the title) must travel in that interaction's own `action.context` ⇒ it cannot be recovered from history, because you removed it |
| I15 | A service the `Runner` was never given raises inside whichever agent needs it, and inside a parallel stage that exception ends the **whole sequence** | ⇒ later stages never run ⇒ the symptom is empty output, not a missing-service error, so it reads as a model failure rather than a wiring one |

Verified by observation: I1, I2, I3, I4, I6, I7, I8, I9, I10, I11, I12, I13,
I14, I15. Verified by reading the client: I5.

**I3 and I4 are the same rule seen twice.** I3 is the mechanism, I4 is how it
reaches you: not as a missing-data error but as a caller that answers
confidently and wrongly. The instinct is to constrain the caller. The fix is
almost always to make the sub-agent say more.

**I1 and I10 are the same rule seen twice.** Both are "the path you exercised is
not the path your users are served" — once while fixing, once while measuring.

**I13 and I14 are one decision and its bill.** Removing the surface from context
is worth doing, and it is why the round trip has to carry its own state. Do the
first without the second and the button renders, fires, and arrives at an agent
that has no idea what was on the card.

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

## Building an A2UI surface

Whether A2UI can render at all is a hosting question — `agent-platform-architecture`
F4 and F12 settle it, and no code here compensates for the wrong answer. This
section assumes negotiation works and covers what to build once it does.

### Do not ask the model for the tree

A component tree is a schema-constrained document with internal references. A
model asked to emit one produces something plausible most of the time, which is
the worst available failure rate: it renders in development and drops a section
in production, and the fault is invisible because a missing component looks like
a component that was never sent (I12).

Give the model a tool whose parameters are **domain data** and build the tree in
code:

```python
def propose_lunch(
    title: str, rationale: str, attendees: list[str],
    slot_labels: list[str], slot_values: list[str], slot_absentees: list[str],
    ...
) -> str:
```

The model supplies names, labels and prose — the things it is good at — and
never sees a component. The tree is then valid by construction, the surface is
diffable in unit tests without a model in the loop, and the guard from the
previous section has somewhere to live: the tool call is the only place where
domain data and another agent's output are both in scope.

### What Gemini Enterprise actually renders

Being in the v0.8 standard catalog does not mean GE draws it (I12). Components
observed rendering in a deployed GE app:

| Component | Observed as |
| :--- | :--- |
| `Card` | the surface container |
| `Column` | vertical layout |
| `Text` | headings and body copy |
| `MultipleChoice` | selectable chips, single- or multi-select |
| `List` | repeated rows from a data path, via a template |
| `Button` | primary action, carrying `action.context` |

Everything else in the catalog is **unverified**, not unsupported — the
distinction matters, because the cost of finding out is one deployment and the
cost of assuming is a flow built on a component that never appears. A payload
carrying a doubtful component alongside known-good ones settles it in one round
trip: if the rest of the card renders and that component does not, the answer is
in front of you.

Prefer a vetted component that constrains the input over a richer one that does
not. A chip set commits on click and cannot express an invalid choice; a free
date picker admits 3am on a Sunday and a slot that is already booked, so it
needs a validation round trip that the chips never required. That is a change to
the flow, not a swap of one component for another.

### Round-tripping a choice

What comes back from an interaction is the action name and its context — not the
tree, which you have already stripped from history (I13, I14). So the button has
to carry everything the next turn needs:

```python
"Button": {
    "child": "book-button-text",
    "primary": True,
    "action": {
        "name": "book_lunch",
        # Reads live state, so the action follows the user's choice rather than
        # whatever was selected when the surface was built.
        "context": [
            {"key": "selectedSlots", "value": {"path": "/selectedSlots"}},
            {"key": "title",         "value": {"path": "/title"}},
            {"key": "menuItems",     "value": {"path": "/menuItems"}},
        ],
    },
}
```

Bind each entry to a **path into the surface's data model**, not to a literal
captured at build time. The path is resolved when the button is pressed, so a
chip selected after the card rendered is reflected; a literal freezes the
proposal as it was first drawn.

Treat this list as the handler's input contract. Anything the confirmation
needs to state back — the slot, the menu, the attendees — travels here or is
unavailable, and adding a field to the card later without adding it here
produces a confirmation that is confidently missing a detail.

### Keeping the surface out of the model's context

The surface has to reach the client inside the message text, so it lands in the
session event and is replayed on every subsequent turn (I13). Trim it from the
request in a plugin, leaving stored events untouched:

```python
class A2uiHistoryPlugin(BasePlugin):
    async def before_model_callback(self, *, callback_context, llm_request) -> None:
        llm_request.contents = [_strip_a2ui(c) for c in llm_request.contents]
```

Two details that are easy to get wrong:

- **Register it on the `App`, not the emitting agent.** Every agent in the app
  receives the same conversation history, so scoping it to the synthesizer
  leaves the gatherers and memory agents reading the markup instead.
- **Trim the request, never the stored event.** The client may re-fetch the
  conversation, and the surface is what it renders.

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

### Wire the runner the way the server does

A test or script that builds its own `Runner` is a second wiring of the app, and
it is wrong by omission by default. A `Runner` given a session service and
nothing else satisfies most agents; the one that calls `load_memory` raises
`Memory service is not available`, and in a parallel stage that exception takes
the whole sequence with it (I15). Later stages never run, so the reported
symptom is *no output* — which points at the model.

Build it from the same `App` and the same service builders the server uses:

```python
Runner(
    app=adk_app,                                  # carries the app's plugins
    session_service=services.get_session_service(),
    artifact_service=services.get_artifact_service(),
    memory_service=services.get_memory_service(),
)
```

Passing the `App` rather than a bare `root_agent` matters for the same reason as
I13: the plugins are attached to the app, so a runner built from the agent alone
silently drops them.

The general form is worth stating, because it is not specific to memory: **an
agent that needs a service it was not given fails at the point of use, not at
construction.** The stack trace names the tool; nothing names the runner.

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
- **Having the model emit the component tree.** Plausible most of the time is the
  worst failure rate available; build it in code from domain data (I12).
- **Assuming a catalog component renders.** The spec lists it, the payload
  validates, and GE draws nothing. Only observation in GE settles it (I12).
- **Stripping the surface from history without moving state into
  `action.context`.** The button then fires into a turn that has no idea what was
  on the card (I13, I14).
- **Reading "no text content" as a model failure.** In a multi-stage agent it is
  usually a stage that raised — a service the runner was never given (I15).

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
| Card renders, one section missing | Component not implemented by GE — I12 |
| Surface renders in dev, malformed in production | Model emitting the tree — I12, above |
| Prompt grows every turn, latency climbs | Surface replaying as history — I13 |
| Button fires, handler has no data | State not in `action.context` — I14 |
| Gatherer agents echo A2UI markup | History plugin scoped to one agent — I13 |
| Agent produced no text, no error raised | Stage raised on a missing service — I15 |
| Works under the server, fails in a test script | Runner built without the app's services or plugins — I15 |

## Sources

- ADK 2.6.1: `google.adk.agents.remote_a2a_agent.RemoteA2aAgent` (MRO and model
  fields), `google.adk.a2a.executor.a2a_agent_executor`,
  `google.adk.a2a.converters.event_converter` and `.from_adk_event`.
- `agents-cli` 1.2–1.3: `eval generate`, `eval grade`.
- A2UI v0.8 standard catalog; `a2ui-agent-sdk` 0.5.x. The rendering table (I12)
  is what was observed in a deployed Gemini Enterprise app, not what the catalog
  defines — the two are not the same list.
- Remaining claims are from observed behaviour of deployed agents on Agent
  Runtime and Cloud Run; where a claim is a measurement it carries the numbers.
