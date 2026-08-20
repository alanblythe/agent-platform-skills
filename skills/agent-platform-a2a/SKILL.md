---
name: agent-platform-a2a
description: >
  This skill should be used when debugging a deployment on Gemini Enterprise
  Agent Platform: 401 or 403 between agents, "permission denied" on an Agent
  Runtime agent, "CREDENTIALS_MISSING", "Identity Pool does not exist", an IAM grant
  that changes nothing, an agent answering confidently with invented data, a
  deploy failing on an undocumented permission, a 404 for a model that exists,
  or a "global" vs regional location mix-up. Covers identifying the real caller,
  why 401 and 403 need opposite responses, Agent Identity principal forms, which
  region belongs where, and the conditions under which a broken deployment
  reports success. Use agent-platform-architecture instead for choosing a design.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.8.1
  requires:
    bins:
      - gcloud
---

# Debugging agent auth on Agent Platform

Named services of Gemini Enterprise Agent Platform referenced below:
**Agent Runtime** (managed host), **Agent Identity**, **Agent Gateway**, and
**Agent Registry**. Cloud Run is a separate Google Cloud product used as the
alternative host. For **Sessions** and **Memory Bank**, see
`agent-platform-state`.

"Engine" is the API-level noun for a deployed Agent Runtime agent — the REST
resource is `reasoningEngines`, and IAM permissions
(`aiplatform.reasoningEngines.query`) and log fields (`reasoning_engine_id`)
keep that spelling.

## 1. A 401 is not a 403

| Code | Meaning | Response |
| :--- | :--- | :--- |
| **403** `PERMISSION_DENIED` | Credential accepted, principal lacks the role | Grant something |
| **401** `UNAUTHENTICATED` | Credential rejected; no authorization decision occurred | **Grant nothing.** Wrong credential type, missing scope, or a bound credential sent as a plain bearer token |

A 401 leaves no audit entry — `protoPayload.status.code=7` returns nothing,
because nothing was denied. An empty denial query alongside a failing call
indicates the wrong layer is being investigated.

### Two 401s that mean different things

The body distinguishes them, and they have different fixes. Read the wording
before acting:

| Body says | Reason | Meaning | Fix |
| :--- | :--- | :--- | :--- |
| "Request is **missing** required authentication credential" | `CREDENTIALS_MISSING` | Nothing was sent | The *caller* has no credential configured — e.g. a GE A2A registration with no `authorizationConfig` |
| "Request had **invalid** authentication credentials" | `UNAUTHENTICATED` | Something was sent and rejected | A bound credential presented over plain HTTP. Use a client library so it goes over mTLS |

"Missing" points at configuration, "invalid" at transport. Neither points at IAM.

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
# Agent Runtime engine
principal://<TRUST_DOMAIN>/resources/aiplatform/projects/<PROJECT_NUMBER>/locations/<REGION>/reasoningEngines/<ENGINE_ID>

# Cloud Run service running under Agent Identity
principal://<TRUST_DOMAIN>/resources/run/projects/<PROJECT_NUMBER>/locations/<REGION>/services/<SERVICE>
```

On Cloud Run, `serviceAccountName` still shows a service account after the
switch and is not what authenticates. Read the principal from the audit log.

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
  resembles a broken deployment. The ADK REST path takes a *different* name
  again — see `agent-platform-implementation` I2.

**A sub-agent that succeeds can produce the same symptom.** Everything above
assumes the invented content follows a failed call. It also follows a **200**:
only the sub-agent's rendered text crosses the hop, so a fact it looked up but
did not state does not reach the caller, which fills the gap fluently. If both
card fetches return 200 and the answer is still invented, stop looking for an
auth fault and compare what the sub-agent *said* against what the caller needed
— see `agent-platform-implementation` I3.

## 5. Isolating where a header is lost

A protocol that depends on headers has three places to fail, and they are
indistinguishable from the client — all three look like "the feature is off".
Log the middle one and probe the last one rather than guessing:

```python
# In the executor: did the request header arrive, and did we act on it?
logger.info("extension: requested=%s activated=%s",
            sorted(context.requested_extensions), activated)
```

```python
# In middleware: does *any* app-set response header survive the hop?
@app.middleware("http")
async def _probe(request, call_next):
    response = await call_next(request)
    response.headers["X-Probe"] = "container"
    return response
