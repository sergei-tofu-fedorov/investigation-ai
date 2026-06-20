---
name: investigation-gcp-logs
description: >-
  GCP Cloud Observability reference for investigations. SEARCH LOGS by platform (iOS/Android/web), user,
  email, account, endpoint, path, status, error, app version, device, or trace id; query METRICS (time
  series), ALERTS (policies + firing), TRACES (search/get, latency correlation), ERROR REPORTING (group
  stats). Access is the read-only `mcp__gcp_logs_*` observability tools (there is no gcloud CLI here). Exact
  LQL field paths for the Invoices BFF request log, the Subz subscription event stream, and the LB log.
  Invoke BEFORE any query.
---

# GCP Cloud Observability (Tofu / Invoices)

Access is the read-only **`mcp__gcp_logs_<env>__*`** MCP tools. **There is no `gcloud` CLI here** — don't call it;
it has no credentials and will fail. `<env>` is the project label: **`gcp_logs_stage`** (`invoicesapp-project-test`,
exploratory default) · **`gcp_logs_prod`** (`inv-project`, real traffic). The project is implied by the tool — you
do **not** pass a `--project` flag.

| Group | Tools |
|---|---|
| logs | `list_log_entries`, `list_log_names`, `list_buckets`, `list_views`, `list_sinks`, `list_log_scopes` |
| metrics | `list_time_series`, `list_metric_descriptors` |
| alerts | `list_alert_policies`, `list_alerts` |
| traces | `list_traces`, `get_trace` |
| errors | `list_group_stats` |

| I need to… | Section |
|---|---|
| app/request logs; search by user/account/platform/endpoint/error | **`logs`** — start here |
| numeric trend over time (latency, error rate, CPU, count) | **`metrics`** |
| what alerts exist / what is firing now | **`alerts`** |
| follow one request across services; edge-vs-app latency | **`traces`** |
| top/most-frequent exceptions (grouped) | **`errors`** |

## When (and when not) to query — read this first

- **Query only when the ask needs log/metric/trace data.** Don't open with a probe, a "connectivity check", or a
  `list_log_names`/`list_buckets` "is it reachable?" call — the first tool you call should already be the real query.
  The tools are wired at boot; assume they work and use them.
- **Never narrate tool or connectivity status to the user.** No "MCP connected", "switching to MCP", "verifying
  access". Surface the finding, not the plumbing.
- **A failed call is a finding-limitation, not a setup problem.** `PERMISSION_DENIED` (the project SA lacks that role)
  or empty results → note it briefly and move on; don't retry in a loop or explain the auth model.

`list_log_entries` takes an **LQL `filter` string** (syntax in the tables below). **Bound every call**: a
`timestamp>="<ISO>"` clause + a small page size (≤2000 for aggregations). The `entries.list` quota is 60
req/min/project — ask broad, aggregate locally, note when a page cap was hit (counts are then partial).

---

# `logs`

**Main source for most investigations — the BFF request log:**
`logName="projects/<proj>/logs/Invoices.Api.Middleware.RequestLoggingMiddleware"` (container `invoices-api`).
One entry per HTTP request with the fields below.

> ⚠️ **Bodies are heavy — don't pull them unless the query needs them.** `ResponseBodyText`, `RequestBodyText`
> (≤10KB, passwords masked) and the cursor-bearing `RequestPathAndQuery`/`RequestQuery` can be large. Default to
> projecting only the light fields / a small page; pull bodies only when inspecting a specific case (e.g. the
> 200-error envelope or `set_identifiers`).

## Search by …

Fields are under `jsonPayload.properties.` (omitted in the table). Ops: `=` exact · `:` substring · `=~` regex.
Prefer the resolved PascalCase fields (stable casing) over the raw lowercase `xa-*` headers.

| Find by | Field | Notes / values |
|---|---|---|
| **account** | `AccountId` | **primary account selector** — the client `Account-Id` header, so present on most requests (`unknown` only if the client omits it). Sentry stores only the pre-`-` prefix → `=~"^<prefix>"` |
| **user** | `UserEmail` · `MasterUserId` | auth-resolved — **may be absent** on pre-auth/framework errors (see Gotchas) |
| **product / app** | `ProductKey` (raw `"xa-app-type"`) | `invoices` iOS · `invoices-android` · `*.web`⇒web · `tofu-fieldservice` · `tofu-fieldservice-worker` · `expenses` · `taxes` · `mileage` · `payments`; invalid⇒`unknown` |
| **platform** | `"xa-ostype"` | `ios`/`android`/`web`; **absent ⇒ iOS** |
| **app version** | `AppVersion` (raw `"xa-app-version"`) | `3.9.46` |
| **device** | `"xa-vendor-id"` | stable per-device id — correlate one device's requests |
| **endpoint** | `EndpointName` | `…Controller.Action` |
| **path** | `RequestPath` · `RequestPathAndQuery` | `:` substring, `=~` regex |
| **method / status** | `RequestMethod` · `StatusCode` | ⚠️ 200-error envelope (Gotchas) |
| **errors** | `severity>=ERROR` *or* `ResponseBodyText:"error"` | severity works without auth-gated fields |
| **API version** | `ApiVersion` | `1.0`/`2.0`/`3.0` (per-action) |
| **timing** | `Elapsed` (ms) | compare to LB `httpRequest.latency` (§ traces) |
| **correlation** | `TraceId` · `RequestId` | `TraceId` = the `<id>` half of the LB `trace`; `RequestId` e.g. `0HN…:0000035B` |
| **time** | `timestamp>="2026-06-16T00:00:00Z"` | always add one |

