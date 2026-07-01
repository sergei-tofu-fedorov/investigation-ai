---
name: stage-admin
description: >-
  Read AND MODIFY data in NON-PROD environments via the custom `mcp__stage_admin__*` tools — Mongo
  (mongo_find / mongo_count / mongo_insert / mongo_update / mongo_delete), Postgres (pg_query / pg_execute),
  and BigQuery (bq_query / bq_execute). Pick the target with the `conn` argument: Mongo `stage` / `dev` = the
  Invoices `invoicesDB` (accounts, invoices, estimates, clients, items, subscriptions); Postgres `auth` =
  Tofu.Auth (users, roles, tenants, invites); BigQuery `stage` = the `ai_analysis_us` analytics warehouse
  (FSM-fit marts, recurring-offer cohort). Use when asked to inspect, fix, seed, reset, or clean up stage/dev
  data, or to recalc FSM-fit / recurring-offer analytics. Holds the collection/table schema + enum encodings
  so you don't have to rediscover them. NON-PROD ONLY — never prod. Every write needs confirm:true and a
  specific filter / WHERE. Invoke BEFORE calling any stage_admin tool.
---

# Non-prod data admin (stage / dev only)

You have a narrow, audited tool surface to read and **modify data in non-prod environments**. It is **not** a shell
and **not** prod. Use it only when the user explicitly asks you to inspect or change stage/dev data.

| Backend | Tools |
|---|---|
| Mongo | `mcp__stage_admin__mongo_find`, `mcp__stage_admin__mongo_count`, `mcp__stage_admin__mongo_insert`, `mcp__stage_admin__mongo_update`, `mcp__stage_admin__mongo_delete` |
| Postgres | `mcp__stage_admin__pg_query` (read), `mcp__stage_admin__pg_execute` (write) |
| BigQuery | `mcp__stage_admin__bq_query` (read: SELECT/WITH/EXPLAIN), `mcp__stage_admin__bq_execute` (write: INSERT/UPDATE/DELETE/MERGE/CALL) |

These tools are available on any session with the stage-admin capability — the default HTTP session **and** the Slack
bot. If you don't see them in your live tool list, the capability isn't enabled for this session; say so and stop
(there is no `mongosh` / `psql` / `gcloud` / `bq` CLI here — only these MCP tools).

## Connections — pick one with `conn`

Every tool takes a `conn` argument selecting which database to talk to. If a connection isn't configured this run,
the tool says so and lists what's available.

| `conn` | Backend | What it is | Database |
|---|---|---|---|
| `stage` | Mongo | Invoices store, **stage** environment | `invoicesDB` |
| `dev` | Mongo | Invoices store, **dev** environment (same schema as stage) | `invoicesDB` |
| `auth` | Postgres | Tofu.Auth — users, roles, tenant-role assignments, invitations | (Tofu.Auth DB) |
| `stage` | BigQuery | Tofu.AI analytics warehouse (`invoicesapp-project-test`) | dataset `ai_analysis_us` |

> The `conn` label `stage` is shared by Mongo and BigQuery — the **tool** picks the backend (`mongo_*` → Mongo
> `invoicesDB`; `bq_*` → BigQuery `ai_analysis_us`). The BigQuery connection defaults its dataset to `ai_analysis_us`,
> so tables/procedures can be written **dataset-qualified without a project** (e.g. `ai_analysis_us.src_invoices`,
> `CALL ai_analysis_us.build_recurring_offer_cohort()`).

> **`stage` and `dev` are the SAME schema, DIFFERENT data** (two separate environments). Always pass the right
> `conn` — a fix meant for `dev` must not land on `stage`. State which environment you're touching back to the user.
> The `invoicesDB` Mongo store is shared by the core invoices service and the Invoices BFF, so it holds both the
> billing aggregates (invoices/estimates/accounts) and the BFF's account/catalog/subscription collections.

## The rules — read before any write

1. **NON-PROD only.** These tools are wired to stage/dev connection strings; never attempt prod, never ask for prod
   credentials. If a request is about prod data, refuse and explain it's non-prod-only.
2. **Pick the right `conn`** (`stage` vs `dev` for Mongo; `auth` for Postgres). Don't rely on a default.
3. **Look before you leap.** Before any `mongo_update`/`mongo_delete`/`pg_execute`, run the matching read
   (`mongo_find`/`mongo_count`/`pg_query`) with the *same* filter, on the *same* `conn`, and state the count back.
