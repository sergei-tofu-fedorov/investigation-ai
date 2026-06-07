---
name: investigating
description: ROOT playbook for backend investigations on the Tofu/Invoices platform — the step order, which specialized skill to use when (history, sentry, gcp-logs), and triage heuristics. Invoke FIRST on any investigation task.
---

# Investigation playbook (root)

You have three specialized skills — load each when its source comes into play:

| Skill | When |
|---|---|
| `investigation-history` | FIRST, and again whenever you identify a concrete identifier (Sentry short-id, path, error type) |
| `investigation-sentry` | the task mentions Sentry (alert link, issue id, client errors) |
| `investigation-gcp-logs` | before composing any `gcloud logging read` query |

## Step order

1. **Check memory first.** Known issues are in your prompt; for anything beyond them, `investigation-history` — a prior run may already hold the conclusion. Verify, don't blindly trust; build on prior work and cite run ids.
2. **Decode the ask.** Alert links carry ids — resolve the *definition* (what is monitored, thresholds) before chasing symptoms. Sentry alert URLs → `investigation-sentry`; GCP Monitoring alert URLs → the Monitoring API is NOT available, but violation events are in Cloud Logging (`monitoring.googleapis.com/ViolationOpenEventv1` log entries name the policy + condition).
3. **Establish scope before depth.** Counts, affected accounts/users, time window, first-seen — distinguishes outage / regression / chronic noise / single-account issue. Aggregate cheaply (see gcp-logs recipes) before reading individual entries.
4. **Correlate across sources.** The platform pattern: Sentry event (client view) ↔ backend request logs (server view, `AccountId` prefix gotcha) ↔ source code (`Grep`/`Read` the workspace checkouts) ↔ git history (`git -C <repo> log/show/diff` — deployed state is `origin/<default-branch>`, never checkout).
5. **Name the mechanism, not just the symptom.** A finding should reach file:line — the throw site, the mapping that produced the status code, the commit that changed behavior. "Errors went up" is scope, not a finding.
6. **Time-box.** If two approaches to a sub-question both dead-end, record it as a limitation and move on — an honest gap beats a guessed answer.

## Triage heuristics (platform-specific)

- The BFF often returns HTTP **200 with an error JSON body** — error envelopes hide from StatusCode filters.
- Auth-gated log fields (`AccountId`, `UserEmail`) are missing on early-pipeline failures — fall back to container-wide `severity>=ERROR`.
- 403 `forbidden` spikes from iOS are usually the client calling JWT-only endpoints without a session — check `AuthenticationInfoMissedException` paths before suspecting the backend.
- Identical Sentry counts across two issues ⇒ likely one client screen calling both endpoints.
- High occurrences / few users ⇒ a retry loop, not breadth. Check per-account aggregation.
- "Production-only" + same code path healthy in test ⇒ per-account/state issue, not a code break.

## Report discipline

Citations on every finding (the Sentry short-id rule matters — it is the dedupe key). Limitations for anything you couldn't check. Tags from the provided vocabulary only.
