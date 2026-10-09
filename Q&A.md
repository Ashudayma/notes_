# Project Deep Dive — HyperTrack & DevForge (Q&A)

Priority: Very high. Detailed answers covering architecture, implementation choices, trade-offs and examples.
Full background: `hypertrack.md` and `devforge.md` in this folder.

> ⚠️ **Read before the interview.** These points were checked against the actual code. Align your story with them, because an interviewer may dig in:
> - **ERP Tracker** (Q5, Q6) is the finance service **designed and planned with DevForge** (32 stories, 234 AC, 43 WPs). The code is **not yet in the HyperTrack repo**, and all WPs are still `NotStarted`. Talk about it as "designed and in build" unless you have implemented it somewhere else.
> - **"14 roles"** (Q4): the seed data has ~9 business roles (ADMIN, PMO, PM, FH, BU, TL, DEV, BA, CLIENT) plus the WHATSAPP_BOT service role. Roles are DB rows, so admins can create more. If the live system has 14, fine. Otherwise say "~10 seeded roles, extensible, with ~38 API permissions plus hierarchical UI permissions."
> - **Duplicate handling in the WhatsApp bot** (Q8): there is **no message-id dedup today**. Present the answer as "here is how I handle / would harden it".

---

## 1. Explain the architecture of HyperTrack. What happens when a user creates a project, feature, user story and task?

**Architecture (30-second version):**
React 19 SPA → Spring Boot 4 (Java 21) REST backend → PostgreSQL. Keycloak handles identity (OIDC + PKCE) and has a custom SPI. Flowable BPMN runs the maker-checker approvals. AWS S3 stores documents. A separate WhatsApp-bot microservice lets people log timesheets from chat.

```
Browser (React, oidc-client-ts) ──Bearer──► Backend :8082 ──introspect──► Keycloak (+SPI)
                                             │ Controller → Service → Validation → Repository
                                             ├── Flowable (approval workflows)
                                             ├── Notifications (in-app + email, @Async)
                                             ├── 11 schedulers, report engine, CSV import
                                             ├──► PostgreSQL (app tables + Flowable ACT_*)
                                             └──► S3 (documents)
WhatsApp → Meta Cloud API → whatsapp-bot :8083 ──service token──► /api/internal/**
```

**Key design decisions ⭐**
- **Row versioning.** `id` is the row version and `projectId` / `featureId`… is the stable business key. `prevId` links versions. `status` marks the active (A) and historical (D) versions. `stage` holds draft/pending/approved/rejected. An edit creates a new pending row while the approved row stays live, so history comes for free.
- **Maker-checker.** Every master record goes through a Flowable approval process (projectApproval, featureApproval, userStoryApproval, taskApproval…). Each step is logged in `workflow_action`.
- **No JPA associations.** References are plain `Long xxxId` fields, and collections are `@ElementCollection` of IDs. This avoids lazy-loading and cascade surprises.
- **No N+1.** Browse pages resolve display names with SQL correlated subqueries. `getById` hydrates `@Transient` fields such as names and permission flags.

**Create flow (same template for all four levels; Feature shown):**
1. **UI:** the form sends `POST /api/features`. It never sends `createdBy`; the user comes from the token.
2. **Security:** the token is introspected at Keycloak and turned into a `User` with roles and permissions **from the token claims**, with no DB call.
3. **Validation:** `FeatureValidation` checks formats, the backdating rule and that dates fall inside the parent project's window.
4. **Business checks:**
   - The parent Project is **approved**.
   - The caller is the project owner or PMO.
   - The name is unique within the project.
   - The owner is the project's primary owner or has an approved allocation, and is not past their last working day.
5. **Save:** `stage=pending`, `status=A`, `state=Not Started`. After insert, the business key is set: `featureId = id`.
6. **Start workflow:**
   - The approver is resolved (BU Head for implementation projects, otherwise the project owner).
   - `runtimeService.startProcessInstanceByKey("featureApproval", vars)` starts the process.
   - A `workflow_action` row is written. If that write fails, the process instance is deleted (compensation).
