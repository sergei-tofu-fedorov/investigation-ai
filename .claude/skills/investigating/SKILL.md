---
name: investigating
description: ROOT playbook for backend investigations on the Tofu/Invoices platform — the step order, which specialized skill to use when (history, sentry, gcp-logs), and triage heuristics. Invoke FIRST on any investigation task.
---

# Investigation playbook (root)

You have specialized skills — load each when its source comes into play:

| Skill | When |
|---|---|
| `investigation-history` | grep recipes for the knowledge tree at the working-directory root — the history-first step and the continuous-matching rule below |
| `investigation-sentry` | the task mentions Sentry (alert link, issue id, client errors) |
| `investigation-gcp-logs` | before composing any `gcloud logging read` query |
| `reference-codebase` | reading/grepping the `_reference/` source checkouts, or finding the web/iOS API contract (route, DTO) a client depends on — `Invoices.Backend` is the BFF that owns those contracts |

## Step order

1. **History FIRST — before ANY source query.** In one parallel batch:
   - `Read known-issues.md` — on a symptom match: verify with 1–2 cheap checks, return early, tag `kind:known-issue`.
   - `Grep runs/` (and scan `INDEX.md`) for **every literal identifier in the ask**: trace id, Sentry short-id, account id, request path, error text, contentId. A bare id won't *feel* familiar — grep it anyway; the hit IS the familiarity check.
   - On a hit: `Read` the matching run file(s) before touching logs/Sentry — a prior run may already hold the conclusion. Verify, don't blindly trust; build on it and cite the run id.
2. **Decode the ask.** Alert links carry ids — resolve the *definition* (what is monitored, thresholds) before chasing symptoms. Sentry alert URLs → `investigation-sentry`; GCP Monitoring alert URLs → the Monitoring API is NOT available, but violation events are in Cloud Logging (`monitoring.googleapis.com/ViolationOpenEventv1` log entries name the policy + condition).
3. **Establish scope before depth.** Counts, affected accounts/users, time window, first-seen — distinguishes outage / regression / chronic noise / single-account issue. Aggregate cheaply (see gcp-logs recipes) before reading individual entries.
4. **Correlate across sources.** The platform pattern: Sentry event (client view) ↔ backend request logs (server view, `AccountId` prefix gotcha) ↔ source code (`Grep`/`Read` the workspace checkouts) ↔ git history (`git -C <repo> log/show/diff` — deployed state is `origin/<default-branch>`, never checkout).
5. **Name the mechanism, not just the symptom.** A finding should reach file:line — the throw site, the mapping that produced the status code, the commit that changed behavior. "Errors went up" is scope, not a finding.
6. **Time-box.** If two approaches to a sub-question both dead-end, record it as a limitation and move on — an honest gap beats a guessed answer.

## Continuous history matching (standing rule, not a step)

Evidence collection SURFACES new identifiers the ask didn't contain — an exception type, an error message, a Sentry short-id, an account id, an endpoint path. **The moment a new concrete identifier appears, grep `runs/` for it** — add the grep to the same parallel batch as your next source query, it costs nothing. A hit means this thread was already walked: read the run file, reuse its mechanism/file:line conclusions (verified), and spend your remaining turns on what's genuinely new. This is how repeat root causes get answered in seconds instead of re-derived in minutes.

## Triage heuristics (platform-specific)

- The BFF often returns HTTP **200 with an error JSON body** — error envelopes hide from StatusCode filters.
- Auth-gated log fields (`AccountId`, `UserEmail`) are missing on early-pipeline failures — fall back to container-wide `severity>=ERROR`.
- 403 `forbidden` spikes from iOS are usually the client calling JWT-only endpoints without a session — check `AuthenticationInfoMissedException` paths before suspecting the backend.
- Identical Sentry counts across two issues ⇒ likely one client screen calling both endpoints.
- High occurrences / few users ⇒ a retry loop, not breadth. Check per-account aggregation.
- "Production-only" + same code path healthy in test ⇒ per-account/state issue, not a code break.

## Report discipline

Citations on every finding (the Sentry short-id rule matters — it is the dedupe key). Cite prior run ids you built on. Limitations for anything you couldn't check. Tags from `taxonomy.json` only.
