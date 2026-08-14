---
name: agent-platform-a2a
description: >
  This skill should be used when debugging a deployment on Gemini Enterprise
  Agent Platform: 401 or 403 between agents, "permission denied" on a reasoning
  engine, "CREDENTIALS_MISSING", "Identity Pool does not exist", an IAM grant
  that changes nothing, an agent answering confidently with invented data, or a
  deploy failing on a permission the docs never mention. Covers how to identify
  the real caller, why 401 and 403 need opposite responses, the Agent Identity
  principal forms, and the traps that make a broken deployment look healthy.
  Use agent-platform-architecture instead for choosing a design.
metadata:
  author: Alan Blythe
  license: Apache-2.0
  version: 0.2.0
  requires:
    bins:
      - gcloud
---

# Debugging agent auth on Agent Platform

Almost every hour lost here goes to one of four mistakes: treating a 401 as a
permissions problem, inferring the caller instead of measuring it, guessing at
permissions instead of reading them out of the error, or believing a response
that says it worked.

## 1. A 401 is not a 403

| Code | Meaning | Response |
| :--- | :--- | :--- |
| **403** `PERMISSION_DENIED` | Credential accepted, principal lacks the role | Grant something |
| **401** `UNAUTHENTICATED` | Credential rejected. No authorization decision happened | **Never grant anything.** Wrong credential type, missing scope, or a bound credential sent as a plain bearer token |

A 401 leaves **no audit entry** — `protoPayload.status.code=7` returns nothing,
because nothing was denied. An empty denial query next to a failing call is
itself the signal that you are working the wrong layer.

The most expensive version of this: under Agent Identity, sub-agent A2A calls
fail 401 because the credential is mTLS/DPoP-bound and `RemoteA2aAgent` sends a
bearer header. No combination of roles will ever fix it.

## 2. Measure the caller, never infer it

`spec.effectiveIdentity` and the resource fields will mislead you. Read the
principal the API actually authenticated.

```bash
# Enable Data Access logging (off by default; bills by volume, turn it back off)
gcloud projects get-iam-policy PROJECT --format=json > /tmp/p.json
jq '.auditConfigs = [{"service":"aiplatform.googleapis.com",
      "auditLogConfigs":[{"logType":"ADMIN_READ"},{"logType":"DATA_READ"},{"logType":"DATA_WRITE"}]}]' \
  /tmp/p.json > /tmp/p2.json
gcloud projects set-iam-policy PROJECT /tmp/p2.json     # preserves existing bindings

# Reproduce, then read who called
gcloud logging read 'protoPayload.serviceName="aiplatform.googleapis.com"' \
  --project PROJECT --limit 5 --freshness=10m \
  --format="value(protoPayload.authenticationInfo.principalSubject,
                  protoPayload.methodName, protoPayload.status.code)"
```

`principalSubject` is ground truth. Everything else is a guess.

### Agent Identity principal forms

Three exist, and granting the wrong one fails in two distinguishable ways:

| Form | Notes |
| :--- | :--- |
| `agents.global.org-<ORG_ID>...` | documented for org'd projects; what `effectiveIdentity` reports |
| `agents.global.proj-<PROJECT_NUMBER>...` | observed as the *authenticating* principal in some projects |
| `agents.global.project-<PROJECT_NUMBER>...` | documented for orgless projects |

```
principal://<TRUST_DOMAIN>/resources/aiplatform/projects/<N>/locations/<REGION>/reasoningEngines/<ID>
```

- Wrong-but-existing trust domain ⇒ the binding is accepted and **inert**. Nothing authenticates as it; behaviour is unchanged.
- Trust domain with no pool ⇒ `INVALID_ARGUMENT: Identity Pool does not exist`.

Neither is a permissions problem. Measure the form (§2) rather than copying one
from a doc — including this one.

An Agent Identity workload has **no service account**, so a `serviceAccount:`
member grants it nothing.

## 3. Read permissions out of the error; never guess

Deploys onto a network attachment fail one permission at a time, each only after
the container builds. The 403 body **names the permission verbatim**:

```
Required 'compute.regionOperations.get' permission for 'projects/.../operations/...'
```

and some errors name the role:

```
Please make sure the Vertex AI service account has the dns.peer role
```

So: deploy → read the name → grant exactly that → wait for propagation → repeat.
Adding permissions you *think* come next grants access nothing needs, and you
never learn which were required.

Two things that make this loop confusing:

- **Consecutive failures look identical.** They name the same service account
  and differ only in the method — `NetworkAttachmentsService.Get` then `.Patch`
  then `RegionOperationsService.Get`. The second reads like the first grant not
  having worked.
- **IAM propagation takes minutes.** An immediate retry fails identically and
  looks like the wrong role. Wait ~3 minutes before concluding anything.

Observed set for deploying an Agent Runtime agent onto a customer-owned network
attachment, granted to `service-<N>@gcp-sa-aiplatform` (**not** the `-re` service
agent, which is the one everything else uses):

```
compute.networkAttachments.get       # NetworkAttachmentsService.Get
compute.networkAttachments.update    # .Patch — accept-lists its own PSC connection
compute.regionOperations.get         # polls the update it started
roles/dns.peer                       # creating the DNS peering zone
```