7. **Notify:** the approver gets in-app and email notifications. These are best-effort and never roll back the save.
8. **Approve:** `POST /workflow/complete` completes the Flowable task. The stage becomes approved, the previous version becomes `D`, and the new row is live.

**Level-specific behaviour:**
- **Project:** links client, owner, co-owner and BU head; can be a sub-project; holds document metadata (BRD, SOW, PO); approved by the BU head.
- **Feature:** must sit under an approved project. `actualEfforts` rolls up from its stories.
- **User Story:** can be saved as **Backlog** (a private placeholder). Tasks can be **staged** with the story; when the story is approved they are promoted (status S → A).
- **Task:** `projectId` and `featureId` are derived, not stored. Task types are suggested from the story type. `state` (Open / In Progress / Completed) is **derived from timesheet efforts and completion %**. When all tasks complete, a story-complete notification fires, and the same happens one level up for features.

**Trade-off:** copying the service template per master means duplicated code. I accepted that because each master's validation and approver routing genuinely differ, and a shared base class would have become a mess of flags.

---

## 2. How did you implement authentication and role-based access control using Keycloak?

**Authentication**
- **Frontend:** `oidc-client-ts` with the Authorization Code + **PKCE** flow (a public SPA client, so no secret in the browser).
  - Tokens are kept in `sessionStorage`.
  - Silent renew runs on user activity and is single-flight, so a rotating refresh token is never used twice.
  - Idle timeout is 15 minutes with a warning dialog, matching Keycloak's session idle setting.
  - On a 401 the client refreshes once and retries; if that fails, it forces logout.
- **Backend:** a stateless OAuth2 resource server using **opaque-token introspection** (`SpringOpaqueTokenIntrospector`). Every request is checked with Keycloak's `/introspect` endpoint, so a logout or revocation takes effect immediately.
- **Custom Keycloak SPI:**
  - User federation over our `user_tbl`, so users live in the app DB.
  - Email OTP MFA, forced password reset, password expiry, lockout tracking (`user_attempt`) and login audit (`user_activity`).
  - A **protocol mapper** that runs **once per login or refresh**, reads user → roles → permissions from the app DB, and puts `id`, `roles`, `permissions`, `positions` and `positionTypes` into the token or introspection response as claims.

**RBAC**
- `JwtAuthenticationConverter` maps the claims to Spring authorities: `ROLE_<role>` plus each permission string (e.g. `CREATE_PROJECT`). It also builds the `User` principal. Zero DB queries per request.
- Enforcement in four layers:
  1. **Route:** `authLoader` and `permissionLoader('PROJECT_MASTER.*')` in React Router redirect to `/forbidden`.
  2. **UI element:** `<PermissionGate permission="CLIENT_MASTER.ClientMain.*.BillingSection">`.
  3. **API:** `@PreAuthorize("hasAuthority('CREATE_PROJECT')")` and `@EnableMethodSecurity`. The internal bot API uses `.requestMatchers("/api/internal/**").hasRole("WHATSAPP_BOT")`.
  4. **Service:** ownership and role checks, e.g. only the project owner or co-owner can review timesheets, and only the SIT owner approves a defect. This layer is the real source of truth; the UI checks are only for UX.
- **External (client) users** go through `ExternalUserAccessFilter`, which is deny-by-default with an allow-list (tickets, files, user info).
- **Service-to-service:** the bot uses the client-credentials grant. Its realm role `WHATSAPP_BOT` comes from `realm_access.roles`.

**Why put permissions in the token?** It avoids a DB lookup per request. The cost is staleness: a permission change only applies after the next token refresh (minutes). That was acceptable for us.

---

## 3. What is the difference between authentication and authorization? How does Keycloak handle each?

| | Authentication (AuthN) | Authorization (AuthZ) |
|---|---|---|
| Question | **Who are you?** | **What are you allowed to do?** |
| Happens | First | After AuthN |
| Failure | **401 Unauthorized** | **403 Forbidden** |
| Artifacts | Credentials, MFA, session, ID token | Roles, permissions, scopes, policies |
| Standard | OpenID Connect | OAuth2 scopes, RBAC/ABAC |

