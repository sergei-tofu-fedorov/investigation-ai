---
name: investigation-gcp-logs
description: GCP Cloud Logging reference for automated investigations — exact LQL field paths for Invoices.Backend request logs, query/aggregation recipes, LB-trace correlation, project rules. Invoke BEFORE composing any gcloud logging read query.
---

# GCP logs for investigations (Invoices platform)

Uses the locally-authenticated gcloud CLI. Only `gcloud logging read` is permitted — never any mutating gcloud command.

## Projects

- **prod**: `inv-project` — real traffic; READ-ONLY queries allowed and expected when investigating production issues.
- **test**: `invoicesapp-project-test` — default for anything exploratory.
- Always pass `--project=<id>` explicitly.
- ⚠️ Test-project gotchas: `dev-gateway-api`/`dev2-gateway-api` are **gateway proxies, not the BFF**; the test project co-locates sibling services (`tofu-invoices-api`, `auth-api`) in the same logs. Filter by container explicitly.

## Command shape

```
gcloud logging read '<LQL>' --project=inv-project --limit=50 --freshness=24h --format=json
```

- Always bound with `--limit` (default 50; aggregations ≤2000) and `--freshness`. Note in findings when a limit was hit — counts are then partial.
- `entries.list` quota is 60 req/min/project — ask fewer, broader questions and filter locally.

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

`trace` is the **join key** between LB and app logs: `trace="projects/<proj>/traces/<TRACE_ID>"`. Interpretation: `Elapsed ≈ latency` ⇒ slowness is in-app; `latency >> Elapsed` ⇒ LB/network overhead.

## Recipes

```bash
# Errors on a path, last 24h (remember the 200-error-envelope gotcha)
gcloud logging read 'jsonPayload.properties.RequestPath:"/api/tap2pay" AND (jsonPayload.properties.StatusCode>=500 OR jsonPayload.properties.ResponseBodyText:"error")' --project=inv-project --limit=50 --freshness=24h --format=json

# Aggregate distinct values with counts (who is affected / which endpoints / which versions)
gcloud logging read '<filter>' --project=inv-project --limit=2000 --freshness=24h --format='value(jsonPayload.properties.AccountId)' | awk 'NF==0{print "(empty)"; next} {print}' | sort | uniq -c | sort -rn
# the awk keeps rows where the field was missing — silently dropping them skews percentages

# Multi-field aggregation — | separator survives commas in paths
--format='csv[no-heading,separator="|"](jsonPayload.properties.RequestMethod,jsonPayload.properties.EndpointName)'

# Find a user's account + recent activity
gcloud logging read 'jsonPayload.properties.UserEmail="<email>"' --project=inv-project --limit=50 --freshness=7d --format=json

# Container-wide errors when auth-gated fields are missing
gcloud logging read 'resource.labels.container_name="invoices-api" severity>=ERROR' --project=inv-project --limit=50 --freshness=2h --format=json
```
