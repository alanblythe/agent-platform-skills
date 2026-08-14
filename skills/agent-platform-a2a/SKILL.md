---
name: agent-platform-a2a
description: >
  This skill should be used when the user wants to "connect two agents",
  "call one agent from another", "set up A2A", "register an agent with
  Gemini Enterprise", "make A2UI render in Gemini Enterprise", or is
  debugging 401/403 errors between agents on Gemini Enterprise Agent
  Platform. Covers ADK vs A2A registration, which one can render A2UI,
  the authentication each path requires, Agent Identity principals and
  scopes, Agent Gateway, and how to identify a caller from audit logs.
  Do NOT use for writing agent code or for eval.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.1.0
  requires:
    bins:
      - gcloud
      - agents-cli
    install: "uv tool install google-agents-cli"
---

# ADK and A2A communication on Agent Platform

Getting two agents to talk, or Gemini Enterprise (GE) to talk to one, fails in ways
that look alike and are not. Read [Diagnose first](#diagnose-first) before changing
any IAM, because the most common failure here is not an IAM problem at all.

## The one distinction that matters

| Code | Meaning | Fix |
| :--- | :--- | :--- |
| **403** `PERMISSION_DENIED` | The credential was accepted; the principal lacks the role. | An IAM grant. |
| **401** `UNAUTHENTICATED` | The credential was rejected. No authorization decision happened. | Never an IAM grant. Wrong credential *type*, missing scope, or a binding-aware credential replayed as a plain bearer token. |

A 401 leaves **no audit entry** — `protoPayload.status.code=7` returns nothing, because
nothing was denied. An empty denial query alongside a failing call is itself the signal
that you are chasing the wrong layer.

## Diagnose first

Do not infer the caller from a resource field. Measure it.

```bash
# 1. Turn on Data Access logging for the service (off by default).
gcloud projects get-iam-policy PROJECT --format=json > /tmp/p.json
jq '.auditConfigs = [{"service":"aiplatform.googleapis.com",
      "auditLogConfigs":[{"logType":"ADMIN_READ"},{"logType":"DATA_READ"},{"logType":"DATA_WRITE"}]}]' \
  /tmp/p.json > /tmp/p2.json
gcloud projects set-iam-policy PROJECT /tmp/p2.json     # preserves existing bindings

# 2. Reproduce the call, then read who actually called.
gcloud logging read 'protoPayload.serviceName="aiplatform.googleapis.com"' \
  --project PROJECT --limit 5 --freshness=10m \
  --format="value(protoPayload.authenticationInfo.principalSubject,
                  protoPayload.methodName, protoPayload.status.code)"
```

`principalSubject` is ground truth. Turn the audit config off again afterwards — Data
Access logs bill by volume.

## Gemini Enterprise → your agent

Two registration modes, and the choice is not cosmetic.

| | `adkAgentDefinition` | `a2aAgentDefinition` |
| :--- | :--- | :--- |
| GE invokes | `:streamQuery` on the reasoning engine | the `url` inside the agent card |
| Carries a card | no — only `provisionedReasoningEngine` | yes — `jsonAgentCard` |
| **Renders A2UI** | **no** | **yes** |
| Auth, Agent Runtime | discoveryengine SA + `aiplatform.reasoningEngines.query` | an **Authorization with `cloud-platform` scope** |
| Auth, Cloud Run | n/a (no reasoning engine) | discoveryengine SA + `roles/run.servicesInvoker` |
| Agent Gateway applies | yes | **no** |

### Why ADK registration cannot render A2UI

GE learns an agent can render A2UI from the **A2UI extension in its agent card**
(`https://a2ui.org/a2a-extension/a2ui/v0.8`). `adkAgentDefinition` has exactly one
field — the reasoning engine resource — so there is no card, therefore no capability
negotiation, therefore no reason for GE to look for surfaces. The agent's payload
arrives as literal text in the reply.

This is a protocol fact, not a bug to code around. Emitting the surface some other way
does not help: adding a `Part(inline_data=Blob(mime_type="application/json+a2ui", …))`
to the response was tested and changed nothing. Do not spend time on it.

### The caller identity for ADK registration

GE calls as the Discovery Engine service agent of the **GE app's** project, which is
often a *different* project from the agent's:

```
service-<GE_PROJECT_NUMBER>@gcp-sa-discoveryengine.iam.gserviceaccount.com
```

Grant it query access on the engine resource, not project-wide:

```bash
# Minimal role: aiplatform.reasoningEngines.query (+ .get). No predefined role is
# narrower than roles/aiplatform.user, which carries hundreds of permissions.
gcloud iam roles create geAgentCaller --project AGENT_PROJECT \
  --title "Gemini Enterprise Agent Caller" \
  --permissions aiplatform.reasoningEngines.query,aiplatform.reasoningEngines.get --stage GA
```

Then bind it with `reasoningEngines/<ID>:setIamPolicy`.

### A2A registration on Agent Runtime

Supported, but it needs an OAuth Authorization — a plain registration gets
`CREDENTIALS_MISSING`, because GE's A2A client presents an ID token while
`aiplatform.googleapis.com` requires an OAuth access token.

> If your A2A agent is hosted on Agent Runtime, you must create an authorization with
> `cloud-platform` scope.

The consequence is a per-user model: GE sends the **end user's** OAuth token, so every
GE user needs `aiplatform.reasoningEngines.query` on the engine and must consent once.
Attach it with `agents-cli publish gemini-enterprise --authorization-id …`.

If per-user consent is unwanted, host on Cloud Run instead: GE then calls as the
discoveryengine service agent with `roles/run.servicesInvoker`, no consent and no
per-user IAM.

### Registering

Prefer the CLI over hand-rolled REST — it fetches the card and refuses to register one
lacking the A2UI extension:

```bash
# A2A (Cloud Run / GKE)
agents-cli publish gemini-enterprise \
  --agent-card-url https://SERVICE.run.app/a2a/APP_NAME/.well-known/agent-card.json \
  --gemini-enterprise-app-id projects/N/locations/global/collections/default_collection/engines/APP

# ADK (Agent Runtime)
agents-cli publish gemini-enterprise --registration-type adk \
  --agent-runtime-id projects/N/locations/REGION/reasoningEngines/ID \
  --gemini-enterprise-app-id projects/N/.../engines/APP
```

Registration is idempotent — re-running updates in place.

## Agent → agent

### The card path carries the ADK App name

`attach_a2a_routes` mounts at `/a2a/{app.name}`, so an app named `luncher_agent` serves
`/a2a/luncher_agent/.well-known/agent-card.json`, **not** `/a2a/app/…`. Scripts that
hardcode `/a2a/app/` 404 against it, and the 404 looks exactly like a broken deployment.

Agent Runtime also exposes a native A2A REST surface at `{a2a_url}/v1/card` and
`/v1/message:send`, which is distinct from the ADK/a2a-SDK JSON-RPC routes above.

### Sub-agent URLs are env vars derived from the agent name

ADK's `RemoteA2aAgent` wiring typically reads `f"{agent_name.upper()}_URL"`. An agent
named `scheduling_agent` reads `SCHEDULING_AGENT_URL`. **A name that does not match is
ignored silently** and the agent falls back to its local default, so a deployed
orchestrator quietly calls `localhost` and never says so. Verify from the logs:

```
INFO:app.agent:Connecting 'scheduling_agent' using direct URL: https://…
```

### Failure is silent — verify sub-agents, not task state

When `RemoteA2aAgent` cannot resolve a sub-agent, the orchestrator does **not** fail.
The model answers from nothing and returns `state: completed` with confident, fabricated
content. Two 401s produce a plausible reply. Never treat `completed` as success:

```bash
gcloud logging read 'resource.labels.reasoning_engine_id="ID" AND
  (textPayload:"agent-card.json" OR textPayload:"Failed to resolve")' \
  --project PROJECT --limit 6 --freshness=10m --format="value(textPayload)"
```

## Agent Identity

`agents-cli deploy --agent-identity` (Preview) replaces the shared runtime service agent
with a per-agent SPIFFE principal. There is **no `--no-agent-identity`**; a plain
redeploy preserves it.

### Get the principal right

The trust domain differs from what the resource reports, and both forms exist:

| Form | Where it comes from |
| :--- | :--- |
| `agents.global.org-<ORG_ID>.system.id.goog` | documented for org'd projects; also what `spec.effectiveIdentity` reports |
| `agents.global.proj-<PROJECT_NUMBER>.system.id.goog` | observed as the authenticating principal in some projects |
| `agents.global.project-<PROJECT_NUMBER>.system.id.goog` | documented for orgless projects |

```
principal://<TRUST_DOMAIN>/resources/aiplatform/projects/<PROJECT_NUMBER>/locations/<REGION>/reasoningEngines/<ENGINE_ID>
```

Granting the wrong form fails in two distinguishable ways: an accepted-but-**inert**
binding (the role is granted to a principal nothing authenticates as), or
`INVALID_ARGUMENT: Identity Pool does not exist` when that trust domain has no pool in
the project. Neither is a permissions problem. **Measure the form from the audit log**
(see [Diagnose first](#diagnose-first)) rather than reading `spec.effectiveIdentity`.

### It starts with no roles

`--agent-identity` provisions the identity but grants **nothing**. The first failure is
usually not the one you expect — every Google API call fails complaining about
`serviceusage.serviceUsageConsumer`. The set `agents-cli` itself grants:

```
roles/aiplatform.user
roles/serviceusage.serviceUsageConsumer
roles/browser
roles/cloudapiregistry.viewer
roles/logging.logWriter
roles/monitoring.metricWriter
```

Add whatever the agent actually calls — e.g. `roles/run.invoker` on a Cloud Run
sub-agent — bound to the principal above, not to a service account. **An Agent Identity
workload has no service account**, so a `serviceAccount:` member grants it nothing.

### Outbound calls need an explicit scope

A service account carries `cloud-platform` implicitly; an Agent Identity credential does
not. Unscoped ADC yields a 401 that no grant will fix.

```python
credentials, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])
```

Let the credential apply itself rather than lifting the token off it — this also removes
any need to track expiry, which the library handles:

```python
creds.before_request(Request(), "POST", url, headers)   # not: creds.refresh(); creds.token
```

Agent identity explicitly covers "access to other agents hosted on Agent Runtime using
A2A", so this path is supported.

> **Cloud Run sub-agents are a separate credential.** Cloud Run wants an
> audience-bound **ID token**, which ADC does not supply. With no service account to
> mint one from, an Agent Identity workload may be unable to call a Cloud Run agent
> directly — route it through Agent Gateway.

## Agent Gateway

The managed policy-enforcement point for ingress and egress, covering user↔agent,
agent↔tool and agent↔agent. It "automatically handles mTLS handshakes and termination …
without developer effort", which is what makes bound Agent Identity credentials usable
for A2A without the caller implementing mTLS/DPoP.

- No `agents-cli` command yet — Console, REST, or Terraform
  (`google_network_services_agent_gateway`, `google-beta`).
- Needs `networkservices.googleapis.com` enabled.
- Attach egress to an existing engine by patching
  `spec.deploymentSpec.agentGatewayConfig.agentToAnywhereConfig`.
- **Attaching does not change `identity_type`.** An engine not already on Agent Identity
  must be redeployed; you cannot patch your way in.
- A2A agents registered directly with GE bypass the gateway entirely.

## Troubleshooting

| Symptom | Cause |
| :--- | :--- |
| A2UI renders as raw JSON text | Registered as ADK. Only `a2aAgentDefinition` negotiates A2UI. |
| `CREDENTIALS_MISSING` on an A2A call to Agent Runtime | No Authorization with `cloud-platform` scope on the registration. |
| `PERMISSION_DENIED` on `reasoningEngines.query` | The GE discoveryengine SA lacks query on the engine. Check the **GE app's** project number, not the agent's. |
| 401 between agents, IAM looks correct | Missing `cloud-platform` scope, or a bound credential sent as a plain bearer token. |
| `Identity Pool does not exist` | Wrong Agent Identity trust domain for this project. |
| Grant applied, behaviour unchanged | Inert binding — the principal form is not what authenticates. Read `principalSubject`. |
| Agent answers confidently with invented data | A sub-agent failed to resolve. Check card fetches; `state: completed` proves nothing. |
| Card URL 404s | Path carries the ADK `App` name, not `app`. |

## Sources

- [Agent Identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity)
- [Use an A2A agent](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/use-an-a2a-agent)
- [Register A2A agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent)
- [Register ADK agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent)
- [Register an A2UI agent](https://docs.cloud.google.com/gemini/enterprise/docs/a2ui-agents/register-and-manage-an-a2ui-agent)
- [Agent Gateway overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview)
- [Route traffic through Agent Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-gateway-runtime-deploy)
