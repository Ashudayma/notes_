# HyperTrack — Project & Resource Management Platform

> Source: `D:\Hypertrack\hypertrack` (branch `hypertrack_live`), its `CLAUDE.md`, `bot.md`, and the backend/frontend code.
> One-liner for interviews: **"An internal delivery-management platform where every business record (Client → Project → Feature → User Story → Task, plus resourcing, timesheets, defects, tickets) goes through a maker-checker approval workflow. It uses Spring Boot, Flowable BPM, PostgreSQL, Keycloak and React."**

---

## 1. What it does

HyperTrack started as a Client/User master and grew into about 15 **maker-checker masters** covering:

| Area | Modules |
|---|---|
| Delivery hierarchy | Client → Project (incl. sub-projects) → Feature → User Story → Task, Delivery (SIT rounds), Defect |
| Resourcing | Resource, Resource Allocation (with %), Corporate Pool (bench/availability), My Allocations, Release/De-allocation, Resignation |
| Time & effort | Timesheet logging (task/defect/leave/idle/study), Timesheet Register, owner review (Green/Amber/Red per project per day) |
| Support | Ticket Master (external clients raise tickets), Escalation, Feedback |
| Org / RBAC | Users, Roles, Permissions, Positions, Position Types |
| Cross-cutting | Report engine + Developer Dashboard, in-app and email notifications, CSV bulk import, file storage, 11 schedulers, WhatsApp timesheet bot |

---

## 2. Tech stack

| Layer | Tech |
|---|---|
| Frontend | React 19, TypeScript 6, Vite 8, MUI 9, Material React Table, React Router 7, Axios, **oidc-client-ts (PKCE)**, Emotion |
| Backend | **Spring Boot 4.0.5 / Java 21**, Spring Data JPA, Spring Security (OAuth2 resource server, **opaque-token introspection**), **Flowable 8 BPMN**, springdoc, commons-csv |
| DB | **PostgreSQL**, `ddl-auto=validate`, hand-run `schema.sql` (new installs) and `migration.sql` (append-only, date-stamped). No Flyway or Liquibase |
| Auth | **Keycloak 25** (realm `HyperTrack`) with a **custom SPI**: user federation backed by `user_tbl`, plus a protocol mapper that injects roles and permissions into the token |
| Files | Pluggable `StorageService`: **AWS S3** (SDK v2) or Azure Blob, chosen by `storage.provider` |
| Email | Pluggable `EmailSenderProvider`: SMTP (works) or SendGrid (stub). Templates are stored in the DB |
| Bot | Separate Spring Boot microservice `whatsapp-bot` (:8083) on the Meta WhatsApp Cloud API |
| Deploy | Docker Compose per service; nginx serves the SPA (:3000); backend :8082; Keycloak :9090 under `/talachabi` |

---

## 3. High-level architecture

```
                ┌────────────────────────────┐
  Browser ─────►│ React SPA (nginx)          │
                │ oidc-client-ts (PKCE)      │
                └──────┬───────────────┬─────┘
          login/refresh│               │ REST + Bearer token
                       ▼               ▼
              ┌──────────────┐   ┌──────────────────────────────────┐
              │  Keycloak    │◄──│  HyperTrack backend :8082         │
              │  + custom SPI│   │  Controller → Service → Repo       │
              │  (reads app  │   │  Validation | Flowable | Schedulers│
              │   DB for     │   │  Notification | Report engine      │
              │   roles/perm)│   └──────┬───────────────┬────────────┘
              └──────┬───────┘          │ JPA/JPQL      │ S3 SDK
                     │                  ▼               ▼
                     └──────────► PostgreSQL       AWS S3 bucket
                                  (app tables +
                                   Flowable ACT_*)
  WhatsApp user ─► Meta Cloud API ─► whatsapp-bot :8083 ──(service-account token)──► /api/internal/**
                                         └── bot_session table (same DB)
```

**Layering:** `controller/` (about 35 REST controllers) → `service/` (business rules and workflow) → `validation/` (static `{Entity}Validation`) → `repository/` (Spring Data, `{Entity}Specification`, `{Entity}QueryRepository`) → PostgreSQL.

---

## 4. Core design patterns ⭐

### 4.1 Row versioning ("every edit = new row")
- `id` is the **row version** (surrogate PK). `{entity}Id` (e.g. `projectId`) is the **stable business key**, set to the first row's `id`.
- `prevId` links each version to the one before it. `status` is `A` for the active version and `D` for historical versions (`P` pending, `S` staged for tasks).
- `stage` is the approval state: draft / pending / approved / rejected / archived / staged / backlog. A separate `state` holds business progress (Not Started / In Progress / Completed).
- Result: a full audit history comes for free, and the current approved version stays live while an edit is pending.