4. **Writes need `confirm:true`.** Treat it as a deliberate second step: read → show what will change → write with
   `confirm:true`.
5. **Always scope the write.** A Mongo write needs a **non-empty filter**; an `UPDATE`/`DELETE` needs a **WHERE**.
   Prefer the narrowest key you have (`_id` / primary key).
6. **Row cap.** Reads return at most `maxRows`; writes refuse if more than `maxRows` rows match. Narrow and batch
   rather than trying to defeat it.
7. **Single statement, no DDL.** `pg_query`/`pg_execute` run exactly one statement; no `;`-chains, no
   DROP/TRUNCATE/ALTER. Schema changes are a migration, not a stage fix.
8. **`mongo_update` uses operators** (`{ "$set": {…} }`, `$unset`, `$inc`), never a bare replacement document.
9. **Default is single-doc.** `mongo_update`/`mongo_delete` touch the first match only unless you pass `multi:true`
   — and only after a `mongo_count`.
10. **Every call is audited** (`event="stage_admin_op"`, with the `conn`, filter / SQL, and affected counts).
11. **BigQuery specifics.** `bq_query` is read-only (SELECT/WITH/EXPLAIN); `bq_execute` runs one write —
    INSERT/UPDATE/DELETE/MERGE **or `CALL <proc>`** — with `confirm:true` (UPDATE/DELETE still need a WHERE; no
    DDL — CREATE/DROP/ALTER/TRUNCATE is rejected, so schema/proc changes stay a deploy). Every BQ job is
    hard-capped at `bqMaxBytesBilled` (~1 GB) — a wide scan **fails** rather than billing; narrow with the
    partition columns (`date`) / cluster keys (`account_id`) shown below. Prefer dataset-qualified names.

---

# Mongo schema — `invoicesDB` (`conn: stage` | `dev`)

