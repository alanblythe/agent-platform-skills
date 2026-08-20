---
name: agent-platform-architecture
description: >
  This skill should be used when designing how agents fit together on Gemini
  Enterprise Agent Platform: "should this be one agent or several", "how do I
  expose an agent in Gemini Enterprise", "do I need Agent Gateway", "should I
  enable Agent Identity", "can these agents call each other", "how do agents
  authenticate to each other", "how does an agent call a Cloud Run service",
  "why does A2UI not render", or when choosing
  between Agent Runtime and Cloud Run. Gives the platform constraints, the
  transport compatibility matrix for agent-to-agent calls, and the architectures
  they permit. Use agent-platform-a2a for debugging a built deployment.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.8.1
---

# Agent architecture on Agent Platform

## Service names

Named services of Gemini Enterprise Agent Platform, used throughout:
**Agent Runtime** (managed host), **Agent Identity**, **Agent Gateway**, and
**Agent Registry**. **Sessions** and **Memory Bank** hold state — see
`agent-platform-state`. Cloud Run is a separate Google Cloud product, used here
as the alternative host.

"Engine" is the API-level noun for a deployed Agent Runtime agent: the REST
resource is `reasoningEngines`, and log fields, IAM permissions, and resource
paths all use that spelling. The product is Agent Runtime; the resource is an
engine.

## Quick answers

