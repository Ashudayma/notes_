# DevForge — AI-Driven SDLC Pipeline (BRD → Delivery)

> Source: `D:\ERP_AI\prompts\prompt1.md … prompt7.md` + the artifacts it produced in `D:\ERP_AI\` (BRD, User Stories, Implementation plans, WBS, Traceability Matrix).
> First real use: building the **ACT21 ERP Tracker** (finance module: PO, Invoice, Billing, Collection, Revenue Recognition, Alerts) as a new service on top of HyperTrack.

---

## 1. What DevForge is (one paragraph)

DevForge is a **seven-prompt, file-based SDLC pipeline** that runs on an AI coding agent (Claude Code). Each prompt is a strict "role contract": it owns specific files, may only make specific status transitions, and must re-read shared files before writing. The output of one prompt is the input of the next. Every artifact has a **permanent ID** (US, AC, WP, TC, RUN, CR), and a single shared **Traceability Matrix** links them, so you can always answer *"which requirement → which work package → which tests → which change requests → what's its status?"* from one file.

**The core idea:** treat the LLM like a team of specialists (BA, Architect, Developer, QA Lead, Release Gatekeeper, Change Analyst, Change Implementer) with **separation of duties** — no single step can both write code and declare it done.

---

## 2. The seven stages

| # | Prompt | Persona | Input | Output (files it owns) | Status it may set |
|---|---|---|---|---|---|
| 1 | BRD → User Stories & AC | Business Analyst / PO | `BRD/BRD_v{n}.md` | `User_Stories_BA/US-xxx_*.md`, `User_Story_AC_UID_Mapping.md`, `Missing_Requirements_BA.md`, seeds `Traceability_Matrix.md` | none (stories never carry status) |
| 2 | Implementation Plan + WBS | Solution Architect / Tech Lead | Stories, mapping, BRD, **existing codebase** | `Implementation_Static.md` (IMPL-S), `Implementation_NonStatic.md` (IMPL-D), `WBS/WBS.md` | `NotStarted` (for every new WP) |
| 3 | Development + WP-level tests | Senior Full-Stack Dev | One WP from WBS, plans, linked AC | App code, `Tests/WP/TC-WP-xxx-yy_*.spec.ts` | `NotStarted/Reopened → InProgress → InReview` |
| 4 | SIT tests | SIT / QA Automation Lead | All stories + AC | `Tests/SIT/TC-SIT-US-xxx-yy_*.spec.ts` | none |
| 5 | Test execution + promotion | Test Execution Engineer / Release Gatekeeper | Both test suites, WBS, matrix | `TestRuns/RUN-YYYY-MM-DD-nn.md` (immutable) | **only** `InReview → Completed` |
| 6 | Change Request analysis | Change Control Analyst | Change ask + **every prior CR** | `CRs/CR-xxx.md` (always `Draft`) | none |
| 7 | CR application + loop-back | Change Implementer / Delivery Mgr | One **Approved** CR | Appends stories/AC/WPs, updates plans, closes CR (`Applied`) | **only** `→ Reopened` (and `NotStarted` for new WPs) |

```
 BRD ─► [P1 BA] ─► US + AC ─► [P2 Architect] ─► IMPL-S, IMPL-D, WBS (WPs)
                                                     │
                     ┌───────────────────────────────┤
                     ▼                               ▼
              [P3 Dev, per WP]                [P4 SIT, per US]
              code + WP tests                 end-to-end tests
              WP → InReview                          │
                     └──────────────┬────────────────┘
                                    ▼
                          [P5 Run + Gatekeeper]
                     all tied tests pass → Completed
                     any fail → back to P3 (bug) or P6 (intended change)
                                    │
                    change requested│
                                    ▼
                        [P6 CR analysis — Draft]
                                    │ human sets Approved
                                    ▼
                        [P7 Apply CR] → WPs Reopened
                                    │
                                    └──► loop back to P3 / P4 / P5