**How Keycloak handles AuthN:** login pages, the user store (built-in or federated, as with our SPI), MFA/OTP, brute-force lockout, password policy, SSO sessions, and token issuance (ID, access and refresh tokens). Flows are OIDC Authorization Code + PKCE for the SPA and client credentials for services.

**How Keycloak handles AuthZ:**
- It **issues** the authorization data: realm roles, client roles, groups and custom claims (our mapper adds permissions).
- Keycloak also has *Authorization Services* (resources, scopes, policies, UMA). We **did not use it**. We kept the decision inside the app.
- **Enforcement is done by the resource server** (Spring Security `@PreAuthorize` plus service checks), because rules like "only the owner of this project" depend on business data Keycloak doesn't have.

**In HyperTrack:** Keycloak does all of AuthN and is the **source of the claims**. The Spring backend does AuthZ enforcement. Example: introspection fails → 401. A valid user without `CREATE_PROJECT` → 403.

---

## 4. How did you implement fine-grained permissions for 14 user roles? How would you design the database schema for it?

**Model: Users ↔ Roles ↔ Permissions, with two permission types.**
- **API permissions:** flat verbs like `CREATE_PROJECT`, `EDIT_TASK`, `APPROVE_USER_STORY`, `IMPORT_DATA`, `RELEASE_RESOURCE_ALLOCATION`, `TIMESHEET_REGISTER` (~38). Checked with `@PreAuthorize`.
- **UI permissions:** hierarchical keys `MODULE.Page.Tab.Element`, e.g. `CLIENT_MASTER.ClientMain.*.BillingInformationSection`. The frontend `usePermission` hook matches them segment by segment with `*` wildcards, so one grant like `PROJECT_MASTER.*.*.*` can cover a whole module.
- Roles (ADMIN, PMO, PM, FH, BU, TL, DEV, BA, CLIENT…) are **data, not an enum**. Admins create or edit roles through the Role master, and those changes go through the same approval workflow.
- **Context rules** that a static matrix can't express sit in the service layer. Examples: project owner vs co-owner, SIT owner, fix owner, PMO-only fields.

**Schema (what we have, simplified):**

```sql
CREATE TABLE roles (
  id BIGINT PRIMARY KEY,          -- row version
  role_id BIGINT NOT NULL,        -- stable business key
  name VARCHAR(100) NOT NULL,
  stage VARCHAR(50), status CHAR(1), prev_id BIGINT,   -- versioning + approval
  created_by BIGINT, created_on TIMESTAMP, modified_by BIGINT, modified_on TIMESTAMP
);
CREATE TABLE permissions (
  id BIGINT PRIMARY KEY,
  name VARCHAR(200) UNIQUE NOT NULL,     -- 'CREATE_PROJECT' or 'CLIENT_MASTER.ClientMain.*.Billing'
  type VARCHAR(10) NOT NULL CHECK (type IN ('API','UI')),
  description TEXT
);
CREATE TABLE role_permission (role_id BIGINT, permission_id BIGINT, PRIMARY KEY (role_id, permission_id));
CREATE TABLE user_role       (user_id BIGINT, role_id BIGINT,       PRIMARY KEY (user_id, role_id));
-- junction tables store BUSINESS keys so links survive role versioning
```

**If I were redesigning for scale, I'd add:**
- `permission.module`, `permission.action` (`CREATE`/`READ`/`UPDATE`/`DELETE`/`APPROVE`) and `resource` columns instead of free-text names, to make querying and admin UIs easier.
- **Scoped grants:** `user_role(user_id, role_id, scope_type, scope_id)` so someone can be PM *of project 42* only (row-level RBAC without code).
- A `user_permission_override(user_id, permission_id, effect ALLOW|DENY)` table for exceptions.
- A `permission_audit` table recording who granted what and when.
- Cache the resolved permission set: in the token (as we do) or in Redis keyed by user, invalidated on role change.

**Why roles → permissions rather than checking role names in code?** Code checks *permissions* (`hasAuthority('APPROVE_TASK')`), never `if role == PM`. Adding a 15th role is then pure data with no deploy.

---

## 5. Explain the ERP Tracker microservice. How do you manage purchase orders, invoices and billing?