`roles/compute.networkUser` is *not* sufficient: it carries `.get` but not
`.update`.

## 4. Do not trust a successful-looking response

**A failed sub-agent does not fail the turn.** When `RemoteA2aAgent` cannot
resolve a sub-agent, the model answers from nothing and the task reports
`state: completed` with fluent, entirely invented content. Two 401s produce a
confident, plausible answer. This is the single most dangerous behaviour here —
it will convince you a broken deployment works.

Verify from logs:

```bash
gcloud logging read 'resource.labels.reasoning_engine_id="ID" AND
  (textPayload:"agent-card.json" OR textPayload:"Failed to resolve")' \
  --project PROJECT --limit 6 --freshness=10m --format="value(textPayload)"
```

Both card fetches should be 200.

Related traps:

- **Engine `state` lags.** It reports `ACTIVE` while an update is in flight, so a
  state-based wait reads a stale config and reports success. Poll the returned
  **operation** instead. A second PATCH during the window is rejected with "not
  in ACTIVE state", so a naive retry loop fights itself.
- **PATCH is async and slow** — it redeploys the container. A non-error response
  means accepted, not applied.
- **Sub-agent URL env vars are derived from the agent name** (`{AGENT_NAME}_URL`,
  e.g. `SCHEDULING_AGENT_URL`). A name that does not match is **ignored
  silently** and the agent falls back to localhost. A deployed orchestrator will
  quietly call `localhost` forever. Confirm from the startup log line.
- **Card paths carry the ADK `App` name**, not `app`: an app named
  `luncher_agent` serves `/a2a/luncher_agent/.well-known/agent-card.json`.
  Scripts that hardcode `/a2a/app/` 404, which looks like a broken deployment.

## 5. Outbound credentials in agent code

```python
# Ask for the scope explicitly. A service account carries cloud-platform
# implicitly; an Agent Identity credential does not, and unscoped ADC 401s.
credentials, _ = google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])

# Let the credential apply itself. Reading .token bypasses whatever binding the
# credential implements, and hands expiry back to the library.
creds.before_request(Request(), "POST", url, headers)     # not: refresh(); creds.token
```

Both are correct regardless of identity mode. **Neither rescues A2A under Agent
Identity** — that needs Agent Gateway. Cloud Run targets need an audience-bound
ID token, which ADC does not supply, so that stays a separate path.

A pinned token is its own bug: tokens last ~1h, a warm instance holds its clients
far longer, so anything fetched once at import starts 401ing after an hour.

## 6. Running Terraform with the right credential

Go's auth library **ignores `CLOUDSDK_CONFIG`**. On a machine with several gcloud
profiles, Terraform silently uses `~/.config/gcloud/application_default_credentials.json`
— the wrong account — and fails with a 403 that reads like a missing role.

```bash
export GOOGLE_OAUTH_ACCESS_TOKEN="$(gcloud auth print-access-token)"
```

Pointing `GOOGLE_APPLICATION_CREDENTIALS` at the profile's ADC file is the
obvious fix and does not work for Workspace accounts: the Go client rejects the
stored credential with `invalid_rapt` and demands an interactive reauth a script
cannot perform. Python's `google-auth` *does* respect `CLOUDSDK_CONFIG`, so a
Python check reports the right identity while Terraform uses a different one —
which makes this trap very hard to see.

## Troubleshooting index

| Symptom | Cause |
| :--- | :--- |
| A2UI renders as raw JSON | Registered as ADK. Only `a2aAgentDefinition` negotiates A2UI |
| `CREDENTIALS_MISSING` from GE | A2A on Agent Runtime without an Authorization (`cloud-platform` scope) |
| `PERMISSION_DENIED` on `reasoningEngines.query` | discoveryengine SA of the **GE app's** project lacks query on the engine |
| 401 between agents, IAM correct | Bound credential as a bearer token. Not fixable in the caller — Agent Gateway, or no Agent Identity |
| `Identity Pool does not exist` | Wrong Agent Identity trust domain for this project |
| Grant applied, nothing changed | Inert binding — that principal form is not what authenticates. Read `principalSubject` |
| Confident answer, invented data | A sub-agent failed to resolve. `state: completed` proves nothing |
| `Cannot connect to ... mtls.googleapis.com` | Gateway attached without PSC networking |
| Card URL 404s | Path carries the ADK `App` name, not `app` |
| Deploy 403s on a compute permission | Grant the named permission to `service-<N>@gcp-sa-aiplatform`, wait 3 min |

## Sources

- [Agent Identity](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-identity)
- [Use an A2A agent](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/use-an-a2a-agent)
- [Register A2A agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-a2a-agent)
- [Register ADK agents with GE](https://docs.cloud.google.com/gemini/enterprise/docs/register-and-manage-an-adk-agent)
- [Agent Gateway overview](https://docs.cloud.google.com/gemini-enterprise-agent-platform/govern/gateways/agent-gateway-overview)
- [Route traffic through Agent Gateway](https://docs.cloud.google.com/gemini-enterprise-agent-platform/scale/runtime/agent-gateway-runtime-deploy)
