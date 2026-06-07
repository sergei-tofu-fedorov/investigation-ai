---
name: investigation-history
description: Search past investigations and known issues of this very investigation service (its own API) — find whether a symptom, Sentry issue, endpoint, or account was already investigated, and read prior reports. Invoke the moment you identify a concrete identifier (Sentry short-id, request path, error type).
---

# Investigation history (self-serve recall)

The investigation service stores every past run. Search it BEFORE deep-diving anything that smells familiar — a prior report may already hold the conclusion, and a known issue may let you return early. Local API: `http://localhost:5027`.

```bash
# Full-text search over past descriptions + finding texts
curl -s "http://localhost:5027/api/investigations?q=tap2pay+500&limit=10"

# Exact evidence match — BEST signal: same Sentry short-id, commit sha, file ref
curl -s "http://localhost:5027/api/investigations?citationRef=INVOICE-MAKER-IOS-2Z6"

# By tag
curl -s "http://localhost:5027/api/investigations?tag=area:payments&limit=10"

# Read a prior run's full report (rich markdown — the complete prior conclusion)
curl -s "http://localhost:5027/api/investigations/<RUN_ID>/report"

# Human-verified known issues (also injected into your prompt — this is the live list)
curl -s "http://localhost:5027/api/investigations/known-issues"
```

## How to use what you find

- **Prior run concluded the same symptom** → read its report, VERIFY its conclusion still holds with 1–2 cheap checks against current data, then build on it — cite the run id in your report instead of re-deriving everything.
- **Known issue matches** → follow the known-issues rule from your instructions: verify cheaply, return early, tag `kind:known-issue`.
- **Prior run exists but evidence changed** (counts exploded, new release, different stack) → say so explicitly: "run X concluded …, but the situation now differs: …".
- Ignore your own in-flight run if it appears in results (it will be the `running` one matching your task).

Citing prior work makes your findings cheaper to trust and keeps the cross-run links accurate — always mention prior run ids you relied on.