```

| Observation | Conclusion |
| :--- | :--- |
| `requested=[]` | The request header never arrived — the caller or an intermediary dropped it |
| `requested=[…] activated=None` | Arrived but was rejected — usually a URI the agent card does not advertise |
| `activated=…` but the client sees nothing, **and** `X-Probe` is missing | The hop strips response headers. Not a code bug |
| `activated=…`, `X-Probe` arrives, echo missing | A genuine server-side bug worth chasing |

On Agent Runtime the third row is the answer: only Google frontend headers reach
the caller. See `agent-platform-architecture` F12.

## 6. Outbound credentials in agent code

```python
# Request the scope explicitly. A service account carries cloud-platform
# implicitly; an Agent Identity credential does not, and unscoped ADC returns 401.
credentials, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])

# Let the credential apply itself. Reading .token bypasses whatever binding the
# credential implements, and delegates expiry to the library.
creds.before_request(Request(), "POST", url, headers)     # not: refresh(); creds.token
```

Both are correct regardless of identity mode, and neither makes a plain-bearer
transport work under Agent Identity: that needs the genai client transport
(Architecture D) or Agent Gateway (E).

A token fetched once at import is its own defect: tokens last about an hour while
a warm instance holds its clients far longer.

### Reaching Cloud Run under Agent Identity

The runtime cannot mint an audience-bound ID token as itself. The metadata server
**returns 200** for `identity?audience=…`, which reads like success — but Cloud
Run rejects the result 401. Do not treat that 200 as evidence; only the receiving
service's status code settles it.

Mint as a delegate service account instead (Architecture F):

```python
from google.cloud import iam_credentials_v1     # NOT a hand-built POST — see below

token = iam_credentials_v1.IAMCredentialsClient(credentials=creds).generate_id_token(
    name=f"projects/-/serviceAccounts/{DELEGATE_SA}",
    audience=CLOUD_RUN_SERVICE_URL,             # service root, not the request path
    include_email=True,
).token
```

Requires `roles/iam.serviceAccountOpenIdTokenCreator` for the agent's
`principal://…` on the delegate, and `roles/run.invoker` for the delegate on the
target.

**The same 401 lies in wait here.** Calling `iamcredentials.googleapis.com` with a
hand-built request and an `Authorization` header fails
`UNAUTHENTICATED: Request had invalid authentication credentials` — the binding
applies to *every* Google API call, not only agent-to-agent ones. This is the
original 401 in a new place, and the fix is the same: use the generated client.

## 7. Running Terraform with the intended credential

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

## 8. "Location" is three different values

An agent carries several regions that are not interchangeable, and the failures
are quiet: a wrong one either builds an invalid host or a valid host pointing at
the wrong place.

| Value | What it names | Typical |
| :--- | :--- | :--- |
| Model / GenAI location | Which endpoint serves the **model** | often `global` |
| Deploy region | Where the service or engine **runs** | a real region |
| Engine location | The region of the engine holding **sessions and Memory Bank** | a real region |

Two rules follow:

- **`global` is a model endpoint, not a region.** Some model versions are served
  *only* from `global`, and a regional endpoint returns **404** for them — which
  reads as a bad model name rather than a bad location. The reverse also holds:
  interpolating `global` into a regional host yields
  `global-aiplatform.googleapis.com`, which does not resolve.
- **Never default a region.** A guessed region produces a URL that resolves and
  is wrong — an agent card advertising a reachable host in the wrong region looks
  like a working deploy. Resolve from config, and when nothing is set, decline to
  build the URL and say so:

```python
# Engine region first, deploy region second, no fallback.
location = os.getenv("AGENT_ENGINE_LOCATION") or os.getenv("CLOUD_LOCATION")
if engine_id and project and location:
    return f"https://{location}-aiplatform.googleapis.com/..."
logger.warning("No region configured; the card cannot advertise a reachable URL")
```

A hardcoded region also drifts silently: it stays correct until something is
deployed elsewhere, and nothing rereads the comment explaining it.

## 9. Service agents are created lazily

A service agent does not exist until the service is first used, so a binding
made before that is refused:

```
INVALID_ARGUMENT: Service account
service-<PROJECT_NUMBER>@gcp-sa-aiplatform-re.iam.gserviceaccount.com
does not exist.
```

On a new project this hits every grant to a service agent, and setup scripts
routinely hide it -- `add-iam-policy-binding ... >/dev/null 2>&1 || true`
reports success whatever happened, so the missing roles surface later as 403s
during deployment with nothing tying them back. Never mask the result of a
grant.

Note which agent you need. `gcp-sa-aiplatform` and `gcp-sa-aiplatform-re` are
different principals, and the runtime one is the `-re` form.

### `-re` is minted by a sourceless create, not by enabling the API

