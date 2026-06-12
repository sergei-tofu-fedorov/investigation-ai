---
name: reference-codebase
description: Map of the read-only source repos cloned under `_reference/` — what each repo is, its role in the platform architecture, and exactly where the client-facing (web / iOS) API contracts live. Load this whenever you Read/Grep reference source, trace a request to its handler, or need the route / DTO / contract a web or mobile client depends on.
---

# Reference codebase map

Read-only company source repos are cloned under `_reference/<repo>` with full git history. Read/Grep them for
code, and inspect history with `git -C _reference/<repo> log/show/diff` — never checkout; deployed state is
`origin/<default-branch>`. They are reference only — never modify, commit, or push them.

## Repos and their roles

| Repo (`_reference/…`) | Role | Look here for |
|---|---|---|
| `Invoices.Backend` | **BFF** (Backend-For-Frontend) for the **web** and **iOS-admin** clients. Authenticates the client, orchestrates the domain services (Tofu.Auth, Tofu.Invoices, …), and shapes their models into client DTOs. | **Every web/mobile API contract** — routes, request/response shapes, validation, versioning. See below. |

## Web / iOS contracts live in `Invoices.Backend` (the BFF)

When you need the contract a web or iOS client actually calls — the route, the request/response shape, validation,
or which version — look in `_reference/Invoices.Backend`:

- **Endpoints / routes:** `Src/Invoices.Api/Controllers/*.cs`. All inherit `BaseController`; API versioning is
  **per-action** via `[MapToApiVersion]` (v1.0 / v2.0 / v3.0), so the same route can have several shapes by version.
- **Request/response DTOs (the wire contract):** `Src/Invoices.Api/Dto/` (and `Src/Invoices.Api/Models/`).
- **Human-readable API references:** `Docs/API.md` + `Docs/API/*_API_REFERENCE.md` (Account, Authorization, Clients,
  Estimates, Invoices, Items, Jobs, Notifications, Payments, Teams, Invitations, Worker). The source of truth is
  Swagger at `/swagger`; the postman collection is `Docs/API/Invoices API.postman_collection.json`.

### Recipes

- Find the handler for a path the client hit:
  `Grep -n "<route segment>" _reference/Invoices.Backend/Src/Invoices.Api/Controllers`
- Find a DTO's fields (the wire shape):
  `Grep -rl "class <Name>Dto" _reference/Invoices.Backend/Src/Invoices.Api`
- See when/why a contract changed:
  `git -C _reference/Invoices.Backend log -p -- Src/Invoices.Api/Dto/<File>.cs`

> The BFF commonly returns HTTP **200 with an error JSON body** — for failures the contract is an `{ "error": … }`
> envelope, not the status code alone (see the `investigating` triage heuristics).

## Keeping this map current

As repos are added to the agent's `referenceRepoUrls`, add a row above for each — its role and where its
contracts/entrypoints live — so a reader knows which repo owns what before grepping.