Field names below are the **stored BSON names** (no `[BsonElement]` remapping in this store — BSON names match the
C# property names). Decimals are BSON `Decimal128`; timestamps are UTC. Most collections are multi-tenant, keyed by
**`AccountId`**. Scoped aggregates use a composite `_id` of the form **`AccountId|Id`** (also stored as `UniqueId`).

### `invoices`
| Field | Type | Notes |
|---|---|---|
| `_id` / `UniqueId` | string | composite `AccountId\|Id` |
| `Id` | string | invoice id (per-account) |
| `AccountId` | string | owning account — **indexed**, primary filter |
| `Number` | string | human invoice number |
| `Date` | date | invoice date |
| `Status` | string | **enum stored as STRING** — see below |
| `MailStatus` | string | email enum (string) — present only after sending |
| `TotalAmount` / `TotalDue` / `SubtotalAmount` / `TaxAmount` / `DiscountAmount` | decimal | money |
| `CurrencyCode` | string | ISO 4217 |
| `Client` | object | embedded `{ Name, Phone?, Email?, Address?, CatalogId? }` |
| `Items` | array | line items |
| `EstimateId` / `JobId` / `OriginalInvoiceId` | string | source links (omitted if null) |
| `IsDeleted` | bool | soft-delete (omitted if null) — **add `IsDeleted: {$ne:true}` to exclude** |
| `CreatedTime` / `ModifiedTime` | datetime | base entity timestamps |

### `estimates`
Same shape as `invoices` (AccountId, Number, Date, Client, Items, money fields, `IsDeleted`, `CreatedTime`/`ModifiedTime`,
composite `_id`), **except**:
| Field | Type | Notes |
|---|---|---|
| `Status` | int | **enum stored as INT** (0–5) — see below; `null` ⇒ treat as Draft |
| `InvoiceId` | string | linked invoice once converted (omitted if null) |
| `SentMethod` | int | 0 Unknown · 1 Email · 2 Manual |

### `accounts`
| Field | Type | Notes |
|---|---|---|
| `_id` / `AccountId` | string | account id |
| `BusinessName` | string | display name (BFF) |
| `Contacts` | object | email / phone / address (BFF) |
| `Store` | string | source store (appstore / android / web) |
| `Timezone` | string | tz offset |
| `CurrencyCode` | string | default currency (omitted if null) |
| `IsDeleted` | bool | soft-delete |
| `Version` / `CreatedTime` / `ModifiedTime` | int / datetime | base entity fields |

### `accountIdentifiers` — id mapping (`_id` = `AccountId`)
`AccountId`, `UserId`, `Idfa`, `AppsflyerId`, `FirebaseId`, `VendorId`, plus `AppVersion`, `Platform` (BFF). Use to
resolve an external `UserId` / device id ↔ `AccountId`.

### `clients` — manageable client catalog
`_id` = `AccountId|ClientId`; `AccountId`, `ClientId`, `Info` (array of `{ Name, Email, Phone, Address, Type }`),
`DeletedAt`, `ArchivedAt`, `CreatedAt`, `UpdatedAt`. Exclude deleted with `DeletedAt: {$eq:null}`.

### `items` — catalog services/materials
`_id` = `AccountId|ItemId`; `AccountId`, `ItemId`, `Info` (`{ Name, Price, UnitType, Taxable, Type, Description }`),
`DeletedAt`, `CreatedAt`, `UpdatedAt`.

### `subscriptions` — mobile subscription state
`_id` = `AccountId|OriginalTransactionId`; `AccountId`, `OriginalTransactionId`, `RenewalInfo`, `Transactions[]`
(`ProductId`, `PurchaseTime`, `ExpirationTime`, `CancellationTime?`). Filter by `AccountId`.

### `masterUser` — backend user aggregate
`_id`, `PlatformUserLinks[]` (`PlatformId`, `Platform`, `Product`, `OriginalEmail?`), `OwnedAccounts[]` (`AccountId`,
`TenantRole` — null=Owner/Admin, `"Worker"`=invited), `MemberAccounts[]?`, `CreatedAt`, `UpdatedAt`, `DeletedAt?`.

### `emailStatus` — outbound email tracking
`_id` = provider `MessageId`; `AccountId`, `InvoiceId` (invoice **or** estimate id), `Type` (string enum), `Date`,
`EmailTo?`. Indexed `(AccountId, Date desc)`.

> Other `invoicesDB` collections exist (`accountData`, `regionalSettings`, `onboardings`, `entityTemplates`,
> `featureFlags`, `receipts`, `checkoutCustomers`, `contents`, `logos`, `bans`, `operationsQueue`, integrations …).
> Discover their shape with a small `mongo_find` (`limit: 2`) before editing.

## Mongo enums — get the encoding right

**Invoice `Status` — stored as STRING:** `"notPaid"`, `"paid"`, `"paidByCard"`, `"refunded"`, `"partial_refunded"`,
`"dispute"`. Filter/set the string: `{ "$set": { "Status": "paid" } }`.

**Estimate `Status` — stored as INT:** `0` Unknown · `1` Draft · `2` Sent · `3` Approved · `4` Canceled · `5` Done
(`null` ⇒ Draft). Filter/set the number: `{ "$set": { "Status": 3 } }`.

**Mail/email status (`invoices.MailStatus`, `emailStatus.Type`) — STRING:** `"sent"`, `"inProgress"`, `"opened"`,
`"markedAsSent"`, `"error"`.

Other int enums in line items / refunds: `ItemType` (0 None · 1 Service · 2 Material), `RefundStatus`
(0 Unknown · 1 Completed · 2 Canceled). String enums: `DiscountType` (`"percent"`/`"absolute"`), `TaxType`
(`"inclusive"`/`"exclusive"`).

---

# Postgres schema — Tofu.Auth (`conn: auth`)

### ⚠️ Quote identifiers — PascalCase, double-quoted
Tables and columns are **PascalCase and were created quoted**, so unquoted SQL fails (Postgres folds unquoted names
to lowercase → `relation "users" does not exist`). **Always double-quote**:
```sql
SELECT "Id", "Email", "AuthMethod" FROM "Users" WHERE "Email" = 'someone@example.com';
```
**Table names are PLURAL** (`"Users"`, `"Roles"`, `"RolePermissions"`, `"UserTenantRoles"`, `"InvitationTokens"`,
`"TokenRevocations"`, `"EmailSignInAttempts"`, `"HandoffTokens"`, `"InvitationMagicTokens"`) — except `"PermissionRegistry"`
and `"DataProtectionKeys"`. If unsure, list them: `SELECT table_name FROM information_schema.tables WHERE table_schema='public'`.
Ids are `uuid` (Users, TokenRevocations, Invitation*) or `integer` identity (Roles, RolePermissions); timestamps are
`timestamptz`. **There is no `Tenant` table** — a tenant is just a `TenantId` string on `"UserTenantRoles"`.

> **Deleting a `"Users"` row:** `"UserTenantRoles"` and `"HandoffTokens"` cascade; `"InvitationTokens"."AcceptedBy"`
> nulls out; but `"InvitationTokens"."InvitedBy"` is **RESTRICT** — if the user created any invitations, the delete
> is blocked until those rows are removed/reassigned. `"TokenRevocations"."UserId"` is **not** an enforced FK, so it
> neither blocks nor cascades (rows may be left orphaned). Check these counts before deleting a user.

### `"Users"`
| Column | Type | Notes |
|---|---|---|
| `"Id"` | uuid | PK |
| `"Email"` | varchar(320) | nullable; **unique** where not null |
| `"Name"` / `"PictureUrl"` | text | nullable |
| `"ExternalUserId"` | varchar(300) | **unique**; external auth provider id |
| `"IsAnonymous"` | bool | default false |
| `"AuthMethod"` | int | 0 None · 1 Email · 2 Google · 3 Apple · 4 Anonymous |
| `"CreatedAt"` / `"UpdatedAt"` | timestamptz | |

### `"Roles"` (seeded)
`"Id"` int PK, `"Name"` unique, `"Level"` int, `"Description"`. **Seeded: `Id=1` Admin (Level 1), `Id=2` Worker (Level 2).**

### `"RolePermissions"`
`"Id"` int PK, `"RoleId"` → Roles, `"PermissionKey"` varchar (e.g. `'invoice.view'`). Unique `(RoleId, PermissionKey)`.

### `"UserTenantRoles"` — who has which role in which tenant
| Column | Type | Notes |
|---|---|---|
| `"UserId"` | uuid | PK part 1 → Users |
| `"TenantId"` | varchar(150) | PK part 2 (tenant is a string, no table) |
| `"RoleId"` | int | → Roles |
| `"AssignedAt"` | timestamptz | |
| `"AdditionalInfo"` | jsonb | nullable |

### `"PermissionRegistry"`
`"Id"` int PK, `"Key"` unique (e.g. `'invoice.view'`), `"Title"`, `"Description"`, `"Category"`.

### `"InvitationTokens"`
`"Id"` uuid PK, `"TokenHash"` unique, `"Email"`, `"TenantId"`, `"RoleId"` → Roles, `"ExpiresAt"`, `"InvitedBy"` → Users,
`"AcceptedAt"?`, `"AcceptedBy"?` → Users, `"RevokedAt"?`, `"BaseUrl"`. Pending = `"AcceptedAt" IS NULL AND "RevokedAt" IS NULL`.

### `"EmailSignInAttempts"` — OTP rate-limiting
`"Email"` PK, `"LastAttemptDateTime"`, `"PasswordCheckCount"`, `"OneTimePassword_PasswordSha256Hash"`,
`"OneTimePassword_CreatedAt"`, `"Version"`. Clearing a stuck rate-limit = delete the row for that email.

### `"TokenRevocations"`
`"Id"` uuid PK, `"UserId"` → Users, `"Platform"` int (0 Ios · 1 Android · 2 Web), `"ProductKey"`, `"DeviceId"?`,
`"RevokedAt"`. Unique `(UserId, Platform, ProductKey, DeviceId)`.

(Also present: `"InvitationMagicTokens"`, `"HandoffTokens"`, `"DataProtectionKeys"`.)

---

# BigQuery — stage analytics warehouse (`conn: stage`, dataset `ai_analysis_us`)

Tofu.AI's analytics warehouse on `invoicesapp-project-test`. Built from the daily Atlas snapshot by SQL **procedures**
(`build_*`, orchestrated by `rebuild_warehouse`); marts are `CREATE OR REPLACE` snapshots (no history). Partitioned
tables (`date` MONTH) + clustered on `account_id` — filter on those to stay under the byte cap.

**Key tables/views** (list live ones with `bq_query`: `SELECT table_name FROM ai_analysis_us.INFORMATION_SCHEMA.TABLES`):
`src_invoices` · `src_estimates` · `src_accounts` · `src_clients` · `src_items` (typed mirrors of the Mongo collections),
`mart_account_metrics` (per-account 30d volume, avg amount, repeat ratio …), `mart_account_fsm_fit` (FSM-fit results),
`mart_account_current_plan` (active-subscription audience), `mart_recurring_offer_groups` / `mart_recurring_offer_cohort`
(recurring-offer targeting), `dim_account`, `v_fsm_fit` (metrics ⟕ fsm-fit view).

> **Stage caveats.** ⚠️ On stage `mart_account_current_plan` is a **fake static table — every account is "active"**
> (no real subscription events on non-prod), so the subscription gate is effectively always-true here. The
> `mart_recurring_offer_*` tables **don't exist until the procedures are first CALLed** (the procs `CREATE OR REPLACE`
> them). Subscription/SKU routines are prod-shaped; treat cohort output on stage as structural, not real audience.