**A create with no source at all mints both service agents in ~20 seconds**,
which is what makes a single-pass provisioning run possible:

```jsonc
// POST .../v1beta1/projects/P/locations/R/reasoningEngines
{"displayName": "bootstrap", "spec": {"identityType": "AGENT_IDENTITY"}}
```

Measured on two fresh projects, with the middle row as the control:

| State | `-re` present? |
| :--- | :--- |
| Fresh project, nothing enabled | No — `NOT_FOUND` |
| `aiplatform.googleapis.com` enabled, billing linked, no engine | **No** — a grant fails `INVALID_ARGUMENT: … does not exist` |
| After the sourceless create | **Yes** — the grant succeeds |

**Enabling the API is not what mints it.** The create is. Both agents appear
together, each already holding its default role:

```
roles/aiplatform.reasoningEngineServiceAgent -> service-<NUM>@gcp-sa-aiplatform-re…
roles/aiplatform.serviceAgent               -> service-<NUM>@gcp-sa-aiplatform…
```

The create requires **billing enabled** — without it the API returns
`403 BILLING_DISABLED`, so a provisioning script must check billing before this
step rather than after.

So anything that binds `-re` can run in one pass: mint, then grant. There is no
need to deploy running code first, and no need for a two-stage apply that
converges on a later run.

#### Two routes that do not work

- **`gcloud beta services identity create --service=aiplatform.googleapis.com`**
  mints `gcp-sa-aiplatform`, not the `-re` form. No service name mints `-re`:
  `aiplatform` is the only one that resolves, and both
  `reasoningengine.googleapis.com` and `aiplatform-re.googleapis.com` fail with
  `SERVICE_CONFIG_NOT_FOUND_OR_PERMISSION_DENIED`.
- **`gcloud workload-identity service-agents generate`** exists at GA and looks
  like the intended tool. It needs `workloadidentity.googleapis.com` enabled,
  and then **403s for a project Owner** on
  `workloadidentity.serviceAgents.create`. Owner is the strongest role most
  people hold on their own project, so treat this as unavailable.

### An Agent Identity is not the `-re` service agent

These are two different principals. One sourceless create mints both, but they
are not interchangeable and a grant to one does nothing for the other:

| | `gcp-sa-aiplatform-re` | Agent Identity |
| :--- | :--- | :--- |
| Form | `service-<NUM>@gcp-sa-aiplatform-re.iam.gserviceaccount.com` | `principal://agents.global.org-<ORG>...` |
| What it is | Google-managed service agent for the project | Per-engine SPIFFE identity |
| Scope | One per project | One per engine |
| Created by | A sourceless create, or any deploy | The same sourceless create, with `identityType: AGENT_IDENTITY` |
| Pre-creatable | **Yes — without any source code** | **Yes — without any source code** |

Which one you need depends on the identity mode. An engine running as a
**custom service account** needs `-re` to hold
`roles/iam.serviceAccountTokenCreator` on that account, so it can be
impersonated. An engine running under **Agent Identity** has no service account
at all, so nothing needs impersonating and `-re` stays off its permission
path entirely.

**An engine can be created with no source at all**, purely to mint its
identity:

```jsonc
// POST .../v1beta1/projects/P/locations/R/reasoningEngines
{"displayName": "my-agent", "spec": {"identityType": "AGENT_IDENTITY"}}
```

This returns a real engine in **~20 seconds** (a full deploy takes minutes) and
`spec.effectiveIdentity` is populated immediately, so IAM can be granted before
any code exists. Source is deployed to the same engine later. Two details make
or break it:

- **It requires `v1beta1`.** `agents-cli` switches API version precisely for
  this; on `v1` the sourceless create does not behave.
- **`identityType` must be set.** A create with only `displayName` and no spec
  is accepted but does not give you an identity.

`agents-cli`'s `setup_agent_identity` (`deploy/agent_runtime.py`) is this
pattern, and it then grants six roles to the new principal —
`aiplatform.user`, `serviceusage.serviceUsageConsumer`, `browser`,
`cloudapiregistry.viewer`, `logging.logWriter`, `monitoring.metricWriter` —
because **a fresh Agent Identity holds no roles at all**. The first failure is
otherwise `serviceusage.serviceUsageConsumer`, which breaks *every* Google API
call rather than the one you were testing.

The same trick is how a **Cloud Run**-hosted agent gets an agent identity: mint
it with a sourceless Agent Runtime create, grant it, and let the Cloud Run
workload assert it. The engine exists to issue the identity, not to serve.

