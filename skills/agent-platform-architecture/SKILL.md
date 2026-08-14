---
name: agent-platform-architecture
description: >
  This skill should be used when designing how agents fit together on Gemini
  Enterprise Agent Platform: "should these be one agent or several", "how do I
  expose an agent in Gemini Enterprise", "do I need Agent Gateway", "should I
  turn on Agent Identity", "can these agents call each other", "why does A2UI
  not render", or when choosing between Agent Runtime and Cloud Run. Gives the
  constraints that govern the design space and the four architectures they
  permit. Use agent-platform-a2a instead for debugging a deployment that is
  already built.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.2.0
---

# Agent architecture on Agent Platform

The design space is smaller than it looks. A handful of platform constraints
compose, and only four architectures survive them. Most of the cost sits behind
a single question.

## The question that decides everything

> **Do you need per-agent identity *and* an agent-to-agent hop?**

Only "yes to both" is expensive. Everything else is nearly free.

```
per-agent identity?  ──no──▶  agent→agent hop?  ──yes──▶  B. Orchestrator, shared identity
        │                            └──no───▶            (any of A/B/C works)
       yes
        │
        ▼
  agent→agent hop?  ──no──▶  A. Fan-out
        │
       yes
        ▼
  C. Composed orchestrator   (drop the hop, keep both)
        or
  D. Orchestrator + per-agent identity   (keep the hop, pay for Agent Gateway)
```

## The forces

Each is an implication, not a preference. `⇒` means the platform gives you no
choice.

| # | Force | Consequence |
| :--- | :--- | :--- |
| F1 | Agent Identity replaces the service account with a SPIFFE principal bound by mTLS/DPoP | ⇒ no bearer tokens. Any hand-rolled `Authorization: Bearer` call fails **401**, and no IAM grant fixes it |
| F2 | Google client libraries implement that binding; plain HTTP does not | ⇒ SDK calls keep working under Agent Identity; `RemoteA2aAgent` does not |
| F3 | Gemini Enterprise learns A2UI support from the **agent card** | ⇒ A2UI requires `a2aAgentDefinition`. `adkAgentDefinition` carries no card, so it can never render A2UI |
| F4 | A2A registration means GE calls the card's `url` directly | ⇒ Agent Runtime needs an Authorization with `cloud-platform` scope (per-user OAuth); Cloud Run needs `roles/run.servicesInvoker` on the discoveryengine SA |
| F5 | Agent Gateway terminates the mTLS handshake for the caller | ⇒ it is the only supported way to keep an agent→agent hop under Agent Identity |
| F6 | The gateway's endpoint is a PSC service attachment in a Google tenant project | ⇒ VPC + subnet + network attachment + DNS peering, and **every agent redeployed onto it** |
| F7 | Attaching a gateway flips the engine's Google API traffic to `*.mtls.googleapis.com` | ⇒ a gateway the agent cannot reach breaks it **worse than not having one** — session calls fail, every turn errors |
| F8 | Independent deployment costs nothing; only the **hop** costs | ⇒ "several engines" and "agents calling each other" are different decisions. Do not conflate them |

Verified on live deployments: F1, F2, F3, F5, F7, F8. From documentation: F4, F6.

## The four architectures

### A. Fan-out — *N independent agents, no hop*

Several separately deployed engines, each serving users directly. No agent calls
another.

```
user ──▶ agent 1     (Agent Identity)
user ──▶ agent 2     (Agent Identity)
user ──▶ agent N     (Agent Identity)
```

- **Gives:** per-agent identity, independent deploy/scale/version, per-agent audit attribution
- **Costs:** nothing beyond the engines
- **Requires:** that your fan-out is user→agent, not agent→agent
- **In the wild:** ensemblr runs five engines this way, all `identity_type=AGENT_IDENTITY`, **no gateway anywhere** in their infrastructure. Their ADR reserves A2A for agents they do not own.

The cheapest way to have per-agent identity. If your agents do not genuinely need
to call each other, stop here.

### B. Orchestrator, shared identity — *hop, no per-agent identity*

An orchestrator calls remote sub-agents over A2A. Everything runs as the default
service agent.

```
user ──▶ orchestrator ──A2A──▶ sub-agent (Agent Runtime)
                       ──A2A──▶ sub-agent (Cloud Run)
```

- **Gives:** independent sub-agents, working A2A, zero extra infrastructure
- **Costs:** one identity for all agents — no per-agent authorization or attribution
- **Requires:** a service account runtime (i.e. *not* Agent Identity)
- **In the wild:** upstream luncher (`mbychkowski/luncher`). Its deploy commands pass no identity flags at all, which is exactly why its A2A works.