## FSM-fit analysis (what it is)

Per account, the Hangfire job **`AnalyzeFsmFitJob`** (in `tofu-ai-api`, LLM + rule scorer) classifies the account's
invoices/estimates and UPSERTs one row into **`mart_account_fsm_fit`** (CDC via Storage Write API). It derives 6
evidence booleans (on_site_work, labour_billing, scheduling, recurring_billing, complex_multi_line_jobs,
contract_based_billing), an **industry** (3A taxonomy, e.g. cleaning / lawn_care) + specialization, a **score** (0–100)
and **tier**, an `industry_bonus`, and `recommended_offers[]`. `mart_account_fsm_fit` columns (stage-verified):
`account_id`, `industry`, `score` (FLOAT), `tier`, `industry_bonus` (FLOAT), `analyzed_at`, **`expires_at`**,
`updated_at` (plus evidence/offers/provenance columns). **Recompute is done by the C# job, not by a BQ procedure** —
`bq_execute` can only *mark* accounts for recompute (below), it cannot run the LLM.

## Recalculate `mart_recurring_offer_cohort` (pure BQ — via `bq_execute` CALL)

The cohort is 100% SQL procedures, so you can rebuild it directly. **Canonical order** (from `refresh_recurring_offer.sql`):

```
bq_execute { "conn": "stage", "confirm": true, "sql": "CALL ai_analysis_us.build_recurring_offer_groups()" }
bq_execute { "conn": "stage", "confirm": true, "sql": "CALL ai_analysis_us.build_recurring_offer_cohort()" }
```