| Question | Answer |
| :--- | :--- |
| Do agents calling each other require Agent Gateway? | **No.** Only a transport the identity mode supports. See [Transport matrix](#transport-matrix) |
| Does Agent Identity break agent-to-agent A2A? | It breaks **plain-bearer transports** (`RemoteA2aAgent`). Use the genai client transport instead |
| When is Agent Gateway actually required? | **Only for gateway policy enforcement.** Cloud Run targets no longer force it — see Architecture F |
| How does an Agent Identity agent reach Cloud Run? | Mint an ID token **as a delegate service account** via `generateIdToken` (F). Not on its own credential (F3) |
| Can `adkAgentDefinition` render A2UI? | **Never.** A2UI is negotiated from an agent card, which only A2A registration has |
| Where can A2UI actually run? | **Cloud Run only.** Agent Runtime's passthrough strips the `X-A2A-Extensions` echo A2UI negotiation depends on (F12) |
| Can a Cloud Run agent have Agent Identity? | **Yes**, via hidden alpha flags (F13). Not in the docs, the published API schema, or `--help` |
| Which host gives both A2UI and Agent Identity? | **Cloud Run only** — Agent Runtime cannot serve A2UI at all |
| Does running N separate engines cost extra? | **No.** Independent deployment is free; only the calling pattern has consequences |
| Cheapest way to get per-agent identity? | Do not make agent-to-agent calls (Architecture A), or keep sub-agents in-process (C) |

## Two independent axes

Design decisions on this platform reduce to two choices that compose:

1. **Identity mode** — service account (default) or Agent Identity
2. **Hop transport** — how one agent reaches another, if at all

Everything else follows.

**Bootstrap does not discriminate between the identity modes.** A create with
no packaged code (F15) mints the per-engine Agent Identity *and* the project's
`gcp-sa-aiplatform-re` service agent, in ~20s, so either mode can be fully
provisioned before any code exists. Choose on what the modes actually differ
on — transport compatibility (F1, F2), whether a service account is available
to delegate through (F3, F11), and that Agent Identity is Preview. See
`agent-platform-a2a` §9 for the call and its measured effects.

What each mode needs granted differs, though. A custom service account needs
`-re` to hold `roles/iam.serviceAccountTokenCreator` on it, so the platform can
impersonate it. Agent Identity has no service account, so nothing needs
impersonating and `-re` stays off the agent's permission path.

## Forces

Rules, not preferences. `⇒` means the platform gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| F1 | Agent Identity replaces the service account with a SPIFFE principal bound by mTLS/DPoP | ⇒ no bearer tokens; a plain `Authorization: Bearer` call fails **401** and no IAM grant fixes it |
| F2 | Google client libraries implement that binding; hand-rolled HTTP does not | ⇒ **the transport determines whether a call works under Agent Identity**, not the protocol |
| F3 | Agent Identity has no service account | ⇒ the runtime cannot mint an audience-bound ID token **as itself** (the MDS `identity?audience=` endpoint does not serve one) ⇒ Cloud Run is unreachable on the credential alone — but reachable by minting *as a delegate service account* (Architecture F) |
| F4 | Gemini Enterprise learns A2UI support from the agent card | ⇒ A2UI requires `a2aAgentDefinition`; `adkAgentDefinition` carries no card and can never render A2UI |
| F5 | A2A registration means GE calls the card's `url` directly | ⇒ Agent Runtime needs an Authorization with `cloud-platform` scope; Cloud Run needs `roles/run.servicesInvoker` on the Discovery Engine service agent |
| F6 | Agent Gateway terminates the mTLS handshake for the caller | ⇒ it makes any transport work under Agent Identity, including Cloud Run targets — but it is not the only way to reach them (F3, Architecture F) |
| F7 | The gateway endpoint is a PSC service attachment in a Google tenant project | ⇒ VPC, subnet, network attachment, DNS peering, and every agent redeployed onto it |
| F8 | Attaching a gateway moves the engine's Google API traffic to `*.mtls.googleapis.com` | ⇒ a gateway the agent cannot reach breaks it **worse than having none**: the engine's own session calls fail |
| F9 | Independent deployment costs nothing; only the calling pattern matters | ⇒ "several engines" and "agents calling each other" are separate decisions |
| F10 | The binding applies to **every** Google API call the agent makes, not just agent-to-agent ones | ⇒ even minting a delegated token must go through a generated client; a hand-built POST to `iamcredentials.googleapis.com` is rejected **401 "invalid authentication credentials"** |
| F11 | An agent identity is a first-class IAM principal | ⇒ it can hold roles on other resources, **including on a service account** — which is what makes Architecture F possible |
| F12 | Agent Runtime's `/api/` passthrough replaces response headers wholesale — **nothing an engine's own app sets reaches the caller** | ⇒ any protocol needing a *response* header fails there. A2UI is negotiated by echoing `X-A2A-Extensions`, so **A2UI cannot work on Agent Runtime**; the A2UI-capable host is Cloud Run |
| F13 | Cloud Run supports Agent Identity via annotations, exposed only as **hidden alpha** flags | ⇒ a Cloud Run service can hold a SPIFFE principal **and** serve A2UI, so it is the only host that satisfies both |
| F14 | A Cloud Run agent identity receives **no default roles**, while an Agent Runtime one is created with `roles/aiplatform.agentDefaultAccess` bound to a project-wide `principalSet` | ⇒ on Cloud Run nothing is granted implicitly and every permission the agent had as a service account must be re-granted; on Agent Runtime a bare identity can already reach Agent Platform, which hides the difference until you move host |
| F15 | An engine is a resource, not a running thing: it can be created with a display name and **no packaged code**, and Sessions and Memory Bank attach to it either way. Such an engine has **no `deploymentSpec`** — no image, no instances, nothing warm | ⇒ a host that injects no engine id is not cut off from managed state — provision an engine that runs nothing and point the agent at it, at no standing cost. Anything with a real deployment spec defaults to `min_instances: 1` |

Verified by observation: F1, F2, F3, F4, F6, F8, F9, F10, F11, F12, F13, F14, F15. From documentation: F5, F7.

**F1 and F10 are the same rule seen twice.** F1 is usually met at the agent-to-agent
hop and read as "A2A is broken under Agent Identity". It is not about A2A: any
hand-rolled HTTP carrying the credential as a bearer fails the same way, whatever
the destination. Reach for a client library before reaching for IAM.

## Transport matrix

The central table. Rows are ways one agent can reach another; columns are the
caller's identity mode.

| Transport | Service account | Agent Identity | Notes |
| :--- | :---: | :---: | :--- |
| **In-process sub-agent** (no network) | ✅ | ✅ | No credential involved at all |
| **genai client** `request()` to `…/a2a/message:send` | ✅ | ✅ | Binding-aware; the transport `stream_query` uses |
| **ADK `RemoteA2aAgent`** (httpx + bearer) | ✅ | ❌ 401 | Plain bearer; cannot satisfy DPoP |
| **`RemoteA2aAgent` over a genai-client httpx transport** | ✅ | ✅ | Same binding-aware path, with ADK's card resolution and part conversion kept |
| **Hand-rolled httpx + ADC bearer** | ✅ | ❌ 401 | Same failure as `RemoteA2aAgent` |
| **Delegate-SA ID token** (`generateIdToken` via client library) | ✅ | ✅ | The only non-gateway route to Cloud Run (F3, Architecture F) |
| **Agent Gateway** | ✅ | ✅ | Terminates mTLS for the caller; adds policy enforcement |

And by **target**, under Agent Identity:

| Target hosting | Reachable under Agent Identity | How |
| :--- | :--- | :--- |
| Agent Runtime engine | ✅ | genai client transport (D), or Agent Gateway (E) |
| Cloud Run service | ✅ — but not on the agent credential itself | Delegate service account (F), Agent Gateway (E), or move the target to Agent Runtime |

A Cloud Run service can itself **run under** Agent Identity (F13), which is separate from being reachable as a target.

### The genai transport, concretely

The credential comes from `vertexai.Client(...)._api_client`, which carries the
binding. **The path and body depend on how the target serves A2A**, and the two
shapes are not interchangeable — using the wrong one returns 404.

**Target serves A2A natively** (Agent Runtime's own A2A, a2a-sdk 1.x):

```python
api = vertexai.Client(project=P, location=L, http_options={"api_version": V})._api_client
resp = api.request(
    "post",
    f"{engine_resource}/a2a/message:send",
    proto_json_body,                          # a2a-sdk 1.x SendMessageRequest as proto-JSON
    {"headers": {"A2A-Version": "1.0"}},      # mandatory
)
# reply is {"task": {...}} or {"message": {...}}
```

**Target self-serves ADK routes** (a FastAPI app on Agent Runtime, a2a-sdk 0.3.x)
— the engine's own routes are exposed under an `/api/` prefix, and the version
prefix is two segments, so it goes in `api_version` rather than the path:

```python
api = vertexai.Client(project=P, location=L,
                      http_options={"api_version": "reasoningEngines/v1"})._api_client
resp = api.request(
    "post",
    f"{engine_resource}/api/a2a/{app_name}",  # app_name is the ADK App name
    {"jsonrpc": "2.0", "id": ..., "method": "message/send",
     "params": {"message": {...}}},           # JSON-RPC, no version header
    None,
)
```

Requirements that fail obscurely if missed:

- **`A2A-Version: 1.0` header** on the *native* path only. A missing version header is read as `0.3` and returns `VERSION_NOT_SUPPORTED`. Sending it to a 0.3.x self-served route is wrong in the other direction.
- **`…/a2a/message:send` 404s against a self-served engine**, and `…/api/a2a/{app}` 404s against a native one. Confirm which the target serves before assuming the transport is broken.
- **Do not use aiplatform's own A2A client wrapper.** It is hard-coded to the a2a-sdk 0.3.x surface and cannot call a 1.x engine.

#### Implementing it without leaving ADK

The genai client can be wrapped as an `httpx.AsyncBaseTransport` and handed to an
ordinary `RemoteA2aAgent`, which is far less work than writing an A2A client:

```python
class GenaiApiTransport(httpx.AsyncBaseTransport):
    async def handle_async_request(self, request):
        path = str(request.url)[len(f"{base_url}/{api_version}/"):]
        # request() is blocking — it owns its own retry loop and credential refresh
        return await anyio.to_thread.run_sync(self._send, request, path)

client = httpx.AsyncClient(transport=GenaiApiTransport(...))
agent = RemoteA2aAgent(name=..., agent_card=card_url, httpx_client=client)
```

Card resolution, the JSON-RPC envelope, and part conversion stay ADK's code; only
the bytes-on-the-wire path changes. On a2a-sdk 0.3.x ADK already defaults to
`streaming=False`, so the client uses `message/send` and no SSE is involved.

## Architectures

| | Identity | Hop | Transport | Extra infrastructure |
| :--- | :--- | :--- | :--- | :--- |
| **A. Fan-out** | Agent Identity | none | — | none |
| **B. Shared identity** | service account | yes | `RemoteA2aAgent` | none |
| **C. Composed** | Agent Identity | in-process | — | none |
| **D. Client transport** | Agent Identity | yes | genai client | none |
| **E. Gateway** | Agent Identity | yes | Agent Gateway | VPC + PSC + DNS + IAM |
| **F. Delegated service account** | Agent Identity | yes | D, plus a delegate SA for Cloud Run targets | one service account + two IAM bindings |

### A. Fan-out — N independent agents, no hop
Separately deployed engines, each serving users directly.
- **Gives** per-agent identity, independent deploy/scale/version, per-agent audit attribution
- **Requires** that the fan-out is user-to-agent, not agent-to-agent
- **Choose when** agents do not need to call each other. Cheapest per-agent identity

### B. Shared identity — hop on the default service account
Orchestrator calls remote sub-agents with `RemoteA2aAgent`.
- **Gives** independent sub-agents, working A2A, no extra infrastructure, works with Cloud Run targets
- **Costs** one identity for all agents: no per-agent authorization or attribution
- **Choose when** per-agent identity is not a requirement. This is the default

### C. Composed — sub-agents in-process
Sub-agents are ordinary ADK objects rather than `RemoteA2aAgent` proxies.
- **Gives** per-agent identity for the deployed unit, no credentials between components, lower latency, failures raise instead of degrading
- **Costs** sub-agents no longer independently deployable or callable by others
- **Choose when** you own all the agents and they are not separately consumed. ADK guidance points here: use A2A when the target is "a separate, standalone service" or "maintained by a different team or organization"

### D. Client transport — hop under Agent Identity, no infrastructure
Agent Identity with the hop made through the genai client transport instead of
`RemoteA2aAgent`.
- **Gives** per-agent identity **and** independent sub-agents, with no additional infrastructure
- **Costs** one transport module. `RemoteA2aAgent` can be kept by swapping its httpx transport (above); writing a whole A2A client is not required
- **Requires** the target to be an Agent Runtime engine (F3). For Cloud Run targets, extend to F
- **Choose when** you need both per-agent identity and independently deployed sub-agents. **This is the default answer to that requirement** — prefer it over E

### E. Gateway — hop under Agent Identity, with policy enforcement
Agent Identity with Agent Gateway carrying the hop.
- **Gives** everything D gives, plus Cloud Run targets and centralised policy enforcement
- **Costs** VPC, subnet, PSC network attachment, private DNS zone, additional IAM on the AI Platform service agent, and a redeploy of every agent onto the network attachment
- **Requires** engines created with `identity_type=AGENT_IDENTITY`; attaching a gateway does not change it
- **Choose when** gateway policy enforcement is a requirement. **Not needed merely to make A2A work, and no longer needed merely to reach Cloud Run** — F does that for one service account and two bindings

### F. Delegated service account — Cloud Run targets under Agent Identity
Architecture D for Agent Runtime peers, plus a purpose-made service account the
agent impersonates for each Cloud Run target. The agent uses its own bound
credential to ask IAM to mint an ID token **as** the delegate; Cloud Run accepts
that token normally.

```python
# The client library, not a hand-built request: the agent's credential is
# certificate-bound and must be presented over mTLS (F10).
from google.cloud import iam_credentials_v1

creds, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])
token = iam_credentials_v1.IAMCredentialsClient(credentials=creds).generate_id_token(
    name=f"projects/-/serviceAccounts/{DELEGATE_SA}",
    audience=CLOUD_RUN_SERVICE_URL,      # the service root, not the path
    include_email=True,
).token
# then: Authorization: Bearer <token>  to the Cloud Run service
```

Two bindings, both required and both narrow:

| Grant | On | To |
| :--- | :--- | :--- |
| `roles/iam.serviceAccountOpenIdTokenCreator` | the delegate SA | the agent's `principal://…` (F11) |
| `roles/run.invoker` | the target service | the delegate SA |

Use `openIdTokenCreator`, not `serviceAccountTokenCreator`: it carries
`getOpenIdToken` alone, so the agent can mint ID tokens as the delegate and
nothing else. (`gcloud auth print-identity-token --impersonate-service-account`
additionally needs `getAccessToken`, because it fetches an access token first —
the API call above does not.)

- **Gives** everything D gives, plus Cloud Run targets, with no VPC, PSC, DNS or gateway, and no auth code in the receiving service
- **Costs** per-agent attribution *at the receiver*: Cloud Run sees the delegate SA, not the agent. The agent-to-delegate step is recorded in the IAM audit log instead
- **Choose when** any target is on Cloud Run and gateway policy is not otherwise required. Prefer it over E
- **Do not choose when** the receiver must authorize on the calling agent's own identity — move the target to Agent Runtime (D) or use the gateway (E)

The agent's own credential never leaves the runtime and is never sent to the
target: it only authorizes the token request.

## Agent Identity on Cloud Run

Undocumented. Exposed only as `hidden=True` flags on the **alpha** track, so it
appears in neither `--help` on any track, the Cloud Run v1/v2 API schemas, nor
the IAM documentation — which lists only Agent Runtime and Gemini Enterprise as
supporting Agent Identity.

```bash
gcloud alpha run services update SERVICE --region REGION \
  --identity-type=agent-identity \
  --functional-type=agent            # registers it in Agent Registry
# same flags on: gcloud alpha run deploy
```

The flags set annotations, so the API reaches them without alpha gcloud:

| Annotation | Value |
| :--- | :--- |
| `run.googleapis.com/identity-type` | `agent-identity` \| `workload-identity` \| `service-account` |
| `run.googleapis.com/identity-certificate-enabled` | `true` — set automatically |
| `run.googleapis.com/identity` | `//TRUST_DOMAIN/ns/NAMESPACE/sa/NAME` (managed workload identity) |
| `apphub.cloud.google.com/functional-type` | `agent` \| `mcp-server` |

The resulting principal differs from the Agent Runtime form only in its resource
path:

```
principal://agents.global.org-<ORG_ID>.system.id.goog/resources/run/projects/<PROJECT_NUMBER>/locations/<REGION>/services/<SERVICE>
```

`spec.template.spec.serviceAccountName` still shows a service account
afterwards; it is not what authenticates. Measure the principal from audit logs
(`agent-platform-a2a` §2) rather than reading it off the service.

**Grant everything explicitly (F14).** An Agent Runtime agent identity arrives
with `roles/aiplatform.agentDefaultAccess` and similar; a Cloud Run one arrives
with nothing. Switching identity type silently drops every permission the
service had, and the first symptom is a **403** on model access
(`aiplatform.endpoints.predict`) — a 403, not a 401, because the new credential
authenticates correctly and simply holds no roles.

## State for an agent that is not on Agent Runtime

Agent Runtime injects the engine id its runtime was deployed to, so Sessions and
Memory Bank resolve with no configuration. Every other host injects nothing, and
the usual factory then falls through to an in-process store without saying so
(see `agent-platform-state` S13).

The engine and the code that runs on it are separate concerns. Creating one with
only a display name yields a `reasoningEngines` resource with no `packageSpec` --
nothing to invoke, but a valid target for Sessions and Memory Bank:

```python
engine = client.agent_engines.create(config={"display_name": "my-agent"})
# session_service_uri = f"agentengine://{engine.resource_name}"
```

Give each agent its own. The engine id selects the session store as well as the
memory store, so pointing two agents at one relocates one agent's sessions into
the other's resource, and buys nothing: Memory Bank isolates by scope regardless.

**Find-or-create at startup rather than provisioning ahead of deploy.** A build
order that creates the engine first has to thread its id into the deploy, and on
a fresh project the lookup returns empty before the first deploy has run. ADK's
scaffolding takes the other path -- list by display name, create when absent --
which is idempotent and removes the ordering problem:

```python
existing = list(agent_engines.list(filter=f"display_name={name}"))
engine = existing[0] if existing else agent_engines.create(display_name=name)
```

Generated by `agents-cli scaffold --session-type agent_platform_sessions`. The
flag is documented; this behaviour is not, and it only matters off Agent Runtime,
which overrides `session_type` anyway.

## Choosing

```
Do agents need to call each other?
├── no ──────────────────────────────────▶ A. Fan-out
└── yes
    └── Is per-agent identity required?
        ├── no ───────────────────────────▶ B. Shared identity
        └── yes
            └── Must sub-agents be independently deployable?
                ├── no ───────────────────▶ C. Composed
                └── yes
                    └── Is gateway policy enforcement required?
                        ├── yes ──────────▶ E. Gateway
                        └── no
                            └── Is any target on Cloud Run?
                                ├── no ───▶ D. Client transport
                                └── yes
                                    └── Must the receiver authorize
                                        on the agent's own identity?
                                        ├── no ──▶ F. Delegated SA
                                        └── yes ─▶ E. Gateway, or move
                                                   the target to Agent
                                                   Runtime (D)
```

Gateway policy is now the *only* thing that forces E. Cloud Run targets, which
used to, are handled by F.

## Exposing an agent in Gemini Enterprise

Orthogonal to the above: how GE reaches the agent.

| Registration | GE invokes | A2UI | Auth requirement |
| :--- | :--- | :---: | :--- |
| `adkAgentDefinition` | `:streamQuery` on the engine | ❌ | Discovery Engine service agent + `aiplatform.reasoningEngines.query` |
| `a2aAgentDefinition` on Agent Runtime | the card's `url` | ✅ | Authorization with `cloud-platform` scope ⇒ per-user OAuth consent |
| `a2aAgentDefinition` on Cloud Run | the card's `url` | ✅ | Discovery Engine service agent + `roles/run.servicesInvoker` |

Rules:
- **A2UI ⇒ A2A registration.** No exceptions (F4)
- **A2UI ⇒ Cloud Run.** Agent Runtime cannot serve it at all (F12, below)
- **A2A on Agent Runtime ⇒ per-user OAuth consent.** To avoid it, host on Cloud Run
- A2A agents registered this way **bypass Agent Gateway policy** (the console says so on the agent's page)

### Why A2UI needs Cloud Run

A2UI is not simply "emit the right JSON". It is an A2A **extension**, and A2A
negotiates extensions per request:

1. the client sends `X-A2A-Extensions: <uri>` to activate it;
2. the server activates it and **echoes that URI in the response header**;
3. only then may the client interpret the `application/json+a2ui` data parts.

Miss step 2 and a conforming client discards the surface. It does not error — it
renders **nothing**, because the parts arrived but were never licensed for use.

On Agent Runtime step 2 is impossible (F12). Verified by marking every response
with an extra header in middleware and observing that it never reaches the
caller: only Google frontend headers arrive (`server: scaffolding on HTTPServer2`,
`alt-svc`, `x-frame-options`, …), none from the container. Server-side logging
confirmed the other three steps all worked — the request header arrived, the
extension activated, the SDK wrote the echo — and the proxy still discarded it.

The failure is silent and symmetrical with the *other* A2UI dead end, so the two
are easily confused:

| Registration | Symptom | Cause |
| :--- | :--- | :--- |
| `adkAgentDefinition` | surface arrives as **raw JSON text** | no agent card, so A2UI is never negotiated (F4) |
| `a2aAgentDefinition` on Agent Runtime | **blank reply**, no error | echo stripped by the passthrough (F12) |
| `a2aAgentDefinition` on Cloud Run | renders | app-set headers reach the caller |

Activating the extension server-side is still required wherever it is hosted —
ADK's executor does not do it, so it must be added:

```python
from a2ui.a2a.extension import try_activate_a2ui_extension

class _A2uiActivatingExecutor(A2aAgentExecutor):
    async def execute(self, context, event_queue) -> None:
        try_activate_a2ui_extension(context, agent_card)   # sets the echo
        await super().execute(context, event_queue)
```

Converting the emitted parts is a separate step from activating the extension,
and ADK dispatches to **one of two executors per request** — so a converter
attached to only one of them works under `curl` and not under Gemini
Enterprise. See `agent-platform-implementation` I1.

### Wiring the Authorization

`authorizationConfig` is a **top-level field on `Agent`**, not nested inside a
definition type — so it applies to `a2aAgentDefinition` as much as to
`adkAgentDefinition`. Two fields, and they are not interchangeable:

| Field | Token is sent | Purpose |
| :--- | :--- | :--- |
| `agentAuthorization` | in the request **auth header** | authorizes GE to **invoke the agent** — this is the one that fixes `CREDENTIALS_MISSING` |
| `toolAuthorizations` | in the request **body** | lets the agent access **other resources** on the user's behalf |

```jsonc
// 1. create the Authorization resource
POST …/projects/{project}/locations/{loc}/authorizations?authorizationId={id}
{ "serverSideOauth2": { "clientId": …, "clientSecret": …,
                        "authorizationUri": …, "tokenUri": … } }

// 2. reference it from the agent
{ "authorizationConfig": {
    "agentAuthorization": "projects/{project}/locations/{loc}/authorizations/{id}" } }
```

The OAuth client must be a **Web application** with both redirect URIs
registered: `https://vertexaisearch.cloud.google.com/oauth-redirect` and
`https://vertexaisearch.cloud.google.com/static/oauth/oauth.html`.

**Documentation trap:** this lives under a section headed *"Add the authorization
resource to Gemini Enterprise (optional)"*, and the surrounding prose frames it as
delegated access — "if you want the agent to access Google Cloud resources on
behalf of you", which describes `toolAuthorizations`. The requirement is a
conditional sentence *inside* that optional section: "If your A2A agent is hosted
on Agent Runtime … you must create an authorization with `cloud-platform` scope."
Optional as a feature; mandatory for this configuration.

Consequence worth planning for: the token is the **end user's**, so the caller at
the engine is the user, not the Discovery Engine service agent. Every GE user of
the agent needs `aiplatform.reasoningEngines.query` on it.

## Dead ends

| Attempt | Result |
| :--- | :--- |
| A2UI over `adkAgentDefinition` | Surface arrives as raw JSON text. Emitting it as an `inline_data` blob instead changes nothing — GE does not look for a capability it was never offered |
| A2UI over `a2aAgentDefinition` **on Agent Runtime** | Blank reply, no error. The `/api/` passthrough strips the `X-A2A-Extensions` echo, so the client may not interpret the surface (F12). Host on Cloud Run — no code change fixes this |
| Debugging that blank reply by validating the A2UI payload | Wasted effort. Correct v0.8 extension URI, standard catalog, valid components and a `state: completed` task all coexist with rendering nothing — the negotiation fails, not the payload. Note the converse once negotiation works: a valid catalog component can still be dropped, because GE implements a subset — see `agent-platform-implementation` I12 |
| `RemoteA2aAgent` under Agent Identity | 401 on every call, with correct principal, scope and IAM. Change the transport (D) or add a gateway (E) |
| Fixing that 401 with IAM | Impossible. A 401 is a rejected credential, not a denied permission; there is no authorization decision to grant |
| Cloud Run target on the agent's **own** credential | The MDS `identity?audience=` endpoint returns 200 but not a token Cloud Run accepts, and the agent's access token is rejected too. Mint as a delegate SA instead (F) |
| Hand-built POST to `iamcredentials.googleapis.com` to mint that delegated token | 401 `UNAUTHENTICATED`, *"Request had **invalid** authentication credentials"* — the bound credential must go over mTLS. Use `iam_credentials_v1.IAMCredentialsClient` (F10) |
| Reading a 200 from the metadata server as proof an ID token is usable | It is not. Under Agent Identity the response is not a credential Cloud Run will accept; only the receiving service's status code settles it |
| Agent Gateway without PSC networking | Worse than nothing: the engine loses its own session calls and every turn fails (F8) |
| A2A registration on Agent Runtime without an Authorization | `CREDENTIALS_MISSING` — the body says the credential is *missing*, so GE sends none at all. The request is refused at the Google frontend, leaving no audit entry and nothing to grant |
| Attaching a gateway to enable Agent Identity | `identity_type` is fixed at creation; the engine must be redeployed |
| aiplatform's built-in A2A client wrapper against an a2a-sdk 1.x engine | Hard-coded to the 0.3.x surface; cannot call it |

## Build order for E

The only architecture with an ordering constraint; getting it wrong takes the
agent down (F8).

1. Deploy agents with `--agent-identity`
2. Create VPC, subnet, network attachment, DNS zone, and the gateway
3. Grant the AI Platform service agent (`service-<PROJECT_NUMBER>@gcp-sa-aiplatform`, **not** the `-re` service agent) network-attachment access and `roles/dns.peer`
4. Redeploy each agent with `--network-attachment` and `--dns-peering-*`
5. Attach the gateway to the engines

Steps 4 and 5 are not interchangeable. Verify from logs, never from a
successful-looking response — see `agent-platform-a2a`.

**If E is abandoned after step 4**, remove the network attachment from the
engines. An engine left on the network with no gateway attached still resolves
`*.mtls.googleapis.com` through the private zone, which answers for the domain
but holds no records for those hosts — so the engine's own session calls fail
`Name or service not known` and every turn dies before reaching any sub-agent
(F8). Clearing `spec.deploymentSpec.pscInterfaceConfig` restores public
resolution and takes about a minute, against 3–10 for a redeploy.

## Build order for F

No ordering constraint, and nothing to take down if a step is missed — each
failure is a clean 401 or 403 at the hop being built.

1. Deploy agents with `--agent-identity` and Architecture D's transport
2. Create the delegate service account
3. Grant the agent's `principal://…` `roles/iam.serviceAccountOpenIdTokenCreator` **on the delegate**
4. Grant the delegate `roles/run.invoker` **on the target service**
5. Allow ~3 minutes for IAM propagation before concluding a step failed

Determine the agent's principal by measurement, not from documentation — see
`agent-platform-a2a` §2.