The default, and it works because F1 never bites.

### C. Composed orchestrator — *hop removed, both kept*

Sub-agents become ordinary ADK objects in the orchestrator's process instead of
`RemoteA2aAgent` proxies.

```
user ──▶ orchestrator process
           ├── sub-agent  (in-process)
           └── sub-agent  (in-process)
```

- **Gives:** per-agent identity for the deployed unit, no credentials between components, lower latency, failures raise instead of degrading
- **Costs:** sub-agents are no longer independently deployable or callable by anyone else
- **Requires:** that you own all the agents

Google's ADK guidance points here directly: use A2A when the target is "a
separate, standalone service" or "maintained by a different team or
organization"; prefer local sub-agents for "internal code organization". Agents
in one repo with one deploy script meet none of the A2A criteria.

### D. Orchestrator + per-agent identity — *both, via Agent Gateway*

Architecture B with Agent Identity, which breaks A2A (F1), recovered with a
gateway (F5).

```
user ──▶ orchestrator (Agent Identity, PSC interface)
             │
        Agent Gateway  ──mTLS──▶ sub-agents
```

- **Gives:** everything — per-agent identity, independent sub-agents, working A2A
- **Costs:** VPC, subnet, PSC network attachment, private DNS zone, an undocumented IAM set on the AI Platform service agent, and a redeploy of every agent onto the network attachment
- **Requires:** engines created with `identity_type=AGENT_IDENTITY` — attaching a gateway does **not** change it, so ordering is fixed
- **In the wild:** this build. No reference implementation found doing it.

Choose deliberately. It is the only quadrant that costs infrastructure, and the
requirement that lands you here is narrow: per-agent identity *and* a hop.

## Comparison

| | A. Fan-out | B. Shared identity | C. Composed | D. Gateway |
| :--- | :--- | :--- | :--- | :--- |
| Per-agent identity | ✅ | ❌ | ✅ | ✅ |
| Agent→agent A2A | n/a | ✅ | n/a | ✅ |
| Sub-agents independently deployable | ✅ | ✅ | ❌ | ✅ |
| Extra infrastructure | none | none | none | VPC + PSC + DNS + IAM |
| Callable by other teams | ✅ | ✅ | ❌ | ✅ |

## Exposing an agent in Gemini Enterprise

An orthogonal axis: how GE reaches whichever architecture you chose.

| Registration | GE invokes | A2UI | Auth |
| :--- | :--- | :--- | :--- |
| `adkAgentDefinition` | `:streamQuery` on the engine | **never** (F3) | discoveryengine SA + `aiplatform.reasoningEngines.query` |
| `a2aAgentDefinition`, Agent Runtime | the card's `url` | ✅ | Authorization with `cloud-platform` scope ⇒ per-user OAuth consent |
| `a2aAgentDefinition`, Cloud Run | the card's `url` | ✅ | discoveryengine SA + `roles/run.servicesInvoker` |

**If you need A2UI, you need A2A registration.** If you also want to avoid
per-user OAuth consent, host on Cloud Run. Note that A2A agents registered this
way bypass Agent Gateway policy entirely.

## Dead ends

Combinations that do not work, and what they look like when you try.

| Attempt | Result |
| :--- | :--- |
| A2UI over `adkAgentDefinition` | Surface arrives as raw JSON text in the reply. Emitting it as an `inline_data` blob instead was tested and changed nothing — GE has no reason to look for a capability it was never offered |
| Agent Identity + A2A, no gateway | 401 on every sub-agent call. Correct principal, correct scope, correct IAM, still 401 |
| Agent Identity + A2A, fixed with IAM | Impossible. A 401 is a rejected credential, not a denied permission — there is no authorization decision to grant |
| Agent Gateway without PSC networking | Worse than nothing: engine loses its own session calls, every turn fails |
| A2A registration on Agent Runtime, no Authorization | `CREDENTIALS_MISSING` — GE sends an ID token, the endpoint wants OAuth |
| Attaching a gateway to enable Agent Identity | `identity_type` is fixed at creation; the engine must be redeployed |

## Build order for D

The only architecture with an ordering constraint, and getting it wrong takes
the agent down (F7):

1. Deploy the agents with `--agent-identity`
2. Create the network and gateway (VPC, subnet, attachment, DNS zone)
3. Grant the AI Platform service agent — `service-<N>@gcp-sa-aiplatform`, not
   the `-re` one — attachment access and `roles/dns.peer`
4. Redeploy each agent with `--network-attachment` and `--dns-peering-*`
5. **Then** attach the gateway to the engines

Steps 4 and 5 are not interchangeable. Verify from logs, never from a
successful-looking response — see `agent-platform-a2a`.