**What it is:** the finance module for ACT21 (Phase 1): Client/Project commercial info, Revenue Recognition, **PO**, **Invoice** (multi-PO allocation + GST), **Billing Tracker**, **Collection Tracker**, Alerts, Dashboard, Export, Recycle Bin, Audit Log. It was designed through DevForge from a BRD: 32 user stories, 234 AC, 43 work packages split between 2 developers.

**Architecture ⭐**
- **`erp-backend`:** a separate Spring Boot 4 / Java 21 service, a sibling module in the same monorepo.
- **Single entry point:** the browser only calls the HyperTrack backend. HyperTrack authenticates with Keycloak, then **proxies `/api/erp/**`** to erp-backend and adds trusted headers (`X-Hypertrack-User-Id`, `-Username`, `-Roles`) protected by a shared secret. erp-backend has no public ingress and no login of its own.
- **Same PostgreSQL, separate `erp` schema.** HyperTrack's tables are read-only to the ERP. `erp.client` and `erp.project` are FK-linked copies plus commercial fields (GST, PAN, payment terms, revenue-recognition type).
- **Two roles:** `finance_head` (Super Admin) and `finance` (Admin), defined in Keycloak.
- **Shared conventions:** same error envelope, same SQL migration style, no native SQL, no JPA associations. Manual alert emails go through HyperTrack's `EmailNotificationService`.

```
SPA ─► HyperTrack backend (introspect token) ─► /api/erp/** proxy + identity headers ─► erp-backend ─► erp.* schema
```

**Purchase Orders**
- Fields: project, PO number (unique), date, amount, status (**Draft / Active / Pending / Exhausted / Expired**), expiry date, frequency.
- A project has **one or more** POs.
- Computed values:
  - `Invoiced = Σ allocations from active invoices`
  - `Balance = Amount − Invoiced`
  - `% Utilised = Invoiced ÷ Amount × 100`
- **Rules:**
  - Status becomes Expired after the expiry date and Exhausted when balance = 0 (daily job).
  - The amount can be edited but **never below the amount already allocated**.
  - A PO with allocations **cannot be deleted**; everything else is soft-deleted.
  - Drafts don't count anywhere until finalised.
  - **Clone** pre-fills a new PO from an existing one.

**Invoices**
- **PO ↔ Invoice is many-to-many** through `erp.invoice_po_allocation(invoice_id, purchase_order_id, allocated_amount)`. Invoice base amount = Σ allocations, and every allocated PO must belong to the **same client**.
- **GST** is computed at save and **persisted**, not recalculated on read:
  - Same state as ACT21's home state: CGST 9% + SGST 9%.
  - Different state: IGST 18%.
- **Due date** = invoice date + client payment terms (default 30 days, with a warning).
- **DPD** (days past due) is colour-coded.
- Client billing data is **snapshotted at issue**, so a later GST or address change doesn't alter old invoices.
- Draft and Clone (e.g. for monthly recurring billing) are supported. Delete is a soft delete that releases allocations and voids the collection.
- **Integrity:**
  - The save is **one transaction**: invoice + allocations + PO balance + auto-created collection.
  - A **row lock per PO** (see Q6).
  - **Optimistic locking** with a `version` column ("reload and retry").
  - An **idempotent submit**.
  - Audit entries: ADD / EDIT / CLONE.

**Billing & Collections**
- **Billing Tracker:** per-project PO value vs invoiced vs pending, % done, next due date and status (Completed / Partial / Pending / No Invoice / **Overdue = PO expired but not fully invoiced**). An editable *Reason* field records why billing is pending.
- **Collection Tracker:** a collection is **auto-created** when an invoice is finalised and can never be created by hand.
  - Partial payments are supported (amount ≤ outstanding, where outstanding = total incl. GST − received).
  - Status updates automatically (Pending / Partially Received / Received / Overdue / Written Off).
  - Aging buckets: 0–30 / 31–60 / 61–90 / 90+ days.
  - Only the Super Admin can edit, reverse or write off, always with a reason and an audit entry.
- **Daily job** (~02:00 IST): PO status transitions, overdue recalculation, alert generation and snooze restore.

