# Investigation knowledge base

You are reading the knowledge base of past production investigations. How to use it:

1. **FIRST read `known-issues.md`** — human-verified verdicts. If the symptom you're investigating matches an entry, verify the match with 1–2 cheap checks, then return early citing the known-issue id (tag `kind:known-issue`). Do NOT deep-investigate a matching known issue unless your verification contradicts the verdict.
2. **Check for prior work before investigating anything that feels familiar:**
   - scan `INDEX.md` — one line per past run: date | id | status | tags | fingerprints | summary
   - exact-match by fingerprint: `grep -rl "sentry:<issue-id>" runs/`
   - by keyword: `grep -ril "<term>" runs/`
   - open only the matching `runs/*.md` files — verify prior conclusions against current evidence instead of rediscovering them.
3. **Before tagging your report, read `taxonomy.json`** — tag only from that vocabulary (multiple values per key allowed; unknown tags are dropped at persist time).

Rules:
- `INDEX.md` and `runs/` are machine-generated — never edit them.
- `taxonomy.json` and `known-issues.md` are human-curated — propose changes via PR, never edit during an investigation.
- No PII in anything you write: accounts by id prefix only; never quote emails, names, or phone numbers.