```

---

## 3. Stage-by-stage detail

### Prompt 1 — BRD → User Stories, Acceptance Criteria, Missing Requirements
- Splits the BRD into **INVEST** user stories, one per file: `User_Stories_BA/US-001_Login.md`.
- Each story: Title, ID, "As a / I want / so that", Description, Assumptions, Dependencies, **AC in Given/When/Then** (each with its own ID), Business Exceptions, Business Rules, Definition of Done.
- AC categories covered: Happy Path, Business Rules, Authorization & Access, Exception Handling, UI/UX, Performance, Audit, Notifications.
- **Deliberately excludes** field-level validation (length, format, XSS…) unless the BRD states it as a business rule — keeps AC business-focused.
- `User_Story_AC_UID_Mapping.md` = a **derived index/dictionary** of every US + AC (title, file, category) so you never open 30 files to look something up.
- `Missing_Requirements_BA.md` = BA backlog of gaps/ambiguities, priority-tagged 🔴🟠🟡🟢, plus a gap-analysis table. **Rule: when unsure, raise a question — never invent a requirement.**
- Seeds `Traceability_Matrix.md` with **one row per AC**; the other columns stay `—`.

### Prompt 2 — Implementation Plan (Static + Non-Static) + WBS
- **Blocking intake:** asks (1) Greenfield / Continuation / Merge-into-host? and (2) the developer roster (name, role, capacity). Won't generate anything until answered.
- In Continuation/Merge mode it must **read the existing code first** and publish a *Codebase Reconnaissance Report* (stack versions from pom/package.json, auth, ORM, conventions, test setup). **Hard rule: no invention** — every statement must trace to code read, BRD, or an AC; anything unverified goes to *Open Technical Questions*.
- **IMPL-S (Static)** — decisions that must not churn: architecture style, stack, host integration, data model, API conventions, cross-cutting concerns, **Non-Negotiable Decisions table (each with a cited source)**, test architecture baseline (`data-testid` convention), Do-Not-Touch list. **Outranks IMPL-D.**
- **IMPL-D (Non-Static)** — evolving detail: module breakdown, per-module rules (each citing an AC), sequencing, risks. Revised on every CR.
- **WBS** — Work Packages sized for parallel work: one owner, 1–3 dev-days, independently testable, ≥1 AC (or a justified `Enabler`), cut along interface seams, load-balanced. Contains: roster, WP register, **WP↔US/AC many-to-many mapping**, AC coverage check, Mermaid dependency graph (acyclic), parallel track plan + critical path, change log.

### Prompt 3 — Development + WP-level Playwright tests (one WP per run)
- Asks which WP; validates its live status (refuses to touch a `Completed` WP — must come via CR).
- Reads both plans, linked AC (full text), other WPs' live status (build against what exists, **stub** what doesn't), and the real codebase.
- Produces an implementation brief: what exists vs stubbed, static constraints, files to touch, interfaces, business rules per AC, error/permission behaviour, build order, self-review checklist.
- Writes WP tests `TC-WP-001-01_*.spec.ts`: header block with IDs, ID + AC in test title, `data-testid`/role locators only, isolated data, web-first assertions, no `test.skip`.
- Moves WP to **`InReview`** — **never `Completed`**. ("InReview = I believe it's done; Completed = tests proved it.")

### Prompt 4 — SIT tests (User-Story level)
- Derived **from AC, not from implementation** — if code and AC disagree, the test follows the AC and the ambiguity is raised to the BA.
- One test set per AC; full journey as the named role; no internal stubbing; Given/When/Then structure; header quotes the source AC text (makes AC drift visible in code review).
- Unbuilt features still get a test → reported as **Expected-Fail** (an unimplemented AC must not look green).
- Coverage report target: **100% of AC**, every gap justified.

### Prompt 5 — Execution, run report, auto-promotion
- Runs WP + SIT suites (never against production). New immutable `RUN-YYYY-MM-DD-nn.md` every run.
- **Promotion rule:** WP goes `InReview → Completed` only if **every tied test** (its own `TC-WP-xxx-*` **plus** every SIT test whose AC intersects the WP's AC set) executed and passed. Skipped / not executed / no tests → stays `InReview`. Never demotes a `Completed` WP — reports a **regression** instead.
- **Bounded self-healing:** may repair *only* locators broken by incidental DOM churn — max **3 heal iterations per test**, each logged (old/new locator, justification). **Any assertion failure is a failure — full stop.** Mandatory counters: `Assertions changed: 0`, `Application files changed: 0`, `Tests given a 4th iteration: 0`.
- Flaky passes (passed on retry) and healed passes are counted as passes but **reported separately**.
- Every failure gets a classification (app defect / test defect / env / requirement ambiguity) and a route (P3, P6, or BA).

### Prompt 6 — Change Request analysis (analysis only)
- Captures the change in business terms (what, who, why now, target date).
- Reads **every prior CR** and builds a *Prior CR Reconciliation* table (No interaction / Reinforces / Overlaps / Conflicts / Supersedes). Effective baseline = BRD + stories + applied CRs.
- Classifies **every** US as Fully / Partially (names exact AC + change type: Changed / Invalidated / Extended / New AC needed) / Not Affected.
- Blast radius: WPs (live status snapshot), tests, modules, data model, integrations, NFRs.
- Two control fields: **`Static Plan Impact: Yes/No`** (with justification — the only gate that allows editing IMPL-S) and **`CR Status: Draft`** (always; only a human sets `Approved`).
- Zero footprint on delivery artifacts — a rejected CR leaves nothing behind.

### Prompt 7 — Apply approved CR + loop back
- Hard gate: refuses unless CR is `Approved` with approver/date, Static impact set, critical questions answered, not already applied, no unresolved conflict with another approved CR.
- **Additive only:** new US/AC/WP get `max+1` IDs; changed AC keep their ID (`Changed by CR-xxx`); removed AC marked `Superseded`/`Withdrawn` — never deleted.
- Reopen table: affected `Completed`/`InReview` WP → **`Reopened`**; `InProgress`/`NotStarted` → scope update only; `Deferred` → human decision.
- Always revises IMPL-D; revises IMPL-S **only if** `Static Plan Impact: Yes`.
- Ends with a mandatory **Loop-Back** section naming exact WP IDs for P3, US IDs for P4, then P5 verification, human decisions, schedule impact.

---

## 4. How data flows between stages

| Artifact | Written by | Read by | Key link |
|---|---|---|---|
| BRD | human | P1, P2, P6, P7 | source of truth for business |
| US/AC files | P1, P7 (append) | P2, P3, P4, P6 | AC IDs are the atom of traceability |
| Mapping dictionary | P1, P7 | P2, P4, P6 | index of every US/AC |
| Missing_Requirements_BA | P1 (+ P2/P4 append) | BA, P2 | open questions instead of guesses |
| IMPL-S / IMPL-D | P2, P7 | P3, P4, P5, P6 | decisions + module design |
| WBS (only place with status) | P2, P3, P5, P7 | everyone | WP ↔ AC mapping, status |
| Tests (WP / SIT) | P3 / P4 | P5 | test titles carry TC ID + AC ID |
| Run reports | P5 | P3, P6 | evidence for promotion |
| CRs | P6, P7 | P6 (all prior), P7 | change history |
| **Traceability Matrix** | every prompt, **own columns only** | everyone | the single "state of X" view |

**Traceability Matrix — fixed 6 columns, each owned by different prompts:**

```
| US ID | AC ID | WP ID | Test IDs (WP/SIT) | CR IDs touching this | WP Status |
  P1      P1     P2/P7   P3 + P4 (append)      P6 (Draft)→P7 (Applied)  P2/P3/P5/P7 (mirror of WBS)