**Why a separate service?** The BRD asked for it. Finance data and deploys are isolated, with a clear ownership split: HyperTrack owns identity and delivery, the ERP owns commercial truth. Users still get one SPA and one login. **Cost:** an extra network hop and a trust boundary to secure (the shared secret and network isolation).

---

## 6. How would you prevent two finance employees from creating invoices that together exceed a purchase order's amount?

**The race:** PO = ₹10L with ₹8L already used, so the balance is ₹2L. User A and User B each create a ₹1.5L invoice at the same moment. Both read "balance ₹2L", both pass validation, both insert, and the PO ends up at ₹11L. That is a classic **read-check-write race**, a lost update.

**Primary solution: pessimistic row lock per PO, inside the save transaction (what the design specifies) ⭐**

```java
@Transactional
public Invoice finalise(InvoiceRequest req) {
    // lock in a FIXED order (sorted PO ids) → avoids deadlocks with multi-PO invoices
    List<Long> poIds = req.allocations().stream().map(Alloc::poId).sorted().toList();
    List<PurchaseOrder> pos = poRepo.findAllByIdForUpdate(poIds);   // SELECT ... FOR UPDATE

    for (Alloc a : req.allocations()) {
        PurchaseOrder po = byId(pos, a.poId());
        BigDecimal allocated = allocRepo.sumActiveAllocations(po.getId()); // re-read UNDER the lock
        if (allocated.add(a.amount()).compareTo(po.getAmount()) > 0)
            throw new AppException("Allocation exceeds available PO balance");
    }
    // insert invoice + allocations, update balances, auto-create collection — all or nothing
}

@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM PurchaseOrder p WHERE p.id IN :ids ORDER BY p.id")
List<PurchaseOrder> findAllByIdForUpdate(@Param("ids") List<Long> ids);
```

- User B's transaction **blocks** on the PO row until User A commits. B then re-reads the balance (now ₹0.5L) and is rejected with a clear message.
- **Lock in sorted ID order** so two multi-PO invoices can't deadlock (A locks PO1 then PO2, B locks PO2 then PO1). Also set a lock timeout (`jakarta.persistence.lock.timeout`).
- Keep the transaction short: no email or HTTP calls inside it. Notifications run after commit.

**Defence in depth:**
1. **Optimistic locking** (`@Version` on PO). Lighter, and good when conflicts are rare. The second writer gets `OptimisticLockException`; then retry or ask the user to reload. Weakness: it only works if every allocation also **updates the PO row** (e.g. a stored `invoiced_amount`), otherwise the version never changes.
2. **DB-level guard:** keep a denormalised `invoiced_amount` on the PO and do an atomic conditional update:
   ```sql
   UPDATE erp.purchase_order
      SET invoiced_amount = invoiced_amount + :amt, version = version + 1
    WHERE id = :poId AND invoiced_amount + :amt <= amount;
   -- 0 rows updated ⇒ over-allocation ⇒ rollback
   ```
   Add `CHECK (invoiced_amount <= amount)` as a last line of defence; the database can then never hold an invalid state.
3. **SERIALIZABLE isolation:** also works, but it causes more serialization failures and retries, so it's overkill here.
4. **Idempotent submit** (a separate problem: the same user double-clicking). Disable the button on click and send an `Idempotency-Key` header; the server stores it with a unique constraint and returns the original result on replay.

**Trade-off summary:**

| Approach | Pros | Cons |
|---|---|---|
| `SELECT … FOR UPDATE` | Simple, correct, good for hot rows | Blocks; deadlock risk if lock order isn't fixed |
| Optimistic `@Version` | No blocking, scales | Retries on conflict; PO row must be updated |
| Conditional UPDATE + CHECK | Atomic, enforced by DB | Needs a denormalised counter kept in sync |
| SERIALIZABLE | No app logic | Frequent retries, hard to reason about |

**Interview line:** "Validation in Java alone is never enough under concurrency. The check and the write must be atomic, either under a row lock or as a single conditional update, with a DB constraint as the backstop."

---

## 7. How did you integrate Amazon S3 for document storage? Why not store files directly in PostgreSQL?

