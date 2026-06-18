---
name: investigation-gcp-logs
description: GCP Cloud Observability reference for automated investigations — exact LQL field paths for Invoices.Backend request logs, query/aggregation recipes, LB-trace correlation, project rules. Invoke BEFORE querying GCP logs/metrics/traces.
---

# GCP observability for investigations (Invoices platform)

Access is via the **GCP Cloud Observability MCP tools**, exposed per environment as `mcp__gcp_logs_<env>__*`
(read-only). There is **no `gcloud` CLI in this environment** — do not call `gcloud`; it has no credentials here
and will fail. The tools cover:

| Group | Tools |
|---|---|
| logs | `list_log_entries`, `list_log_names`, `list_buckets`, `list_views`, `list_sinks`, `list_log_scopes` |
| metrics | `list_time_series`, `list_metric_descriptors` |
| alerts | `list_alert_policies`, `list_alerts` |
| traces | `list_traces`, `get_trace` |
| errors | `list_group_stats` |

`<env>` is the project label: **`gcp_logs_prod`** (`inv-project`, real traffic) and **`gcp_logs_stage`**
(`invoicesapp-project-test`, exploratory default). Pick the prefix for the environment you're investigating; the
project is implied by the tool — you do **not** pass a `--project` flag.

## When (and when not) to query — read this first

- **Query only when the ask actually needs log/metric/trace data.** Don't open with a probe, a "connectivity check",
  or a `list_log_names`/`list_buckets` "is it reachable?" call — the first GCP tool you call should already be the
  real query for the question. The tools are wired up at boot; assume they work and just use them.
- **Never narrate tool or connectivity status to the user.** No "GCP MCP is connected", no "switching to MCP", no
  "verifying access". The user wants the finding, not the plumbing. Surface only investigation results.
- **A failed call is a finding-limitation, not a setup problem.** If a tool returns `PERMISSION_DENIED` (the project
  SA lacks that role) or empty results, note it briefly as a limitation in your report and move on — do not retry in
  a loop, switch tools to "test the connection", or explain the auth model to the user.

## Querying logs

`list_log_entries` takes an **LQL filter string** — the same filter syntax the field tables below describe. Always:

- **Bound the window** with a `timestamp >= "<ISO>"` clause (e.g. last 24h) — unbounded scans are slow and waste quota.
- **Bound the result count** with a small page size (default ~50; aggregations ≤2000). Note in findings when a cap was
  hit — counts are then partial.
- The underlying `entries.list` quota is 60 req/min/project — ask fewer, broader questions and aggregate locally
  rather than paging repeatedly.

## App request log (the BFF — most investigations start here)

Selectors: `resource.type="k8s_container"`, cluster `tofu-cluster`, container **`invoices-api`** (BFF API, same name in test+prod), `invoices-worker` (background jobs), `invoices-webroot` (static). Request log stream: `logName="projects/<proj>/logs/Invoices.Api.Middleware.RequestLoggingMiddleware"`.

All per-request fields live under **`jsonPayload.properties.*`** (NOT `jsonPayload.*`). Quote hyphenated keys: `jsonPayload.properties."XA-App-Type"`.

| Field | Notes |
|---|---|
| `RequestPath` / `RequestPathAndQuery` | exact `=`, substring `:`, regex `=~` |
| `RequestMethod`, `StatusCode`, `Elapsed` (ms) | |
| `ResponseBodyText` | ⚠️ the BFF often returns HTTP **200 with an error JSON body** — search here, not only StatusCode. Empty for PDFs/streams |
| `RequestBodyText` | truncated 10KB, passwords masked |
| `AccountId`, `MasterUserId`, `UserEmail`, `ProductKey` | ⚠️ attached only after auth succeeds — early/framework errors won't carry them; fall back to container-wide `severity>=ERROR` + a tight time window |
| `EndpointName` | controller.action display name |
| `"XA-App-Type"` | `invoices` (iOS), `invoices-android`, `invoices.web`, `tofu-fieldservice`, `tofu-fieldservice-worker`, `demo-invoices`, … — header-derived keys keep whatever casing the client sent |
| `"XA-App-Version"`, `"XA-OS-Version"`, `"XA-Device-Model"`, `"XA-OsType"` | client context (`XA-OsType` absent ⇒ iOS) |
| `ApiVersion` | 1.0 / 2.0 / 3.0 (per-action versioning) |

⚠️ `jsonPayload.properties.RequestId` does **not** exist. Sentry's `accountId` tag stores only the prefix before the first `-`; correlate with `AccountId=~"^<prefix>"`.

## Tofu.AI service logs (this service itself)

Different cluster: `resource.labels.cluster_name="invoices-cluster"`, container `tofu-ai-api`. Add noise filters:
`-jsonPayload.properties.ResponseBodyText="Healthy" -jsonPayload.properties.RequestPath="/callback/sendgrid/status_update"`.

## Load balancer log + trace correlation

`resource.type="http_load_balancer"`: `httpRequest.latency` (e.g. `>"2s"`), `httpRequest.status`, `httpRequest.requestUrl` (substring `:`), `httpRequest.userAgent`, `trace`.

`trace` is the **join key** between LB and app logs: `trace="projects/<proj>/traces/<TRACE_ID>"`. Interpretation: `Elapsed ≈ latency` ⇒ slowness is in-app; `latency >> Elapsed` ⇒ LB/network overhead. For the full span tree of a trace, use `get_trace`; to find slow/error traces, `list_traces`.

## Filter recipes

These are LQL `filter` strings — pass them to `list_log_entries` (with a `timestamp>=` bound and a page-size cap).

```
# Errors on a path, last 24h (remember the 200-error-envelope gotcha)
jsonPayload.properties.RequestPath:"/api/tap2pay" AND (jsonPayload.properties.StatusCode>=500 OR jsonPayload.properties.ResponseBodyText:"error")

# A user's account + recent activity
jsonPayload.properties.UserEmail="<email>"

# Container-wide errors when auth-gated fields are missing
resource.labels.container_name="invoices-api" severity>=ERROR
```

For "who/which is affected" breakdowns, pull a capped page (≤2000) projecting the field you're grouping by
(`jsonPayload.properties.AccountId`, `EndpointName`, `XA-App-Version`, …) and **aggregate locally** — count distinct
values, and keep the rows where the field was missing (don't silently drop them; that skews percentages). Note the cap
in your finding so partial counts are honest.

## Monitoring alerts

GCP Monitoring **alert policies/incidents** are available directly via `list_alert_policies` / `list_alerts` (resolve
what an alert monitors and whether it's firing). Policy-violation *events* also land in Cloud Logging as
`monitoring.googleapis.com/ViolationOpenEventv1` entries (they name the policy + condition) — useful for history.
