---
name: investigation-sentry
description: Sentry toolkit for automated investigations (org getpaid-inc) — REST recipes via curl for alert rules, incidents, issues, events, user/account search. Invoke whenever the task involves Sentry (an alert link, an issue id, "check sentry").
---

# Sentry for investigations (org: getpaid-inc)

All access is REST: `curl -s https://sentry.io/api/0/... -H @.tofu-ai/sentry-header.txt`.
Auth comes from the header file at that relative path (pre-materialized; run from the workspace root). **GET requests only — never POST/PUT/DELETE.** Keep calls to a handful per investigation (rate limits are unpublished).

> ⚠️ NEVER reference `$SENTRY_ACCESS_TOKEN` (or any env var) inside commands — the sandbox rejects all commands containing variable expansion. The `-H @file` form exists precisely to avoid that.
>
> ⚠️ Use the **Bash tool** (not PowerShell) and keep the exact command shape `curl -s "https://sentry.io/api/0/..." -H @.tofu-ai/sentry-header.txt` — the permission allowlist matches this literal prefix; reordered flags or other tools get gated.

> Every URL must start exactly `https://sentry.io/api/0/` (the permission allowlist is pinned to that prefix — the org-subdomain form `getpaid-inc.sentry.io` will be blocked).

## Projects (slugs)

`invoices-backend`, `invoices-web`, `invoice-generator`, `invoice-maker-ios`, `fieldservice-ios`, `fieldservice-worker-ios`, `fieldservice-worker-android`, `tofu-web-frontend`. Some endpoints need the **numeric** project id — discover via `GET /api/0/organizations/getpaid-inc/projects/`.

## Decoding an alert URL

`https://getpaid-inc.sentry.io/alerts/rules/details/<RULE_ID>/?alert=<INCIDENT_ID>&…`

```bash
# The rule definition — project, aggregate/query, thresholds, time window. Do this FIRST: it tells you what is monitored.
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/alert-rules/<RULE_ID>/" -H @.tofu-ai/sentry-header.txt

# The incident — when it fired, status, the values that tripped it
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/incidents/<INCIDENT_ID>/" -H @.tofu-ai/sentry-header.txt
```

With the rule in hand, attribute the alert by querying that project's issues within the rule's window — don't guess from org-wide spikes.

## Issues and events

```bash
# Issue by short-id (INVOICE-MAKER-IOS-2Z6 style) → numeric id, counts, status
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/issues/?query=<SHORT_ID>&project=-1" -H @.tofu-ai/sentry-header.txt

# Latest full event for an issue (tags, user, breadcrumbs, exception, contexts)
curl -s "https://sentry.io/api/0/issues/<NUMERIC_ISSUE_ID>/events/latest/" -H @.tofu-ai/sentry-header.txt

# Specific event by 32-char hex id; resolve its project first if unknown:
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/eventids/<EVENT_ID>/" -H @.tofu-ai/sentry-header.txt
curl -s "https://sentry.io/api/0/projects/getpaid-inc/<project>/events/<EVENT_ID>/" -H @.tofu-ai/sentry-header.txt

# Event counts over time — spike-onset timestamps
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/issues/<NUMERIC_ISSUE_ID>/stats/?stat=count&statsPeriod=24h" -H @.tofu-ai/sentry-header.txt
```

## Searching

```bash
# Raw issue search (Sentry query syntax: is:unresolved, release:, environment:, title:"...")
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/issues/?query=<QUERY>&statsPeriod=14d" -H @.tofu-ai/sentry-header.txt

# By end-user
...issues/?query=user.email:foo@bar.com&statsPeriod=14d

# Per-event hits (not aggregated): the events endpoint with explicit fields
curl -s "https://sentry.io/api/0/organizations/getpaid-inc/events/?query=user.email:foo@bar.com&statsPeriod=14d&field=id&field=timestamp&field=title&field=project" -H @.tofu-ai/sentry-header.txt
```

## Cross-referencing with backend logs — the accountId gotcha

Sentry's `accountId` tag stores **only the first segment** of the backend `AccountId` (up to the first `-`).

- Sentry side: `...issues/?query=accountId:<prefix>&statsPeriod=30d`
- Backend logs side (full value): `jsonPayload.properties.AccountId=~"^<prefix>"` — see the `investigation-gcp-logs` skill.

Canonical correlation: Sentry event → `user.email` / `accountId` tag + timestamp → backend request logs around that timestamp.

## Conventions for findings

- Always cite the **issue short-id** (e.g. `INVOICE-MAKER-IOS-2Z6`) as a `sentry-issue` citation on any finding about a Sentry issue — it is the cross-investigation dedupe key.
- Quote occurrence/user counts and first-seen/last-seen timestamps — they distinguish "chronic" from "new regression".
