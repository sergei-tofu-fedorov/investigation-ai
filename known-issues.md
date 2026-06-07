# Known issues — human-verified verdicts

The agent reads this FIRST on every investigation: on a symptom match, verify cheaply (1–2 checks), return early citing the entry id, tag `kind:known-issue` — unless verification contradicts the verdict, in which case say so explicitly.

**Curated via PR only.** Entry format:

```
## KI-<NNN> — <one-line symptom pattern>
- **Verdict:** known-benign | known-bug | wontfix | monitor
- **Fingerprint:** <sentry:<issue-id> or sha256:<hash>, if stable>
- **Note:** <why this verdict; what (not) to do>
- **Source run:** <run id, if promoted from an investigation>
- **Added:** YYYY-MM-DD by <who>
```

Deactivate an entry by moving it under `## Deactivated` (keep it — past runs cite these ids).

---

<!-- entries below; no real entries yet -->

## Deactivated