> Also present, rarely worth filtering on: `xa-device-model` (secondary); `xa-os-version`, `xa-store`,
> `xa-app-build-number`, `xa-timezone`; `SpanId`/`ParentId`/`ConnectionId`.

## Targets per service (container → repo)

`resource.type="k8s_container"`; cluster as noted. Unmapped on the cluster: `invoices-webroot`, `payments-api`,
`expenses-worker`, the `subs-*` subscriptions platform, the `tofu-web-*` frontends.

| Service | Selectors | Repo |
|---|---|---|
| **BFF** (start here) | `tofu-cluster` / `invoices-api`; request stream = the `logName` above | **Invoices.Backend** |
| BFF jobs | `invoices-worker` | Invoices.Backend (Worker) |
| **Auth** | `tofu-cluster` / `auth-api` | **Tofu.Auth.Backend** |
| **Invoices core** | `tofu-cluster` / `tofu-invoices-api` | **Tofu.Invoices.Backend** |
| Tofu.AI sidecar | `invoices-cluster` / `tofu-ai-api`; +`-…ResponseBodyText="Healthy" -…RequestPath="/callback/sendgrid/status_update"` | Tofu.AI.Backend |
| tofu-support (this agent) | `invoices-cluster` / `tofu-support-api`; fields are **top-level `jsonPayload.*`** (not `.properties.*`) | Tofu.Support.Backend |

## Useful endpoints for investigation

Specific BFF requests whose log entry reveals a user's state. Find by `EndpointName`/`RequestPath`, scoped to the
account. _(Seed list — extend as more are confirmed.)_

| To learn | Method · path | `EndpointName` | Reveals |
|---|---|---|---|
| current subscription | `GET /api/plans/current` | `PlansController.Current` | the account's current plan / subscription state (`ResponseBodyText`) |
| account's firebase id / push token | `PUT /api/account/set_identifiers` | `AccountController.PutIdentifiers` (V1) | **primary source for `firebaseId` + `pushToken`** (body also: `userId`, `idfa`, `appsflyerId`, `publicUserId`; for Subz/subscription correlation use the Subz event stream — § below) |

## External identifiers (cross-system correlation)

External ids let you pivot between an account and AppsFlyer / Firebase / push / ads / subscriptions. **Best source per id:**

| Identifier | Go to |
|---|---|
| `firebaseId`, `pushToken` | **BFF `set_identifiers`** (primary; notification/Firebase ids) |
| `publicUserId`/`PublicId`, `appsflyerId`, `idfa`, subscription (`ProductId`, `OriginalTransactionId`), `email`, `OsType` | **Subz event stream** (richest; carries them all) |

| Source | Selectors | Carries |
|---|---|---|
| BFF `set_identifiers` | `…RequestLoggingMiddleware`, `RequestPath="/api/account/set_identifiers"`, **filter a body field** e.g. `RequestBodyText:"firebase"` | body `AccountIdentifiersDto`: `userId`, `idfa`, `appsflyerId`, `firebaseId`, `pushToken`, `publicUserId` (clients send a subset). NB a pre-execution row shows an empty body — match on a body field to hit the populated one |
| **Subz subscription event stream** | container `subs-event-stream-worker` (ns `subs`, `logName=…/Default`), `jsonPayload.message:"<id>"` | `EnrichedSubscription*Event`: `AccountDetails{ AccountId, PublicId, AdapterAccountId, Idfa, AppsflyerId, FirebaseId, OsType, Email }` + `Details{ ProductId, OriginalTransactionId, Idfa, AppsflyerId, FirebaseId }` + `ProductKey`, `EventTime` |
| subs platform workers | container `subs-ios-worker` / `subs-android-worker` | same ids in subscription-processing logs |

> **Bridging Subz ↔ invoices** — they use **different `AccountId` spaces; do NOT join on `AccountId`.** Subz keys on
> our **`publicUserId`** (= Subz `AccountDetails.PublicId`). To resolve a Subz event to an invoices account, map that
> `publicUserId`/`PublicId` to a BFF `AccountId` via the Mongo `accounts` doc (`Identifiers`), or via a
> `set_identifiers` row that carried it (`RequestBodyText:"<PublicId>"`). (Verified in prod.)

