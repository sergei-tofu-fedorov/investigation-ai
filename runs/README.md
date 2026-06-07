# runs/ — per-investigation digests

MACHINE-GENERATED from Postgres by Tofu.AI — do not edit; rebuilt on drift.

- Naming: `YYYY-MM-DD_<id8>_<slug>.md` (sortable by date, greppable by id).
- Fingerprints appear verbatim (`sentry:<issue-id>`, `sha256:<hash>`) — exact-match recall is `grep -rl "<fingerprint>" .`
- Section layout (consistent for section-targeted grep):

```
# <date> — <one-line ask> (run <id>)
status · requested-by · duration · tags

## Findings
1. <summary> (confidence NN)
   - citations: <kind:ref, …>
   - fingerprint: <…>

## Root cause
<when identified>

## Limitations
- <parts of the ask that needed unavailable sources>
```
