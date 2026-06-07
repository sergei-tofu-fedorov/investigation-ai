---
name: investigation-history
description: Search past investigations and known issues — the .tofu-ai knowledge tree (grep, primary) plus the service API (secondary). Invoke at investigation start AND the moment any new concrete identifier surfaces (trace id, Sentry short-id, error type, account id).
---

# Investigation history (self-serve recall)

Every past investigation is projected into the **`.tofu-ai/` knowledge tree** in your working directory — greppable text, no API needed. Search it BEFORE deep-diving anything, and again for every new identifier you surface mid-investigation.

## Primary: grep the knowledge tree

```bash
# Exact-match recall — trace ids, Sentry short-ids, fingerprints, account ids, error text.
# A hit = this was investigated before; read the file.
grep -ril "c23f65a67d88734068d761367a437f0c" .tofu-ai/runs/
grep -ril "ContentNotFoundException" .tofu-ai/runs/

# Fingerprint match (canonical error identity — sentry:<issue-id> or err:<hash>):
grep -rl "sentry:INVOICE-MAKER-IOS-2Z6" .tofu-ai/runs/

# The index — one line per run: date | id | status | tags | fingerprints | summary:
# Read .tofu-ai/INDEX.md (scan it when the ask is thematic rather than id-shaped)

# Human-verified verdicts (you must have read this already — it is mandatory first):
# Read .tofu-ai/known-issues.md
```

Run files are `runs/YYYY-MM-DD_<id8>_<slug>.md` with `## Findings` (citations, fingerprints) and `## Limitations` sections — `Read` the matching file; it carries the prior conclusion with file:line evidence.

## Secondary: the service API (`http://localhost:5027`)

For what the tree doesn't carry — live run state, the full rich report, structured filters:

```bash
# Exact evidence match across runs (same Sentry short-id, commit sha, file ref):
curl -s "http://localhost:5027/api/investigations?citationRef=INVOICE-MAKER-IOS-2Z6"

# By tag:
curl -s "http://localhost:5027/api/investigations?tag=area:payments&limit=10"

# A prior run's FULL report (rich markdown — more detail than the run file digest):
curl -s "http://localhost:5027/api/investigations/<RUN_ID>/report"
```

There is no full-text API search — free-text recall is your grep over the tree. Known issues have no endpoint — `known-issues.md` is the live list (curated via PRs).

## How to use what you find

- **Prior run concluded the same symptom** → read its run file (and `/report` for full detail), VERIFY its conclusion still holds with 1–2 cheap checks against current data, then build on it — cite the run id instead of re-deriving everything. Spend your turns on what's genuinely new.
- **Known issue matches** → verify cheaply, return early, tag `kind:known-issue`.
- **Prior run exists but evidence changed** (counts exploded, new release, different stack) → say so explicitly: "run X concluded …, but the situation now differs: …".
- Ignore your own in-flight run (its id is in your instructions).

Citing prior run ids makes your findings cheaper to trust — fingerprint matches will also be derived automatically at read time, but your explicit citation carries the reasoning.