### 4.2 Maker-checker via Flowable BPMN
- Process definitions include projectApproval, featureApproval, userStoryApproval, taskApproval, resourceAllocationApproval, defectApproval, timesheetApproval, masterApproval, clientOnboardingApproval and ticketClosureApproval.
- Every user task has `candidateGroups="ADMIN,PMO"` plus a dynamic candidate user (`${buHead}`, `${projectOwner}`, `${functionalHead}`…).
- Every action writes a `workflow_action` / `workflow_action_assignee` row. This is the audit trail behind "Pending since / Rejected by / History".
- `validateNoActivePending`: you can't edit a record while an earlier edit is still awaiting approval.

### 4.3 No JPA associations
- No `@OneToMany`, `@ManyToOne` or `@ManyToMany`. A single reference is a plain `Long xxxId`; a collection is an `@ElementCollection` of IDs (`user_role`, `role_permission`, `delivery_feature`…).
- Junction tables store the **business key**, so links survive version changes.
- Why: no LazyInitializationException, no cascade surprises, and fetch cost is visible at each call site (`findAllById` batching).

### 4.4 No N+1 on browse pages
- Paginated lists resolve display names (createdByName, ownerName…) in **SQL correlated subqueries** in `{Entity}QueryRepository.findAllWithNames()`.
- `getById` hydrates `@Transient` fields: names, child lists (contacts, staged tasks) and **server-computed permission flags** (`canPick`, `canAcknowledge`, `canReview`).
- **No native SQL.** Queries use JPQL or Criteria/Specification only.

### 4.5 Standard master API shape
`POST /search` · `GET /{id}` · `POST` (create, `?draft=`) · `POST /batch` · `PUT` · `PUT /batch` · `POST /download` (streamed CSV) · `POST /workflow/complete` · `POST /workflow/complete/batch`

---

## 5. What happens when you create Project → Feature → User Story → Task ⭐

All four follow the same `saveCore` template. Feature is the worked example below.

1. **Frontend:** a react-hook-form form calls `POST /api/features`. The payload never carries `createdBy`; the backend reads the user from the token.
2. **Security:** the token is introspected at Keycloak. `JwtAuthenticationConverter` builds a `User` (id, roles, permissions) from the claims, with **no DB query**.
3. **Validation:** `FeatureValidation.validate()` checks formats, the planned-start backdate rule (`app.feature.planned-start-backdate-days`) and dates within the parent's window.
4. **Parent check:** the parent **approved** Project must exist. `requireCreatePermission` passes only for a project owner or PMO. The name must be unique within the project. The owner must be the project's primary owner or hold an approved allocation, and must not be resigned past their last working day.
5. **Persist:** set `stage=pending`, `status=A`, `state=Not Started`, then `saveAll`. The business key `featureId = id` is assigned and the rows are saved again.
6. **Start workflow:** `submitWorkflow` resolves the approver (BU Head for IMPLEMENTATION projects, otherwise the primary owner). `WorkflowService.startWorkflows` calls `runtimeService.startProcessInstanceByKey` and writes a `workflow_action` row. **Compensation:** if the audit write fails, the process instance is deleted.
7. **Stage sync:** `stage` is overwritten with the BPMN task name (e.g. "Feature Approval").
8. **Notify:** the approver gets an in-app notification (`@Async`) and an email. Notifications are best-effort and never roll back the save.
9. **Approval** (`POST /workflow/complete`): Flowable completes the task, the stage becomes approved, the old version flips to `D`, and the new one is live.
   - **User Story approval** also promotes its **staged tasks** (status S → A).
   - When all stories under a feature are complete, `FeatureCompletionNotificationService` fires. `StoryCompletionNotificationService` does the same one level down when all tasks under a story are complete.
10. **Effort rollup:** approved timesheet hours roll up Task → Story → Feature (`actualEfforts`). Task `state` is derived automatically from efforts logged and completion %.

Extras:
- User Story supports **Backlog** (`stage='backlog'`, a creator-private placeholder).
- Task types are suggested from the story type (`/api/task/suggested-defaults`).
- `Timesheet.projectId`/`featureId` are `@Transient`, derived through the task → story → feature → project chain.

---

## 6. Authentication and authorization ⭐