A sourceless engine has **no `deploymentSpec`** — no image, no instances,
nothing kept warm — so minting an identity ahead of time does not leave a
running resource behind. That matters because anything with a real deployment
spec defaults to `min_instances: 1`.

Terraform reaches the same end differently, since its provider needs a spec: it
creates the engine with a **placeholder source archive** — a tarball containing
only a `Dockerfile` — and then `ignore_changes` on
`spec[0].source_code_spec`, `container_spec` and `deployment_spec` so a later
CI deploy of the real code is never reverted. The placeholder must use
`source_code_spec`, not `container_spec`: the real deploy writes the former, and
a `container_spec` placeholder is left beside it and rejected on update.

**Beware `effectiveIdentity`'s form.** It reports the **`org-`** trust domain,
while the principal the API actually authenticates is the **`proj-`** form. A
binding written from the field verbatim is accepted and grants nothing.

## Troubleshooting index

| Symptom | Cause |
| :--- | :--- |
| A2UI renders as raw JSON | Registered as ADK; only `a2aAgentDefinition` negotiates A2UI |
| A2UI reply is **blank** — no text, no error | The `X-A2A-Extensions` echo never reached the client, so the surface was discarded. On Agent Runtime the passthrough strips it and no code fixes it; host on Cloud Run |
| 403 on everything right after switching a service to `agent-identity` | The new principal holds no roles. Cloud Run agent identities get **no defaults**; re-grant what the service account had |
| A response header the app sets never arrives | Agent Runtime's `/api/` passthrough replaces response headers wholesale. Confirm by setting a throwaway header in middleware — if that vanishes too, it is the proxy, not your code |
| `CREDENTIALS_MISSING` from GE | A2A on Agent Runtime without an Authorization carrying `cloud-platform` scope. Set `authorizationConfig.agentAuthorization` on the agent |
| `PERMISSION_DENIED` on `reasoningEngines.query` | The Discovery Engine service agent of the **GE app's** project lacks query on the engine |
| 401 between agents with correct IAM | Bound credential sent as a bearer token. Fix the transport, not the IAM |
| 401 "invalid authentication credentials" from any `*.googleapis.com` | Bound credential sent over plain HTTP. Use the generated client library |
| 401 from Cloud Run despite `run.invoker` on the agent | Agent Identity cannot mint its own audience-bound ID token. Delegate SA (Architecture F) |
| Metadata server returns 200 but the call still 401s | The 200 is not a usable ID token under Agent Identity |
| `VERSION_NOT_SUPPORTED` on an A2A call | Missing `A2A-Version: 1.0` header; a missing version is read as `0.3` |
| `Identity Pool does not exist` | Wrong Agent Identity trust domain for the project |
| Grant applied, behaviour unchanged | Inert binding — that principal form is not what authenticates |
| Confident answer containing invented data | A sub-agent failed to resolve; `state: completed` is not evidence |
| `Cannot connect to ... mtls.googleapis.com` / `Name or service not known` | Engine on a PSC network whose private zone answers for `mtls.googleapis.com` but holds no records — gateway attached without networking, **or networking attached without a gateway**. Clear `pscInterfaceConfig` if the gateway is not being used |
| Turn fails before any sub-agent is called | Look above the sub-agent lines: the runtime's own session call failed first, typically the mtls case above |
| Card URL returns 404 | Path carries the ADK `App` name |
| 404 for a model that exists | It is served only from the `global` endpoint; a regional one 404s (§8) |
| `global-aiplatform.googleapis.com` does not resolve | The model endpoint value was interpolated into a regional host (§8) |
| Agent reachable but sessions or memories are empty | Card or client built against the wrong region — resolves fine, wrong place (§8) |
| Deploy 403s on a compute permission | Grant the named permission to `service-<PROJECT_NUMBER>@gcp-sa-aiplatform` and allow propagation |
| `Service account service-…@gcp-sa-… does not exist` when granting | The service agent is absent. `services identity create` mints `gcp-sa-aiplatform`; only a sourceless create mints `-re` (§9) |
| `does not exist` when granting to `gcp-sa-aiplatform-re` specifically | Enabling the API does not mint `-re`. A **sourceless create** does, in ~20s (§9) |
| IAM script reports success, deploys still 403 | Bindings masked with `\|\| true`. Re-run without the mask and read what failed (§9) |

## Sources

- [Agent Identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity)
- [Use an A2A agent](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/use-an-a2a-agent)
- [Register A2A agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent)
- [Register ADK agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent)
- [Agent Gateway overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview)
- [Route traffic through Agent Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-gateway-runtime-deploy)