```

---

## 5. ID scheme (all permanent, never reused/renumbered)

| Artifact | Format | Example |
|---|---|---|
| User Story | `US-{3d}` | `US-011` |
| Acceptance Criterion | `AC-{US}-{2d}` | `AC-011-12` |
| Work Package | `WP-{3d}` | `WP-019` |
| WP test | `TC-WP-{WP}-{2d}` | `TC-WP-019-03` |
| SIT test | `TC-SIT-US-{US}-{2d}` | `TC-SIT-US-011-12` |
| Test run | `RUN-{date}-{2d}` | `RUN-2026-08-12-01` |
| Change Request | `CR-{3d}` | `CR-005` (rejected CRs keep their number) |
| Plans | `IMPL-S` / `IMPL-D` with revisions `R1, R2…` | `IMPL-D R3` |

New IDs are always `max(existing on disk) + 1`, **re-read from disk** — never from the model's memory.

---

## 6. Status model (lives ONLY in WBS)

```
NotStarted ──P3──► InProgress ──P3──► InReview ──P5 (all tied tests pass)──► Completed
                        ▲                  │                                    │
                        └──────P3──── Reopened ◄────────P7 (approved CR)────────┘
Deferred = human decision only
```

| Status | Who sets it |
|---|---|
| NotStarted | P2 (and P7 for new WPs) |
| InProgress / InReview | P3 only |
| **Completed** | **P5 only — automatic, evidence-based** |
| **Reopened** | **P7 only** |
| Deferred | human |

User Stories/AC **never** carry status — their state is **derived** from the WPs that implement them (all WPs Completed → AC Completed; any Reopened → AC Reopened, etc.). *Why:* storing status in two places = two sources of truth that drift.

---

## 7. Cross-cutting design principles (interview talking points)

1. **Separation of duties** — the prompt that writes code can't mark it done; the prompt that analyses a CR can't apply it; approval is always human.
2. **Single source of truth per fact** — status only in WBS; story content only in story files; mapping is a *derived* index.
3. **Append-only history** — revision logs, change logs, run reports, CRs are never edited or deleted; superseded AC keep IDs so old tests/reports still resolve.
4. **No hallucination policy** — unknowns become *Open Technical Questions* or *Missing Requirements*, never "decisions". Every non-negotiable cites its source.
5. **Concurrency on plain files** — multiple devs + prompts edit shared markdown, so: re-read before write, edit only your row/column, one atomic edit per change, let git surface conflicts.
6. **Tests as the gate** — evidence-based promotion, honest failures (Expected-Fail), bounded self-healing (locators only, 3 iterations, assertions never touched).
7. **Static vs Non-Static plan** — architecture decisions are protected by an explicit `Static Plan Impact` gate so a routine CR can't silently rewrite architecture.
8. **Blocking intake questions** — the pipeline stops and asks (project mode, roster, which WP, which CR) instead of guessing.

---

## 8. Real run — ERP Tracker on HyperTrack

| Item | Value |
|---|---|
| BRD | `BRD/brd_v1_erp.md` — ACT21 ERP Tracker, Phase 1 |
| Mode | **Continuation** — HyperTrack repo @ `hypertrack_live` (commit pinned) |
| User Stories | **32** (US-001 … US-032) |
| Acceptance Criteria | **234** |
| Work Packages | **43** (WP-001 … WP-043) |
| Roster | 2 Fullstack devs (Ashu, Mohit), ~50 dev-days each |
| Highest-risk WPs | WP-002 (proxy + trust-header authz), WP-019 (invoice transactional integrity) |

**Architecture that P2 decided (IMPL-S):**
- New sibling Spring Boot 4 / Java 21 module **`erp-backend` (:8083)** in the same monorepo — not a package in `backend/`, not a new repo.
- Browser → HyperTrack backend (Keycloak opaque-token introspection) → **proxy `/api/erp/**`** → erp-backend with trusted headers `X-Hypertrack-User-Id/-Username/-Roles` + shared secret; erp-backend has no public ingress.
- Same PostgreSQL, new **`erp` schema**; HyperTrack `client`/`project` read-only; ERP keeps FK-linked copies `erp.client`, `erp.project` + commercial fields.
- Two roles only: `finance_head` (Super Admin) and `finance` (Admin), defined in Keycloak.
- Reused HyperTrack conventions: two-file SQL migration, no native SQL, no JPA associations, `AppException` error envelope, `EmailNotificationService`.
- New: `erp.audit_logs` (full action taxonomy), `erp.system_setting`, `erp.feature_flag`, daily `@Scheduled` alert job, Playwright with `data-testid="{module}-{element}"`.

**Modules covered:** Client/Project commercial info, Revenue Recognition (Indian FY), PO Management, Milestone billing flag, Invoice (multi-PO allocation + GST split), Billing Tracker, Collection Tracker (partial payments, reverse, write-off, aging), Alerts (8 types, snooze, manual email), Dashboard (9 KPIs, 3 charts), Export (CSV/Excel/JSON), Recycle Bin, Version History, Settings & Feature Flags, Audit Log.

**Phase 2 (explicitly deferred, not built):** automated email, invoice cancellation, project reopen, CSV import, success KPIs, PDF export.

---

## 9. Trade-offs / limitations (be honest in interviews)

- **Markdown as a database** — simple, diffable, git-friendly, LLM-readable; but no real transactions — relies on re-read-before-write + git merge conflicts.
- **Prompt discipline depends on the model obeying rules** — mitigated by self-verification checklists at the end of every prompt and narrow file ownership.
- **Overhead** — heavy for tiny features; pays off for multi-developer, multi-month work with changing requirements.
- **Tests need conventional locators** — self-healing only works if `data-testid` is followed; otherwise false failures block promotion.
- **Human gates** (CR approval, roster, mode) are intentional bottlenecks — trade speed for control.