**Integration**
- **Storage abstraction:** a `StorageService` interface with `AwsS3StorageService` and `AzureBlobStorageService` implementations, chosen by `storage.provider=s3|azure` (conditional beans in `StorageConfig`). This is the Strategy pattern, so we can switch cloud providers with config only.
- **S3 client:** AWS SDK v2 `S3Client` with region `ap-south-1`, an optional `endpointOverride` (for MinIO/LocalStack) and path-style access. On startup, `headBucket` checks the bucket exists; failure is fatal in prod only.
- **Upload flow** (`POST /api/file`, multipart):
  1. Validate size (1 MB) and extension allow-list (pdf, doc(x), xls(x), png, jpg, csv).
  2. Commit a `FileRecord` row with status **PENDING**.
  3. `putObject` and capture the S3 **versionId**.
  4. Update the row to **SUCCESS** or **FAILED**, recording size, content type and duration.
  - The method is deliberately non-transactional, so the PENDING/FAILED audit row survives a failed upload.
- **Download:** `GET /api/file?id=` looks up the key and version and streams the object through the backend with `StreamingResponseBody`, without loading it all into memory.
- **Linking:** business tables hold only a reference (`po_document_id`, `evidence_file_id`) and never the bytes. It's used for client NDA/MSA, project SOW/PO/BRD, timesheet and defect evidence, ticket attachments and feedback.

**Why not PostgreSQL (`bytea` / large objects)?**

| Concern | In PostgreSQL | In S3 |
|---|---|---|
| DB size and backups | Bloats the DB; backup/restore and replication slow down | DB stays small; S3 has its own durability (11 nines) |
| Performance | Large blobs churn memory and WAL, pollute the buffer cache, slow queries | Served separately; can go through a CDN |
| Cost | Expensive SSD/IOPS storage | Cheap object storage, lifecycle to Glacier |
| Scalability | Vertical only | Effectively unlimited |
| Features | Build it yourself | Versioning, encryption (SSE-KMS), lifecycle rules, presigned URLs |
| Transactions | ✅ Atomic with the row | ❌ Two systems, so keep a metadata row + status (PENDING/SUCCESS/FAILED) for consistency |

When the DB is fine: tiny files (avatars of a few KB) where transactional consistency matters more.

**What I'd improve:**
- **Presigned URLs** so the browser uploads and downloads directly from S3, keeping bytes off the backend.
- **Server-generated keys** (`{module}/{entityId}/{uuid}`) instead of client-supplied keys.
- **Ownership check on download.** Today any authenticated user can fetch a file by id, which is an IDOR risk.
- IAM role credentials instead of static access keys.
- Virus scanning, and a cleanup job for orphaned PENDING/FAILED objects.

---

## 8. How does the WhatsApp timesheet bot communicate with your backend? How would you handle duplicate requests?

