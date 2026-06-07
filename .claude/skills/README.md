# Investigation skills

Claude skills for the Investigations agent live here (`<skill-name>/SKILL.md` each). At deploy/setup they are wired into the agent workspace's skill discovery (copy or symlink into the workspace `.claude/skills/`).

> Skills are NOT committed yet — they reference internal infrastructure identifiers and land only after this repo is made **private**.

Planned set (currently in the agent workspace):
- `investigating` — root playbook: step order, triage heuristics, which skill to use when
- `investigation-gcp-logs` — Cloud Logging query recipes
- `investigation-sentry` — Sentry REST recipes
- `investigation-history` — searching this very knowledge base