## tofu-support events (filter `jsonPayload.event`)

`agent_turn` (per-turn + cumulative `sessionCostUsd`/tokens, `sessionKey`, `durationMs`, `isError`; latest line per
`sessionKey` = total) · `agent_stall` (first-event watchdog fired) · `agent_soft_timeout`/`agent_turn_extended` ·
`agent_resume`/`pool_session_new`/`pool_session_evict`/`pool_resume_failed` (per-thread lifecycle) ·
`slack_request`/`slack_reply`/`slack_feedback`/`slack_skip` · `gcp_log_query` (tool+filter audit) · `kb_write` ·
`git_cmd`/`repo_op_*`.

## Filter recipes

LQL `filter` strings — pass to `list_log_entries` with a `timestamp>=` bound + a page cap.

```
# Errors on a path, 24h (mind the 200-error envelope)
jsonPayload.properties.RequestPath:"/api/tap2pay" AND (jsonPayload.properties.StatusCode>=500 OR jsonPayload.properties.ResponseBodyText:"error")

# One account's recent activity · one platform's errors
jsonPayload.properties.AccountId="<id>"
jsonPayload.properties."xa-ostype"="android" severity>=ERROR

# Container-wide errors when auth-gated fields are missing
resource.labels.container_name="invoices-api" severity>=ERROR
```

For "who/which is affected" breakdowns: pull a capped page (≤2000) projecting the field you're grouping by
(`AccountId`, `EndpointName`, `xa-app-version`, …) and **aggregate locally** — count distinct values, keeping rows
where the field was missing (dropping them skews percentages). Note the cap so partial counts are honest.

---

# `metrics`

Numeric trends, not events. `list_time_series` (data), `list_metric_descriptors` (discover metric types). For
"is latency/error-rate/CPU/count rising?". Read the tool schema for metric-type/aggregation/interval args.

# `alerts`

`list_alert_policies` (configured alerts + conditions), `list_alerts` (incidents **firing now**). Policy-violation
*events* also land in Cloud Logging as `monitoring.googleapis.com/ViolationOpenEventv1` entries (they name the
policy + condition) — useful for history.

# `errors`

Error Reporting — exceptions grouped + ranked by frequency. `list_group_stats`. For "top errors / new or recurring /
how often" — faster than scanning raw `severity>=ERROR` when you want the shape, not individual lines.

# `traces`

`list_traces` (find slow/error traces), `get_trace` (full span tree of one trace).

**The `trace` field** — the GCP load balancer **mints it per request** and stamps it on the LB log and every backend
log line for that request, so it is the **one backend-only key that stitches a request across all microservices**
(BFF, `tofu-invoices-api`, `auth-api`, …). It is **not** a client/user/session id.
- Format: `projects/<proj>/traces/<TRACE_ID>` (e.g. `projects/inv-project/traces/2c576600af8b05ae0a60e339c271f59a`).
  The BFF log's `TraceId` property = the `<TRACE_ID>` half.
- Filter `trace="projects/<proj>/traces/<id>"`; **drop container/`logName` selectors** to fan it across all services.

**LB log** (`resource.type="http_load_balancer"`): `httpRequest.latency` (`>"2s"`), `.status`, `.requestUrl` (`:`),
`.userAgent`, `trace`. `Elapsed ≈ latency` ⇒ slow in-app; `latency >> Elapsed` ⇒ LB/network.

---

# Gotchas

- **200-error envelope.** BFF often returns HTTP **200 with an error JSON body** — search `ResponseBodyText:"error"`, not only `StatusCode`. Empty for PDFs/streams.
- **Identity fields.** `jsonPayload.properties.AccountId` comes from the client `Account-Id` header — usually present (`unknown` if omitted), so it's the **primary account filter**. `MasterUserId`/`UserEmail` are auth-resolved and absent on pre-auth/framework errors (fall back to container-wide `severity>=ERROR` + tight window); `ProductKey` derives from `xa-app-type`.
- **Header casing.** Raw `xa-*` keys keep the client's casing (iOS sends **lowercase**, e.g. `"xa-ostype"`). Prefer the resolved PascalCase fields (`ProductKey`/`AppVersion`), or use `=~` to be case-tolerant.
- **Platform inference.** `xa-ostype` absent ⇒ iOS. Any `xa-app-type`/`ProductKey` containing `.web` ⇒ web.
- **Test-project mix-ups.** `dev-gateway-api`/`dev2-gateway-api` are gateway proxies, not the BFF; `tofu-invoices-api`/`auth-api` co-locate in the same project — filter by container.