**Communication flow**
```
User msg → Meta WhatsApp Cloud API → POST /webhook (whatsapp-bot, Spring Boot :8083)
  → BotService state machine (bot_session row per phone, shared Postgres)
  → HyperTrackApiClient (RestClient) + Bearer service token → backend /api/internal/**
  → TimesheetService.save() — same validation as the web app
  → reply via Graph API POST /{phone-number-id}/messages
```
1. **Webhook setup:** `GET /webhook` performs Meta's handshake. If `hub.verify_token` matches, the bot echoes `hub.challenge`. `POST /webhook` always returns **200 quickly**, because Meta retries any non-2xx response.
2. **Identify the user:** `GET /api/internal/whatsapp/user?phone=` matches the phone (country code + number) to an approved, active Resource. It returns 404 if unknown and 409 if ambiguous.
3. **Conversation:** a state machine stored in the `bot_session` table: IDLE → AWAITING_DATE → AWAITING_TASK (the bot fetches the user's open tasks for that date) → AWAITING_EFFORTS → AWAITING_COMPLETION → AWAITING_DESCRIPTION → AWAITING_CONFIRMATION → submit. Leave and idle/study flows also exist. The session times out after 5–10 minutes of inactivity.
4. **Service-to-service auth:** a Keycloak **client-credentials** client `whatsapp-bot` with realm role `WHATSAPP_BOT`. `KeycloakTokenService` caches the token and refreshes it 60 s before expiry. The backend restricts `/api/internal/**` to `hasRole('WHATSAPP_BOT')`.
5. **Submit:** `POST /api/internal/timesheets/submit {userId, taskId, date, efforts, completion%, description}`. The backend calls the **same `TimesheetService.save`** as the web app, so the daily caps, backdating window and completion rules all apply.

**Duplicate requests: where they come from**
- Meta **retries** a webhook if the response is slow or not 2xx, so the same message (same `wamid`) can arrive more than once.
- The user sends "yes" twice, or double-taps a button.
- The bot itself retries an HTTP call to the backend after a timeout, even though the first call succeeded.

**How to handle them (layered) ⭐**
1. **Message-level dedup on `wamid`:**
   ```sql
   CREATE TABLE bot_processed_message (wamid VARCHAR(128) PRIMARY KEY, received_at TIMESTAMP DEFAULT now());
   ```
   `INSERT … ON CONFLICT DO NOTHING`. If 0 rows were inserted, it's a duplicate: skip it and still return 200. A cleanup job purges rows older than 7 days.
2. **Ack fast, process async:** return 200 immediately and process via a queue or `@Async`. That avoids Meta's timeout-driven retries in the first place.
3. **State machine as a guard:** a "YES" only submits when `state == AWAITING_CONFIRMATION`. The bot moves to IDLE (or SUBMITTING) **in the same transaction** with `SELECT … FOR UPDATE` on the `bot_session` row, or optimistic `@Version`. The second "YES" then finds IDLE and is ignored.
4. **Idempotency key to the backend:** the bot sends `Idempotency-Key: <wamid>`. The backend stores it with a unique index (e.g. on the timesheet row or an `idempotency_key` table). A replay returns the original result instead of inserting again.
5. **Business-level uniqueness:** a unique constraint on the natural key (`user_id, task_id, timesheet_date, filling_for`), or the existing rule "one in-flight entry per task", makes a duplicate row impossible at the database level.
6. **Security** (related hardening): verify the `X-Hub-Signature-256` HMAC with the app secret so nobody can forge webhooks.

**Honest status:** today the protection is the state machine plus backend validation. `wamid` dedup, signature verification and async processing are the planned hardening.

---

## 9. Explain DevForge's seven-stage SDLC automation pipeline. How does data flow between stages?

**What it is:** seven prompts that run on an AI coding agent. Each acts like a specialist role and **owns specific files**. Shared state lives in versioned markdown in git. Every artifact has a permanent ID.

| # | Stage (persona) | Reads | Writes | Status power |
|---|---|---|---|---|
| 1 | **BA:** BRD → User Stories + AC | BRD | `US-xxx.md` (Given/When/Then AC), UID mapping, Missing Requirements, seeds Traceability Matrix | none |
| 2 | **Architect:** plan + WBS | Stories, BRD, **existing codebase** | `Implementation_Static` (decisions that mustn't churn), `Implementation_NonStatic` (evolving design), `WBS.md` (Work Packages) | `NotStarted` |
| 3 | **Developer:** one WP at a time | WP, plans, AC, live WBS | code + `TC-WP-xxx` Playwright tests | `→ InProgress → InReview` |
| 4 | **SIT lead** | All AC | `TC-SIT-US-xxx` end-to-end tests (derived from AC, not from code) | none |
| 5 | **Gatekeeper:** run tests | Both suites, WBS | Immutable `RUN-date-nn.md` report | **only `InReview → Completed`** |
| 6 | **Change analyst** | Change ask + **all prior CRs** | `CR-xxx.md` (always Draft) with impact per US/AC/WP | none |
| 7 | **Change implementer** | One **human-approved** CR | Appends US/AC/WPs, revises plans, closes CR | **only `→ Reopened`** |

**Data flow:**
```
BRD → P1 → [US + AC IDs] → P2 → [WPs mapped to AC] → P3 (code + WP tests) ─┐
                                              └→ P4 (SIT tests per AC) ───────┤
                                                                              ▼
                                            P5 runs both → Completed / back to P3
                       change → P6 (Draft CR) → human Approves → P7 → WPs Reopened → P3/P4/P5
```
- The **AC ID is the atom**: WPs map to AC, test titles carry AC IDs, CRs list affected AC.
- **Status lives only in `WBS.md`.** Stories never carry status; their state is derived from the WPs that implement them.
- **Promotion rule (P5):** a WP becomes Completed only if *all* tied tests pass, meaning its own WP tests **plus** every SIT test whose AC intersects the WP's AC set. Skipped, missing or failing tests keep it at InReview.
- **Bounded self-healing:** P5 may fix *locators only*, up to **3 iterations per test**, all logged. Assertion failures are never "healed". Required counters: `Assertions changed: 0`, `Application files changed: 0`.
- **Concurrency:** many developers and prompts edit the same files, so every prompt re-reads from disk before writing, edits only its own row or column, uses `max(id)+1` from disk, and lets git surface conflicts.

**Principles:** separation of duties (whoever writes code can't mark it done), human approval gates, append-only history, and "never invent: raise a question instead".

**Real use:** ERP Tracker as a continuation of the HyperTrack repo. P2 read the code first and produced a reconnaissance report. Output: 32 US, 234 AC and 43 WPs across 2 developers, with a critical path and dependency graph.

---

## 10. How do you maintain traceability between requirements, user stories, implementation plans, generated tests and change requests?

**1. Permanent, structured IDs that embed their parent:**
```
US-011 → AC-011-12 → WP-019 → TC-WP-019-03 / TC-SIT-US-011-12 → RUN-2026-08-12-01 → CR-005
```
They are never reused or renumbered; even a rejected CR keeps its number. A new ID is always `max(on disk)+1`.

**2. The IDs appear inside the artifacts themselves:**
- File names: `US-011_Invoices.md`, `TC-SIT-US-011-12_concurrent_po.spec.ts`.
- Test header block and test title: `test('[TC-SIT-US-011-12] … — AC-011-12')`. The run report maps results back by parsing these.
- WP detail block: `Implements: US-011 (AC-011-11, AC-011-12)`.
- Plans: every business rule in IMPL-D cites its AC. Every non-negotiable decision in IMPL-S cites its source (code read, BRD section or AC).
- New stories and WPs carry `Source: CR-005`. A changed AC is marked `(Changed by CR-005)`.

**3. One Traceability Matrix, one row per AC, columns owned by different stages:**

| US ID | AC ID | WP ID | Test IDs (WP/SIT) | CR IDs touching this | WP Status |
|---|---|---|---|---|---|
| US-011 | AC-011-12 | WP-019 | TC-WP-019-03, TC-SIT-US-011-12 | CR-005 (Applied) | Reopened |

P1 fills US/AC, P2 fills WP, P3 and P4 append test IDs, P6 adds `CR (Draft)`, P7 changes it to `(Applied)`, and P5 updates status. Updates are append-only and rows are never deleted.

**4. Two-way coverage gates:**
- P2 **AC coverage check:** every AC is mapped to a WP, or explicitly marked Blocked/Deferred with a reason.
- P4 **SIT coverage report:** 100% of AC have a test, or the gap is justified.
- P5: a WP with no tests is never promoted ("Blocked — no test coverage").
- P7 re-runs the coverage check after every CR, so no AC is left orphaned.

**5. Change history without breaking links:**
- A changed AC **keeps its ID**, so its tests and old run reports still resolve.
- Removed behaviour is marked `Superseded by AC-011-19 (CR-005)` or `Withdrawn`, never deleted.
- Plans have revision logs (IMPL-D R1 → R2 → R3, each citing its CR).
- Run reports are immutable.

**6. Impact analysis through the links:** P6 walks AC → WP → tests → modules, classifies every story (Fully / Partially / Not affected), and lists exactly which WPs would reopen and which tests need rework.

**Result:** questions like *"Is AC-011-12 done?"*, *"What breaks if we change GST rules?"* or *"Which tests prove the PO lock works?"* are answered from one row in one file.
