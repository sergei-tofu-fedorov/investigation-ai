# investigation-ai — knowledge repo for the Investigations agent

The knowledge base of the Tofu.AI Investigations module (FS-1111): everything the Claude CLI agent knows beyond live evidence, as **greppable text files**. Design rationale: `Local.Docs/features/FS-1111/agent-context-pull.md` in the backend workspace.

> ⚠️ **This repo must be PRIVATE before any investigation content (runs/, known-issues entries) or skills land** — findings reference internal infrastructure, error details, and account identifiers.

## Layout

| Path | Kind | Owner | Written how |
|---|---|---|---|
| `taxonomy.json` | **source** | humans | PR — the closed tag vocabulary; persist-time validation reads it |
| `known-issues.md` | **source** | humans | PR — verified verdicts the agent checks FIRST and returns early on |
| `INDEX.md` | projection | the service | regenerated from Postgres; one line per investigation run |
| `runs/` | projection | the service | one markdown file per run: findings, citations, fingerprints, limitations |
| `.claude/skills/` | source | humans | PR — investigation skills the agent loads (wired into the agent workspace at deploy) |

**Ownership rule (prevents merge conflicts):** the service writes only `INDEX.md` + `runs/`; humans write only the source files. Don't hand-edit projections — they are rebuilt from Postgres and your edits will be overwritten.

## How it's consumed

1. The Tofu.AI service clones/pulls this repo at startup (`Investigations:KnowledgeRepoPath` points at the local checkout).
2. Source files are copied into the agent's `.tofu-ai/` context tree alongside the Postgres-projected `INDEX.md` + `runs/`.
3. The agent `Read`s/`Grep`s the tree — it never queries the database.
4. After each investigation the service appends a run file + INDEX line (container phase: commits + pushes here, best-effort).

## Conventions

- Run files: `runs/YYYY-MM-DD_<id8>_<slug>.md` with `## Findings` / `## Root cause` / `## Limitations` headings — consistent so section-targeted grep works.
- Fingerprints appear **verbatim** in run files and INDEX (`sentry:<issue-id>` or `sha256:<hash>`) — "was this investigated before?" = `grep -rl "<fingerprint>" runs/`.
- `INDEX.md` is capped ~25 KB: beyond that it keeps recent runs + stats and the agent greps `runs/` for the tail.
- No PII anywhere: no emails, names, or phone numbers; accounts referenced by id prefix only.