### Authentication (who are you?)
- SPA uses **Authorization Code + PKCE** via `oidc-client-ts`. Tokens live in `sessionStorage`, so they survive a reload and are cleared when the tab closes.
- `automaticSilentRenew` is off on purpose. `useSessionManager` renews on user activity (`signinSilent`, single-flight so a rotating refresh token is never used twice). An inactivity timer (900 s, matching Keycloak's session idle) logs out with a warning dialog.
- The Axios client attaches the Bearer token. On a 401 it refreshes once and retries; otherwise it fires a `session:expired` event and the user is logged out.
- Backend: `SecurityConfig` makes the API a stateless OAuth2 resource server with **`.opaqueToken()`**. `SpringOpaqueTokenIntrospector` calls Keycloak `/token/introspect` on **every request**, so a revoked token stops working immediately.
- **Custom Keycloak SPI:**
  - User federation backed by `user_tbl`.
  - Email-OTP MFA (`is_mfa_enabled`), forced password reset, password expiry.
  - Login/lockout tracking in `user_activity` and `user_attempt`.
  - A **protocol mapper** reads user → roles → permissions from the app DB **once per login/refresh** and injects `id`, `userId`, `roles`, `permissions`, `positions` and `positionTypes` as top-level claims.

### Authorization (what can you do?)
- **Roles** are DB rows (`roles`, versioned, approved via the masterApproval workflow): ADMIN, PMO, PM, FH (Functional Head), BU (BU Head), TL, DEV, BA, CLIENT, plus the service role WHATSAPP_BOT. Admins can create more.
- **Permissions** (`permissions`, typed `API` or `UI`), linked via `role_permission`; users get roles via `user_role`.
  - API permissions: about 38, like `CREATE_PROJECT`, `APPROVE_TASK`, `IMPORT_DATA`, `MOVE_RESOURCE_ALLOCATION`, `TIMESHEET_REGISTER`.
  - UI permissions are hierarchical keys `MODULE.Page.Tab.Element` with `*` wildcards (e.g. `CLIENT_MASTER.ClientMain.*.BillingInformationSection`).
- `JwtAuthenticationConverter` turns the claims into Spring authorities: `ROLE_<role>` plus each permission string.
- **Enforcement happens in four layers:**
  1. Route level: `authLoader` and `permissionLoader(key)` in the React router.
  2. Component level: `<PermissionGate>` and `AppTextField permissionKey`.
  3. Endpoint level: `@PreAuthorize("hasAuthority('CREATE_PROJECT')")`. These are live on Client, Project, Import, Resource-Allocation instant-apply and the internal bot API. Elsewhere the workflow itself is the gate.
  4. **Service level:** ownership and role checks (`AuthUtils.hasAnyRole(user, PMO)`, project owner, fix owner, SIT owner…). For Ticket, Defect and Delivery, the service is the real source of truth.
- **Row-level scoping:** Specifications filter by the current user (My Allocations, Escalation visibility, Timesheet Register access by project ownership).
- `ExternalUserAccessFilter` is **deny-by-default** for EXTERNAL (client) users. They may reach only tickets, user info, config, files and onboarded projects.
- **Field-level example:** only PMO can change Resigned / Last Working Day. That needs a UI permission leaf **and** a server check comparing the new value with the stored one.

---

## 7. Other subsystems

| Subsystem | How it works |
|---|---|
| **Timesheet** | Each entry is TASK / LEAVE / IDLE / STUDY and REGULAR / LATE (after 14:00). It logs against exactly one of task or defect. Soft 8 h/day limit (an OVERTIME dialog asks to continue) and a hard 20 h ceiling. Backdating allowed up to 30 days. Completion is in 5% steps and must not decrease with later dates; evidence is required at 100%. Entries are auto-approved. Owner review is write-once. |
| **Resource Allocation** | Allocation % per project with a 100% cap. Corporate Pool shows availability as `100 − committed%`, computed with a difference array over a date range. Release/Cancel goes through approval. A daily sweep clears the leaver's ownership and emails the project owner. Instant-apply actions bypass maker-checker but require live `@PreAuthorize`. |
| **Defect** | Raised against a running SIT delivery → acknowledge/triage (pin it to a story or task, choose a fix owner) → approved by the SIT owner → in progress → resolve at 100% → approved. The `DefectAttribution` ledger is an immutable snapshot of who a defect counts against. |
| **Ticket Master** | External clients raise tickets. Pick/assign auto-generates a Story and Task (already approved). Lifecycle: Unassigned → Assigned → Sent to QA → In QA → Pending Closure → Closed / Reopened. A closure-approval BPMN starts lazily. |
| **Report engine** | `POST /api/reports {reportCode, componentType, filters}`. **Strategy pattern + Spring auto-registration:** `ReportProvider` (JPQL) produces `ReportData`, then a `ComponentFormatter` shapes it for Pie, Bar, Line, Gauge, Table or ScoreCard. A new report needs only one `@Component`. |
| **Notifications** | Two channels, both triggered by the same events: in-app (`notification` plus per-recipient `notification_recipient` read state) and email (DB templates with `{{placeholder}}`, `@Async send()` or synchronous `sendNow()` for schedulers). Dispatch-ledger tables keep reminders **idempotent**. |
| **Schedulers** | 11 `@Scheduled` jobs (calendar, expiry digest, cleanup, ticket/defect/timesheet reminders, de-allocation and resignation sweeps, personal reminders). Each gets a UUID invocation id, catches its own exceptions and uses the IST zone. There is no ShedLock, so a multi-instance deploy needs one. |
| **Bulk import** | CSV import with a `MasterHandler` per master: header validation → name→id resolution (request-scoped cache) → rule validation → chunked save **through the normal services**, so imported rows get the same workflow. Row errors go to `import_exception`. |
| **File storage** | `FileService` commits a `FileRecord` row as PENDING, uploads via `StorageService` (S3 or Azure), then marks it SUCCESS or FAILED. Metadata (key, name, type, size, S3 versionId, duration) is stored in table `file`. Entities hold only `*_document_id`. Upload is `POST /api/file` (multipart); download is `GET /api/file?id=`, streamed. |
| **Caching** | Only `UserTypeResolver` (`@Cacheable`, INTERNAL/EXTERNAL), evicted when a user changes. |

---

## 8. WhatsApp timesheet bot

```
WhatsApp user → Meta Cloud API → POST /webhook (whatsapp-bot :8083)
   → WhatsAppBotService state machine (bot_session row per phone)
   → HyperTrackApiClient → Bearer <service-account token> → /api/internal/** (backend)
   → TimesheetService.save(...) (same validation as the web UI)
   → reply via Graph API /{phone-number-id}/messages
```
- **Webhook:** `GET /webhook` performs the Meta verify handshake (`hub.verify_token` → return `hub.challenge`). `POST /webhook` always returns 200 and handles only text messages.
- **Identity:** the sender's phone (E.164 without `+`) is matched to an approved, active Resource via `CONCAT(SUBSTRING(mobileCountryCode,2), mobilePhoneNo)`. The API returns 404 if unknown and 409 if more than one resource matches.
- **Auth to backend:** a Keycloak **client-credentials** service account `whatsapp-bot` with realm role `WHATSAPP_BOT`. `KeycloakTokenService` caches the token and refreshes it 60 s before expiry. The backend requires `.requestMatchers("/api/internal/**").hasRole("WHATSAPP_BOT")` plus a class-level `@PreAuthorize`.
- **Conversation state machine:** IDLE → AWAITING_DATE → AWAITING_TASK → AWAITING_EFFORTS → AWAITING_COMPLETION → AWAITING_DESCRIPTION → AWAITING_CONFIRMATION → submit. The built version adds leave and idle/study flows. The session times out after 5–10 minutes of inactivity.
- **Internal endpoints:** `GET /whatsapp/user`, `/whatsapp/tasks`, `/whatsapp/last-working-day`, `/timesheets/my-entries`; `POST /timesheets/submit`, `/idle-study`, `/leave`.
- **Known gaps:**
  - No `X-Hub-Signature-256` verification.
  - No dedup on the WhatsApp message id (wamid), so Meta retries can be processed twice.
  - Processing is synchronous inside the webhook.

---

## 9. ERP Tracker extension (built through DevForge, see devforge.md)
- New sibling service `erp-backend` (:8083 in the plan) for PO, Invoice, Billing, Collection, Revenue Recognition and Alerts.
- The HyperTrack backend **proxies** `/api/erp/**` to it with trusted identity headers. It uses the same Postgres instance with a separate `erp` schema.
- Status: designed and planned (32 stories, 234 AC, 43 WPs). The code is **not yet in the HyperTrack repo**.

---

## 10. Engineering conventions

- Git: work lands on `hypertrack_live`. `CHANGELOG.md` is the single "why" log; `BUGS.md` holds BUG-/OPEN- IDs.
- Every DB change goes into both `schema.sql` (final state) and `migration.sql` (ALTER/UPDATE), each block prefixed with `-- YYYY-MM-DD`.
- All tables are paginated, mostly server-side.
- Draft saves skip "required" validation but still block on format errors. Empty strings become `undefined` before sending.

---

## 11. Known weaknesses (good for "what would you improve?")

1. **No concurrency control:** no `@Version` and no row locks. Allocation-% cap and uniqueness checks are read-then-write races. Fix with optimistic locking, unique constraints, and `SELECT … FOR UPDATE` on hot rows.
2. `@PreAuthorize` is commented out on many masters, and RBAC admin endpoints are open to any authenticated user until `MANAGE_RBAC` is wired.
3. File download by id has no ownership check, and the S3 key comes from the client. Fix by deriving keys server-side, checking access, and using presigned URLs.
4. Introspecting the token on every request adds latency and load on Keycloak. Options: a short-TTL introspection cache, or JWT validation via JWKS.
5. Secrets committed in `application.properties`. Move them to environment variables or a vault.
6. Schedulers have no cluster lock (add ShedLock), and the bot has no webhook signature check or dedup.
