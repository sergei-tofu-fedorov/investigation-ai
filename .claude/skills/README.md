# Investigation skills

Claude skills for the Investigations agent (`<skill-name>/SKILL.md` each). At deploy/setup they are wired into the agent workspace's skill discovery (copy or symlink into the workspace `.claude/skills/`); the copy in the agent workspace is the live one — this repo is the versioned source.

| Skill | Purpose |
|---|---|
| `investigating` | root playbook: step order, triage heuristics, which skill to use when — invoked FIRST on every run |
| `investigation-gcp-logs` | Cloud Logging query recipes (LQL field paths, aggregation, LB-trace correlation) |
| `investigation-sentry` | Sentry REST recipes (issues, events, alert rules) |
| `investigation-history` | searching this very knowledge base (INDEX/runs grep + the service API) |

Curated via PR, like the other source files.
