---
name: stage-admin
description: >-
  Read AND MODIFY data in NON-PROD environments via the custom `mcp__stage_admin__*` tools — Mongo
  (mongo_find / mongo_count / mongo_update / mongo_delete) and Postgres (pg_query / pg_execute). Pick the
  target with the `conn` argument: Mongo `stage` / `dev` = the Invoices `invoicesDB` (accounts, invoices,
  estimates, clients, items, subscriptions); Postgres `auth` = Tofu.Auth (users, roles, tenants, invites).
  Use when asked to inspect, fix, seed, reset, or clean up stage/dev data. Holds the collection/table schema
  + enum encodings so you don't have to rediscover them. NON-PROD ONLY — never prod. Every write needs
  confirm:true and a specific filter / WHERE. Invoke BEFORE calling any stage_admin tool.
---

# Non-prod data admin (stage / dev only)

You have a narrow, audited tool surface to read and **modify data in non-prod environments**. It is **not** a shell
and **not** prod. Use it only when the user explicitly asks you to inspect or change stage/dev data.

| Backend | Tools |
|---|---|
| Mongo | `mcp__stage_admin__mongo_find`, `mcp__stage_admin__mongo_count`, `mcp__stage_admin__mongo_update`, `mcp__stage_admin__mongo_delete` |
| Postgres | `mcp__stage_admin__pg_query` (read), `mcp__stage_admin__pg_execute` (write) |

These tools exist **only in the default HTTP session** — they are not available from the Slack bot. If you don't see
them, the capability isn't enabled for this session; say so and stop (there is no `mongosh` / `psql` / `gcloud` here).

## Connections — pick one with `conn`

Every tool takes a `conn` argument selecting which database to talk to. If a connection isn't configured this run,
the tool says so and lists what's available.

| `conn` | Backend | What it is | Database |
|---|---|---|---|
| `stage` | Mongo | Invoices store, **stage** environment | `invoicesDB` |
| `dev` | Mongo | Invoices store, **dev** environment (same schema as stage) | `invoicesDB` |
| `auth` | Postgres | Tofu.Auth — users, roles, tenant-role assignments, invitations | (Tofu.Auth DB) |

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

## After a change
Report back what changed: the `conn`, the tool, the filter / WHERE, and the affected count the tool returned (e.g.
"dev · updated 1 invoice → Status=paid"). If a write reports `matched:0` / `rowCount:0`, the filter missed — re-read
and adjust rather than broadening blindly.