`build_recurring_offer_groups()` builds `mart_recurring_offer_groups` (clients with ≥2 repeats by client/amount/items);
`build_recurring_offer_cohort()` then filters to eligible groups (≥3 repeats, 3A industry, active plan) into
`mart_recurring_offer_cohort`. Both take **no arguments** and `CREATE OR REPLACE` their target table.

**Upstream freshness** — the cohort reads `mart_account_metrics`, `mart_account_fsm_fit`, and `mart_account_current_plan`.
If those are stale, refresh them *first* (in this order), else re-run the whole warehouse:

```
bq_execute { "conn":"stage","confirm":true,"sql":"CALL ai_analysis_us.build_account_metrics()" }
bq_execute { "conn":"stage","confirm":true,"sql":"CALL ai_analysis_us.build_account_current_plan()" }
# then the two recurring-offer CALLs above
# (nuclear option — full rebuild from the latest snapshot: CALL ai_analysis_us.rebuild_warehouse(<snapshot_uri>, <snapshot_ts>) )
```

## Mark accounts "expired" to trigger FSM-fit recalc

`AnalyzeFsmFitJob` picks candidates = accounts with **no** `mart_account_fsm_fit` row **OR** `expires_at < CURRENT_TIMESTAMP()`
(then optional activity/maturity/volume/subscription gates), ordered by `expires_at`, capped at `BatchSize`. So to force
re-analysis of specific accounts, make their row look stale, then wait for the next job tick:

```
# Option A — expire in place (keeps history until recompute):
bq_execute { "conn":"stage","confirm":true,
  "sql":"UPDATE ai_analysis_us.mart_account_fsm_fit SET expires_at = TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 SECOND) WHERE account_id = '<ID>'" }

# Option B — cold-start (delete the row entirely → treated as never-analyzed):
bq_execute { "conn":"stage","confirm":true,
  "sql":"DELETE FROM ai_analysis_us.mart_account_fsm_fit WHERE account_id = '<ID>'" }
```

