---
name: agent-platform-a2a
description: >
  This skill should be used when debugging a deployment on Gemini Enterprise
  Agent Platform: 401 or 403 between agents, "permission denied" on a reasoning
  engine, "CREDENTIALS_MISSING", "Identity Pool does not exist", an IAM grant
  that changes nothing, an agent answering confidently with invented data, or a
  deploy failing on an undocumented permission. Covers identifying the real
  caller, why 401 and 403 need opposite responses, Agent Identity principal
  forms, and the conditions under which a broken deployment reports success.
  Use agent-platform-architecture instead for choosing a design.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.4.0
  requires:
    bins:
      - gcloud
---

# Debugging agent auth on Agent Platform

## 1. A 401 is not a 403

| Code | Meaning | Response |
| :--- | :--- | :--- |
| **403** `PERMISSION_DENIED` | Credential accepted, principal lacks the role | Grant something |
| **401** `UNAUTHENTICATED` | Credential rejected; no authorization decision occurred | **Grant nothing.** Wrong credential type, missing scope, or a bound credential sent as a plain bearer token |

A 401 leaves no audit entry — `protoPayload.status.code=7` returns nothing,
because nothing was denied. An empty denial query alongside a failing call
indicates the wrong layer is being investigated.

Under Agent Identity, `RemoteA2aAgent` calls fail 401 because the credential is
mTLS/DPoP-bound while it sends a bearer header. No combination of roles resolves
this — **the transport must change**, not the IAM. Route the call through the
genai client transport, or through Agent Gateway. See
`agent-platform-architecture` for the transport matrix.

## 2. Measure the caller

`spec.effectiveIdentity` and other resource fields can report a different
principal from the one the API authenticates. Read the authenticated principal.

```bash
# Data Access logging is off by default. It bills by volume — disable afterwards.
gcloud projects get-iam-policy PROJECT --format=json > /tmp/p.json
jq '.auditConfigs = [{"service":"aiplatform.googleapis.com",
      "auditLogConfigs":[{"logType":"ADMIN_READ"},{"logType":"DATA_READ"},{"logType":"DATA_WRITE"}]}]' \
  /tmp/p.json > /tmp/p2.json
gcloud projects set-iam-policy PROJECT /tmp/p2.json     # preserves existing bindings

gcloud logging read 'protoPayload.serviceName="aiplatform.googleapis.com"' \
  --project PROJECT --limit 5 --freshness=10m \
  --format="value(protoPayload.authenticationInfo.principalSubject,
                  protoPayload.methodName, protoPayload.status.code)"
```

`principalSubject` is authoritative.

### Agent Identity principal forms

```
principal://<TRUST_DOMAIN>/resources/aiplatform/projects/<PROJECT_NUMBER>/locations/<REGION>/reasoningEngines/<ENGINE_ID>
```

| Trust domain | Notes |
| :--- | :--- |
| `agents.global.org-<ORG_ID>.system.id.goog` | documented for projects in an organization; also what `effectiveIdentity` reports |
| `agents.global.proj-<PROJECT_NUMBER>.system.id.goog` | observed as the authenticating principal in some projects |
| `agents.global.project-<PROJECT_NUMBER>.system.id.goog` | documented for projects without an organization |

Two distinguishable failure modes:

- Wrong but existing trust domain ⇒ the binding is accepted and **inert**; behaviour is unchanged.
- Trust domain with no pool ⇒ `INVALID_ARGUMENT: Identity Pool does not exist`.

Neither is a permissions problem. Determine the form by measurement (§2) rather
than by copying one from documentation.

An Agent Identity workload has no service account, so a `serviceAccount:` member
grants it nothing.

## 3. Read permissions out of the error

Deploys onto a network attachment fail one permission at a time, each surfacing
only after the container builds. The 403 body names the permission:

```
Required 'compute.regionOperations.get' permission for 'projects/.../operations/...'
```

Some errors name a role instead:

```
Please make sure the Vertex AI service account has the dns.peer role
```

Deploy, read the name, grant exactly that, wait for propagation, repeat. Adding
permissions predicted to be needed next grants access that nothing uses and
obscures which were actually required.

Two things obscure this loop:

- **Consecutive failures look identical.** They name the same service account and
  differ only in the method: `NetworkAttachmentsService.Get`, then `.Patch`, then
  `RegionOperationsService.Get`. The second resembles the first grant having failed.
- **IAM propagation takes minutes.** An immediate retry fails identically.
  Allow roughly three minutes before drawing conclusions.

Observed set for deploying an Agent Runtime agent onto a customer-owned network
attachment, granted to `service-<PROJECT_NUMBER>@gcp-sa-aiplatform` — not the
`-re` service agent used elsewhere:

```
compute.networkAttachments.get       # NetworkAttachmentsService.Get
compute.networkAttachments.update    # .Patch — accept-lists its own PSC connection
compute.regionOperations.get         # polls the update it started
roles/dns.peer                       # creating the DNS peering zone
```

`roles/compute.networkUser` is insufficient: it carries `.get` but not `.update`.

## 4. A successful response is not evidence

**A failed sub-agent does not fail the turn.** When `RemoteA2aAgent` cannot
resolve a sub-agent, the model answers without it and the task reports
`state: completed` with fluent, invented content. Two 401s produce a plausible
answer.

```bash
gcloud logging read 'resource.labels.reasoning_engine_id="ENGINE_ID" AND
  (textPayload:"agent-card.json" OR textPayload:"Failed to resolve")' \
  --project PROJECT --limit 6 --freshness=10m --format="value(textPayload)"
```

Both card fetches should return 200.

Related conditions:

- **Engine `state` lags.** It reports `ACTIVE` while an update is in flight, so a
  state-based wait reads a stale configuration and reports success. Poll the
  returned **operation**. A second PATCH during that window is rejected with
  "not in ACTIVE state", so a retry loop conflicts with itself.
- **PATCH is asynchronous** and redeploys the container. A non-error response
  means accepted, not applied.
- **Sub-agent URL variables are derived from the sub-agent name**, typically
  `{AGENT_NAME}_URL`. A name that does not match is ignored silently and the
  agent falls back to its local default, so a deployed orchestrator calls
  `localhost` indefinitely. Confirm from the startup log line.
- **Card paths carry the ADK `App` name**, not the literal `app`. Scripts that
  hardcode `/a2a/app/` return 404 against an app named otherwise, which
  resembles a broken deployment.

## 5. Outbound credentials in agent code

```python
# Request the scope explicitly. A service account carries cloud-platform
# implicitly; an Agent Identity credential does not, and unscoped ADC returns 401.
credentials, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])

# Let the credential apply itself. Reading .token bypasses whatever binding the
# credential implements, and delegates expiry to the library.
creds.before_request(Request(), "POST", url, headers)     # not: refresh(); creds.token
```

Both are correct regardless of identity mode, and neither makes a plain-bearer
transport work under Agent Identity: that needs the genai client transport or
Agent Gateway. Cloud Run targets need an audience-bound ID token, which an Agent
Identity workload cannot mint at all.

A token fetched once at import is its own defect: tokens last about an hour while
a warm instance holds its clients far longer.

## 6. Running Terraform with the intended credential

Go's auth library ignores `CLOUDSDK_CONFIG`. On a machine with multiple gcloud
profiles, Terraform uses `~/.config/gcloud/application_default_credentials.json`
regardless of the active profile, and fails with a 403 resembling a missing role.

```bash
export GOOGLE_OAUTH_ACCESS_TOKEN="$(gcloud auth print-access-token)"
```

Pointing `GOOGLE_APPLICATION_CREDENTIALS` at the profile's ADC file does not work
for Workspace accounts: the Go client rejects the stored credential with
`invalid_rapt` and requires an interactive reauth. Python's `google-auth` does
respect `CLOUDSDK_CONFIG`, so a Python check reports the intended identity while
Terraform uses another.

## Troubleshooting index

| Symptom | Cause |
| :--- | :--- |
| A2UI renders as raw JSON | Registered as ADK; only `a2aAgentDefinition` negotiates A2UI |
| `CREDENTIALS_MISSING` from GE | A2A on Agent Runtime without an Authorization carrying `cloud-platform` scope |
| `PERMISSION_DENIED` on `reasoningEngines.query` | The Discovery Engine service agent of the **GE app's** project lacks query on the engine |
| 401 between agents with correct IAM | Bound credential sent as a bearer token. Fix the transport, not the IAM |
| `VERSION_NOT_SUPPORTED` on an A2A call | Missing `A2A-Version: 1.0` header; a missing version is read as `0.3` |
| `Identity Pool does not exist` | Wrong Agent Identity trust domain for the project |
| Grant applied, behaviour unchanged | Inert binding — that principal form is not what authenticates |
| Confident answer containing invented data | A sub-agent failed to resolve; `state: completed` is not evidence |
| `Cannot connect to ... mtls.googleapis.com` | Gateway attached without PSC networking |
| Card URL returns 404 | Path carries the ADK `App` name |
| Deploy 403s on a compute permission | Grant the named permission to `service-<PROJECT_NUMBER>@gcp-sa-aiplatform` and allow propagation |

## Sources

- [Agent Identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity)
- [Use an A2A agent](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/use-an-a2a-agent)
- [Register A2A agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent)
- [Register ADK agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent)
- [Agent Gateway overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview)
- [Route traffic through Agent Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-gateway-runtime-deploy)
