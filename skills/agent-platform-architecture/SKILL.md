---
name: agent-platform-architecture
description: >
  This skill should be used when designing how agents fit together on Gemini
  Enterprise Agent Platform: "should this be one agent or several", "how do I
  expose an agent in Gemini Enterprise", "do I need Agent Gateway", "should I
  enable Agent Identity", "can these agents call each other", "how do agents
  authenticate to each other", "why does A2UI not render", or when choosing
  between Agent Runtime and Cloud Run. Gives the platform constraints, the
  transport compatibility matrix for agent-to-agent calls, and the architectures
  they permit. Use agent-platform-a2a for debugging a built deployment.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.4.0
---

# Agent architecture on Agent Platform

## Quick answers

| Question | Answer |
| :--- | :--- |
| Do agents calling each other require Agent Gateway? | **No.** Only a transport the identity mode supports. See [Transport matrix](#transport-matrix) |
| Does Agent Identity break agent-to-agent A2A? | It breaks **plain-bearer transports** (`RemoteA2aAgent`). Use the genai client transport instead |
| When is Agent Gateway actually required? | Calling a **Cloud Run** target under Agent Identity, or when you need gateway policy enforcement |
| Can `adkAgentDefinition` render A2UI? | **Never.** A2UI is negotiated from an agent card, which only A2A registration has |
| Does running N separate engines cost extra? | **No.** Independent deployment is free; only the calling pattern has consequences |
| Cheapest way to get per-agent identity? | Do not make agent-to-agent calls (Architecture A), or keep sub-agents in-process (C) |

## Two independent axes

Design decisions on this platform reduce to two choices that compose:

1. **Identity mode** — service account (default) or Agent Identity
2. **Hop transport** — how one agent reaches another, if at all

Everything else follows.

## Forces

Rules, not preferences. `⇒` means the platform gives no choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| F1 | Agent Identity replaces the service account with a SPIFFE principal bound by mTLS/DPoP | ⇒ no bearer tokens; a plain `Authorization: Bearer` call fails **401** and no IAM grant fixes it |
| F2 | Google client libraries implement that binding; hand-rolled HTTP does not | ⇒ **the transport determines whether a call works under Agent Identity**, not the protocol |
| F3 | Agent Identity has no service account | ⇒ audience-bound ID tokens cannot be minted ⇒ **Cloud Run targets are unreachable directly** |
| F4 | Gemini Enterprise learns A2UI support from the agent card | ⇒ A2UI requires `a2aAgentDefinition`; `adkAgentDefinition` carries no card and can never render A2UI |
| F5 | A2A registration means GE calls the card's `url` directly | ⇒ Agent Runtime needs an Authorization with `cloud-platform` scope; Cloud Run needs `roles/run.servicesInvoker` on the Discovery Engine service agent |
| F6 | Agent Gateway terminates the mTLS handshake for the caller | ⇒ it makes any transport work under Agent Identity, and is the only option for Cloud Run targets (F3) |
| F7 | The gateway endpoint is a PSC service attachment in a Google tenant project | ⇒ VPC, subnet, network attachment, DNS peering, and every agent redeployed onto it |
| F8 | Attaching a gateway moves the engine's Google API traffic to `*.mtls.googleapis.com` | ⇒ a gateway the agent cannot reach breaks it **worse than having none**: the engine's own session calls fail |
| F9 | Independent deployment costs nothing; only the calling pattern matters | ⇒ "several engines" and "agents calling each other" are separate decisions |

Verified by observation: F1, F2, F3, F4, F6, F8, F9. From documentation: F5, F7.

## Transport matrix

The central table. Rows are ways one agent can reach another; columns are the
caller's identity mode.

| Transport | Service account | Agent Identity | Notes |
| :--- | :---: | :---: | :--- |
| **In-process sub-agent** (no network) | ✅ | ✅ | No credential involved at all |
| **genai client** `request()` to `…/a2a/message:send` | ✅ | ✅ | Binding-aware; the transport `stream_query` uses |
| **ADK `RemoteA2aAgent`** (httpx + bearer) | ✅ | ❌ 401 | Plain bearer; cannot satisfy DPoP |
| **Hand-rolled httpx + ADC bearer** | ✅ | ❌ 401 | Same failure as above |
| **Agent Gateway** | ✅ | ✅ | Terminates mTLS for the caller; adds policy enforcement |

And by **target**, under Agent Identity:

| Target hosting | Reachable under Agent Identity | How |
| :--- | :--- | :--- |
| Agent Runtime engine | ✅ | genai client transport, or Agent Gateway |
| Cloud Run service | ❌ directly (F3) | Agent Gateway, or move the target to Agent Runtime |

### The genai transport, concretely

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

Two requirements that fail obscurely if missed:

- **`A2A-Version: 1.0` header.** A missing version header is read as `0.3` and the call returns `VERSION_NOT_SUPPORTED`.
- **Do not use aiplatform's own A2A client wrapper.** It is hard-coded to the a2a-sdk 0.3.x surface and cannot call a 1.x engine.

## Architectures

| | Identity | Hop | Transport | Extra infrastructure |
| :--- | :--- | :--- | :--- | :--- |
| **A. Fan-out** | Agent Identity | none | — | none |
| **B. Shared identity** | service account | yes | `RemoteA2aAgent` | none |
| **C. Composed** | Agent Identity | in-process | — | none |
| **D. Client transport** | Agent Identity | yes | genai client | none |
| **E. Gateway** | Agent Identity | yes | Agent Gateway | VPC + PSC + DNS + IAM |

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
- **Costs** a custom A2A client module; you leave ADK's `RemoteA2aAgent` behind
- **Requires** the target to be an Agent Runtime engine (F3)
- **Choose when** you need both per-agent identity and independently deployed sub-agents. **This is the default answer to that requirement** — prefer it over E

### E. Gateway — hop under Agent Identity, with policy enforcement
Agent Identity with Agent Gateway carrying the hop.
- **Gives** everything D gives, plus Cloud Run targets and centralised policy enforcement
- **Costs** VPC, subnet, PSC network attachment, private DNS zone, additional IAM on the AI Platform service agent, and a redeploy of every agent onto the network attachment
- **Requires** engines created with `identity_type=AGENT_IDENTITY`; attaching a gateway does not change it
- **Choose when** the target is on Cloud Run, or gateway policy enforcement is a requirement. Not needed merely to make A2A work

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
                    └── Is any target on Cloud Run,
                        or is gateway policy required?
                        ├── no ───────────▶ D. Client transport
                        └── yes ──────────▶ E. Gateway
```

## Exposing an agent in Gemini Enterprise

Orthogonal to the above: how GE reaches the agent.

| Registration | GE invokes | A2UI | Auth requirement |
| :--- | :--- | :---: | :--- |
| `adkAgentDefinition` | `:streamQuery` on the engine | ❌ | Discovery Engine service agent + `aiplatform.reasoningEngines.query` |
| `a2aAgentDefinition` on Agent Runtime | the card's `url` | ✅ | Authorization with `cloud-platform` scope ⇒ per-user OAuth consent |
| `a2aAgentDefinition` on Cloud Run | the card's `url` | ✅ | Discovery Engine service agent + `roles/run.servicesInvoker` |

Rules:
- **A2UI ⇒ A2A registration.** No exceptions (F4)
- **A2A on Agent Runtime ⇒ per-user OAuth consent.** To avoid it, host on Cloud Run
- A2A agents registered this way **bypass Agent Gateway policy**

## Dead ends

| Attempt | Result |
| :--- | :--- |
| A2UI over `adkAgentDefinition` | Surface arrives as raw JSON text. Emitting it as an `inline_data` blob instead changes nothing — GE does not look for a capability it was never offered |
| `RemoteA2aAgent` under Agent Identity | 401 on every call, with correct principal, scope and IAM. Change the transport (D) or add a gateway (E) |
| Fixing that 401 with IAM | Impossible. A 401 is a rejected credential, not a denied permission; there is no authorization decision to grant |
| Cloud Run target under Agent Identity, no gateway | No way to mint an audience-bound ID token (F3) |
| Agent Gateway without PSC networking | Worse than nothing: the engine loses its own session calls and every turn fails (F8) |
| A2A registration on Agent Runtime without an Authorization | `CREDENTIALS_MISSING` — GE sends an ID token, the endpoint requires OAuth |
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