> ⚠️ **Marking only queues the recompute — it does not run it.** The actual re-analysis happens on the next
> `AnalyzeFsmFitJob` tick, and **only if that job is enabled/running on the env** (`Analyses:FsmFit:Enabled`, default
> cadence hourly). If the job is off on stage, an expired/deleted row simply stays missing until it runs. The account
> must also pass the configured gates (recent-invoice window, active plan, etc.) to be picked. Confirm intent with the
> user before bulk-expiring (each account is a real LLM call = cost + time).

---

# Recipes

**Inspect an account's invoices (read, stage):**
```
mongo_find { "conn": "stage", "coll": "invoices", "filter": { "AccountId": "<id>", "IsDeleted": { "$ne": true } },
             "projection": { "Number": 1, "Status": 1, "TotalAmount": 1, "CreatedTime": 1 },
             "sort": { "CreatedTime": -1 }, "limit": 20 }
```

**Mark one invoice paid (write, dev):** confirm the target first.
```
mongo_count  { "conn": "dev", "coll": "invoices", "filter": { "_id": "<accountId>|<invoiceId>" } }   # expect 1
mongo_update { "conn": "dev", "coll": "invoices", "filter": { "_id": "<accountId>|<invoiceId>" },
               "update": { "$set": { "Status": "paid" } }, "confirm": true }
```

**Approve an estimate (write, INT status):**
```
mongo_update { "conn": "stage", "coll": "estimates", "filter": { "_id": "<accountId>|<estimateId>" },
               "update": { "$set": { "Status": 3 } }, "confirm": true }
```

**Find a stage user and grant the Admin role in a tenant (Postgres):**
```
pg_query   (conn: auth)  SELECT "Id","Email","AuthMethod" FROM "Users" WHERE "Email" = '<email>';
pg_query   (conn: auth)  SELECT * FROM "UserTenantRoles" WHERE "UserId" = '<uuid>';
pg_execute (conn: auth, confirm:true)
           UPDATE "UserTenantRoles" SET "RoleId" = 1 WHERE "UserId" = '<uuid>' AND "TenantId" = '<tenantId>';
```

**Clear an OTP rate-limit for an email (Postgres):**
```
pg_query   (conn: auth)  SELECT "Email","PasswordCheckCount","LastAttemptDateTime" FROM "EmailSignInAttempts" WHERE "Email" = '<email>';
pg_execute (conn: auth, confirm:true)  DELETE FROM "EmailSignInAttempts" WHERE "Email" = '<email>';
```

**Bulk archive draft estimates for one account (scoped + counted):**
```
mongo_count  { "conn": "stage", "coll": "estimates", "filter": { "AccountId": "<id>", "Status": 1 } }
mongo_update { "conn": "stage", "coll": "estimates", "filter": { "AccountId": "<id>", "Status": 1 },
               "update": { "$set": { "Status": 4 } }, "multi": true, "confirm": true }
```

**Inspect one account's FSM-fit result (BigQuery read):**
```
bq_query { "conn": "stage",
  "sql": "SELECT account_id, industry, tier, score, analyzed_at, expires_at FROM ai_analysis_us.mart_account_fsm_fit WHERE account_id = '<ID>'" }
```

**Count invoices for an account, cheaply (BigQuery — filter the cluster key):**
```
bq_query { "conn": "stage", "sql": "SELECT COUNT(*) AS n FROM ai_analysis_us.src_invoices WHERE account_id = '<ID>'" }
```

**Rebuild the recurring-offer cohort (BigQuery write — two CALLs, in order):**
```
bq_execute { "conn": "stage", "confirm": true, "sql": "CALL ai_analysis_us.build_recurring_offer_groups()" }
bq_execute { "conn": "stage", "confirm": true, "sql": "CALL ai_analysis_us.build_recurring_offer_cohort()" }
```

## After a change
Report back what changed: the `conn`, the tool, the filter / WHERE, and the affected count the tool returned (e.g.
"dev · updated 1 invoice → Status=paid"; "stage · CALL build_recurring_offer_cohort → OK"). If a write reports
`matched:0` / `rowCount:0` / `affectedRows:0`, the filter missed — re-read and adjust rather than broadening blindly.
