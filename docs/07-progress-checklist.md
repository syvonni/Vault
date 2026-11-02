# Vault POS Implementation Checklist

Review position **07 of 08**. Review task coverage and sparse commit guidance, then the milestone outcomes/gates in document 08. Track both in [GitHub issue #7](https://github.com/syvonni/Vault/issues/7). This remains a draft; implementation is not authorized.

## How Tracking Works

This document keeps the **full roadmap and its checklists as a reference copy**. [GitHub Projects](https://github.com/users/syvonni/projects/5) and its issues hold the active task checklists, progress, and completion evidence. Update checkboxes and statuses there; you do not need to mirror them here. Reference checkmarks below may differ from current GitHub progress. Review issues use compact checkboxes matching document section names and order. Read the document, then record review progress in its issue; focused issues track only their specific task.

Documents 01–06 hold the requirements and design decisions; document 08 owns milestone outcomes and completion gates. Update this roadmap when scope or planned work changes, while keeping day-to-day progress in GitHub. Create future implementation issues from the relevant roadmap tasks when that work approaches; the complete roadmap can stay here without creating every issue now.

## Document Review Order

**Starting point: 01 — Requirements.** See GitHub for the current review status. Review one document at a time; you do not need to finalize all of them at once. Every document is still a draft, and implementation remains unauthorized.

1. [01 — Requirements](01-requirements.md): decide what Vault does, who uses it, the mobile-POS/desktop-back-office split, and what is out of scope.
2. [02 — Feature specifications and workflows](02-feature-specifications.md): walk through what each person does, screen steps, validation, failures, retries, and acceptance examples.
3. [03 — Database design goals](03-database-design-goals.md): agree on integrity, performance, security, maintainability, and backup/recovery expectations.
4. [04 — Tech stack](04-tech-stack.md): settle the remaining technology, authentication, local-running, and delivery choices that support those workflows.
5. [05 — Architecture planning](05-architecture-planning.md): decide how mobile, desktop, API, application services, and PostgreSQL fit together and enforce the rules.
6. [06 — Conceptual schema](06-conceptual-schema.md): review tables, fields, relationships, and constraints against the preceding decisions.
7. [07 — Progress checklist](07-progress-checklist.md): review detailed task coverage and commit guidance.
8. [08 — Milestones](08-milestones.md): review the outcome and completion gate for each stage; this shares review issue #7 with the roadmap.

Keep **07 open as a roadmap reference throughout**, and use GitHub issues to track progress. Its existing task IDs are stable; the filename prefixes give the document review order. If a later review exposes a gap, revisit only the affected earlier decisions.

At each step, resolve the document's open questions, record agreed changes, and explicitly confirm that review before moving on. Draft checkmarks mean a document exists, not that you have approved it. After all planning reviews, coding/setup still requires your explicit authorization.

Start with **01**; use GitHub for the current review position. Runtime commands are added to document 04 only after implementation is authorized and the commands exist.

## What You Are Building

Use [requirements](01-requirements.md) for the purpose, scope, client responsibilities, roles, and exclusions, and [feature specifications](02-feature-specifications.md) for workflow steps and acceptance scenarios. Milestone outcomes and gates live only in [08-milestones.md](08-milestones.md); this document retains the complete task roadmap.

## How to Work Through This Guide

- During planning, follow the [numbered document review order](07-progress-checklist.md#document-review-order), using M0 task IDs as stable references rather than a strict reading sequence. After the planning gate and explicit implementation authorization, work through implementation milestones in order. Complete each GitHub task only with the corresponding reviewed document or implementation/verification evidence.
- Complete the remaining stack decisions in M1-01; Python, Django, PostgreSQL, DRF, and Vue are selected in [04-tech-stack.md](04-tech-stack.md). The working split is Vue desktop back office and React Native mobile POS; Tauri, refund approval/execution direction, device/register direction, and PDF receipt delivery are confirmed; Expo tooling, detailed state models, and mobile OS acceptance remain under review.
- Validate backend flows through SQL/API before their final screens. Drafting the schema does not mean the operations are implemented.
- Privileged actions need atomic audit writing from Milestone 2 onward. Audit viewing/reporting comes in Milestone 3.
- Milestone 2 establishes sale/stock/shift-opening primitives. Milestone 3 adds full refunds and closure; do not claim those workflows work before then.
- When changing scope, update the requirements first, then synchronize the schema and this checklist.

## Git Commit Checkpoints — Keep It Sparse

Commit a coherent result, grouping its related files, documentation, and tests together. Do not commit for every file, checkbox, or wording change. These are suggested checkpoints, not automatic commit/push instructions.

- [ ] **C1 — Scope and workflows settled:** after reviewing requirements and feature workflows (M0-01–M0-05), save the agreed behavior and related documentation together. Example: `docs: finalize POS scope and workflows`.
- [ ] **C2 — Design planning complete:** after the remaining M0 reviews, save the agreed database goals, stack, architecture, schema, and reconciled checklist together. Example: `docs: finalize Vault design plan`. This commit does not itself authorize implementation.
- [ ] **C3 — Database foundation verified:** after the M1 completion gate, save the working setup, migrations, constraints, seeds, and their checks. Example: `feat: establish POS database foundation`.
- [ ] **C4 — Core selling verified:** after the M2 completion gate, save the working identity, inventory, shifts, checkout, retry handling, and correctness tests. Example: `feat: complete safe checkout and stock posting`.
- [ ] **C5 — Returns and oversight verified:** after the M3 completion gate, save full refunds, cash reconciliation, history/reports, and their verification together. Example: `feat: complete returns and operational reporting`.
- [ ] **C6 — Learning-project demonstration complete:** after the M4 completion gate, save the verified clients, performance work, recovery evidence, and operating documentation. Example: `feat: complete Vault learning-project demo`.

Before each commit, review the diff and run the checks relevant to that checkpoint; planning-only changes need document review/link checks, not application tests. If a milestone grows too large, an extra commit for a complete, verified feature or an important fix is reasonable. Keep unfinished experiments out of a commit described as complete.

A **commit** saves a local Git checkpoint; a **push** publishes commits to GitHub. Treat pushing as a separate action. Record C1–C6 completion on the relevant GitHub issue only after the corresponding commit actually exists; the boxes here are reference reminders. Keep the repository README empty.

## Milestone 0 — Feature Specifications, Architecture, and Design Review

- [ ] **M0-01:** Finalize the problem statement and single-project scope, including mobile POS, desktop inventory/admin/reporting/audit workflows, refund handoff, device/register binding, and supported operating systems; list explicit exclusions without promising a later product release.
- [ ] **M0-02:** Review Owner/Manager/Cashier permissions, first-owner setup, staff provisioning, credential delivery/reset, session revocation, and who may see or change each record.
- [ ] **M0-03:** Complete the [feature-specification template and inventory](02-feature-specifications.md) for every feature group, linking all POS-FR requirements to a concrete specification.
- [ ] **M0-04:** Specify actors, preconditions, inputs, screen steps, validation, permissions, state changes, created/changed records, failures/retries, and acceptance examples for each feature.
- [ ] **M0-05:** Review WF-01–WF-09 in the [feature specifications](02-feature-specifications.md#workflow-drafts-by-client), then sketch mobile login/register/shift/cart/payment/receipt/void/refund/closure and desktop setup/catalog/stock/history/report/audit flows. Resolve the listed cross-client questions.
- [ ] **M0-06:** Finish the [architecture plan](05-architecture-planning.md): system context, component responsibilities, desktop/mobile/API/database boundaries, and proposed repository/module organization.
- [ ] **M0-07:** Document checkout/refund/shift-close request flows and state diagrams, lock order, idempotency, failure recovery, and where each validation/authorization rule is enforced.
- [ ] **M0-08:** Review the authentication proposal, including Django-user mapping, hashed per-device tokens, expiration, session history/revocation, password/role changes, secure client storage, and permission checks. Do not implement it.
- [ ] **M0-09:** Review and record the [database goals](03-database-design-goals.md), their measurable evidence, and the proposed backup schedule, retention, recovery-point/recovery-time targets, and outage procedure.
- [ ] **M0-10:** Walk through every DBML entity/relationship, cardinality, nullability, key, normalization choice, money/quantity type, immutable snapshot, and deletion/retention rule against the feature specifications.
- [ ] **M0-11:** Produce the schema data dictionary and invariant-to-enforcement mapping: row checks, foreign keys, uniqueness, posting routines, transactional locks, runtime privileges, and integrity reports.
- [ ] **M0-12:** Review the chosen stack/version baseline and developer tools in [04-tech-stack.md](04-tech-stack.md); apply confirmed Tauri and resolve Expo/auth proposals and local Docker topology, leaving deployment provider undecided if desired.
- [ ] **M0-13:** Specify the verification plan using known fixtures, permission/rollback/concurrency cases, frontend/native/mobile smoke checks, load-test parameters, query plans, and isolated restore evidence.
- [ ] **M0-14:** Plan CI stages, installation/running instructions, desktop/mobile build/distribution/signing expectations, hosting alternatives, and secrets/backups ownership without provisioning anything.
- [ ] **M0-15:** Check traceability from each approved requirement to its specification, schema entities, acceptance scenarios, and implementation task. Resolve contradictory rules and record remaining decisions.
- [ ] **M0-16:** Review the complete documentation with the user. Start implementation only after the planning items are resolved and the user explicitly authorizes code/setup work; the existing do-not-build instruction remains in effect until then.

### Multi-Tenancy Planning Additions

- [ ] **M0-17:** Review limited multi-tenancy scope (POS-FR-30–POS-FR-31): one shared service/database, one store per tenant, one business per staff account, controlled onboarding, and login emails unique across tenants within each installation. Keep public signup, billing, cross-business switching, and platform-admin UI deferred.
- [ ] **M0-18:** Review tenant-aware DBML, per-tenant settings/business uniqueness, same-tenant composite foreign keys, per-tenant Owner/cash-method rules, and the full invariant mapping.
- [ ] **M0-19:** Specify RLS policies/privileges, narrow credential lookup, transaction-local context, request/task/recovery behavior, connection reuse, safe errors, and two-tenant API/database isolation acceptance cases.
- [ ] **M0-20:** Plan both a non-AWS hosted demonstration and a business-local server demonstration, covering deployment/rollback/secrets/health checks, controlled onboarding/audit, installation-specific backup/restore, and per-tenant restore verification. Select the hosted provider and local-server platform later without provisioning now.

- [ ] **M0-21:** Review both deployment topologies, one authoritative database per installation, identical tenant-aware schema/features, independent identities/tokens, and no cloud/local synchronization or automatic failover.
- [ ] **M0-22:** Select a local-server host OS/runtime/resource baseline and plan LAN addressing, certificate trust, firewall/database exposure, installation administrator duties, startup/restart, updates/rollback, and safe client endpoint changes (POS-NFR-10).

### Device/Server Communication and Concurrency Planning

These are future planning tasks, not completed designs or permission to build. Concurrent requests, locks, retry safety, and recovery already appear in M0-07/M0-13, M2-22–M2-26, and the refund/shift race tests in Milestone 3. The tasks below make the communication and synchronization decisions explicit rather than replacing those checks.

- [ ] **M0-23:** ([GitHub issue #11](https://github.com/syvonni/Vault/issues/11)) Plan device-to-server read/write timing and cross-device freshness for both hosted and business-local modes: which actions require server acknowledgment, when stock/prices/receipts/reports refresh, how changes from another device become visible, and how stale/loading/pending/error states are shown. Compare suitable refresh/update mechanisms later; do not select or implement one now. Record the design in documents 02 and 05 and link its acceptance cases to later implementation issues.
- [ ] **M0-24:** ([GitHub issue #12](https://github.com/syvonni/Vault/issues/12)) Review the synchronization boundary explicitly: distinguish clients exchanging data with their selected authoritative API from local-database-to-cloud replication. Confirm whether the current no-cloud/local-sync and no-disconnected-posting scope meets the intended workflows. Record internet-versus-server/LAN outage behavior, uncertain acknowledgments, reconnect/status recovery, and endpoint-change boundaries. A future cloud/local sync proposal must separately review authority, conflict resolution, ordering, duplicate prevention, tenant security, recovery, and scope approval; this task does not add that feature.
- [ ] **M0-25:** ([GitHub issue #13](https://github.com/syvonni/Vault/issues/13)) Consolidate the simultaneous-request/concurrency test matrix across multiple devices/registers: last-unit sales, duplicate submissions/retries, stale cart/catalog edits, checkout-versus-void/refund/shift-close/access changes, stock adjustments, and concurrent refunds. Identify transaction/lock/isolation/retry decisions to review, expected safe outcomes, and links to existing M2/M3 verification tasks. Cover tenant isolation and both deployment modes; create specific implementation/test issues after the plan is approved.

### Pricing and Business Deletion Scope Additions

- [ ] **M0-26:** ([GitHub issue #14](https://github.com/syvonni/Vault/issues/14)) Specify POS-FR-32–POS-FR-35: discount permissions/types/scope, supported promotion eligibility/schedules/stacking, tax-inclusive/exclusive behavior, override authorization/handoff, calculation order/precision/rounding/allocation, zero-total boundaries, immutable snapshots, receipts, full refunds, tax/discount reports, and concurrent changes. Reconcile DBML and acceptance fixtures before approval; retail pricing does not include subscription billing or statutory invoicing/certification.
- [ ] **M0-27:** ([GitHub issue #15](https://github.com/syvonni/Vault/issues/15)) Specify POS-FR-36: Owner verification/confirmation, impact disclosure/export opportunity, pending work/open shifts, cancellation/grace decisions, session revocation, isolated purge/status/retry recovery, retained-evidence policy, backup expiry and restore handling, and hosted/local operator duties. Reconcile retention/FK/schema rules before approval; do not choose a deletion window or implement purge now.

### Confirmed Client Workflow Follow-up

- [ ] **M0-28:** Specify the approved refund direction: cashier mobile request/execution, desktop Manager/Owner request-bound approval, direct Manager/Owner own-shift execution, restricted return lookup, separate identities, expiry/revocation/changed-request invalidation, execution claims, original-paid-value refunds, cash attribution, and concurrency/recovery. Revise DBML and acceptance cases before approval.
- [ ] **M0-29:** Specify logical register selection and one active mobile shift-control session, same-operator replacement-device resume/old-authority invalidation, and related race cases. Specify PDF receipt viewing/download/share and Tauri Windows/macOS packaging acceptance; Tauri itself is confirmed.
- [ ] **M0-30:** Specify reconciliation-required handling when external money movement occurred but posting was rejected: original reference/amount/actors/outcome evidence, Manager/Owner resolution, safe retry/completion/compensation, duplicate-external-action prevention, and cash/shift closure handling. Unknown responses first use existing outcome lookup. Update schema and role/tenant acceptance cases.

## Milestone 1 — Design and Database Foundation

### 1A. Decide and prepare

- [ ] **M1-01:** Complete/document compatible Python/Django/PostgreSQL versions, Django migrations/ORM and driver setup, test runner, and local startup method. Apply the documented planning baseline and selected developer tools only after Milestone 0 and explicit implementation authorization; recheck patch security/compatibility before locking dependencies.
- [ ] **M1-02:** Create directories for application code, migrations, posting SQL/routines, seeds, tests, and operational documentation using the chosen conventions.
- [ ] **M1-03:** Add configuration examples with placeholders; exclude real secrets, local environment files, and generated backups from version control.
- [ ] **M1-04:** Create separate development/test databases and protect reset/fixture commands against non-test targets.
- [ ] **M1-05:** Recheck the approved Milestone 0 sale/refund flows and version-one decisions: one location per business, whole units, PHP, single tender, full refunds, and manual external payment verification.
- [ ] **M1-06:** Render the approved DBML and confirm it matches reviewed feature/state flows for drafts, completed sales, payments, refunds, stock, and shifts. Preserve the reviewed rationale for derived stock and receipt snapshots.

### 1B. Implement schema and constraints

- [ ] **M1-07:** Create migrations for `tenants`, tenant-keyed `store_settings`, `staff_users`, `staff_sessions`, `registers`, and `payment_methods` with normalized staff emails unique across tenants within the installation, tenant-scoped business uniqueness, and session validity checks.
- [ ] **M1-08:** Create `categories`, `products`, and `inventory_movements`; enforce tenant-scoped unique SKU/optional barcode, positive price, whole quantities, and valid movement shape.
- [ ] **M1-09:** Create `operation_requests`, `shifts`, `sales`, `sale_items`, and `payments`, including draft versioning, receipt uniqueness, one product line per sale, cash tender/change checks, and one payment per sale.
- [ ] **M1-10:** Create `refunds`, `refund_items`, and `audit_events`, including one successful full refund per sale and one refund decision per original line.
- [ ] **M1-11:** Add all foreign keys/deletion restrictions, introducing circular request/shift/sale references only after their target tables exist.
- [ ] **M1-12:** Add partial unique indexes for one open shift per tenant/register/operator and exactly one built-in cash-method kind per tenant. Enforce the cash method's immutable/enabled behavior through the controlled configuration path.
- [ ] **M1-13:** Add the DBML local money, timestamp, quantity, sale-state, operation-state, and shift-closure checks, plus candidate lookup/history indexes.
- [ ] **M1-14:** Separate migration-owner and runtime PostgreSQL credentials. Document which invariants need posting routines/privileges/triggers beyond foreign keys and row checks.

### 1C. Seed and verify

- [ ] **M1-15:** For each synthetic business, seed its settings, cash plus example manual noncash methods, two registers, hardware categories, and whole-unit products such as hammers, screws sold by box, and locks.
- [ ] **M1-16:** Provide atomic controlled onboarding of each business and its initial Owner without committing a real password. Seed at least two businesses with separate Owner/Manager/Cashier identities and overlapping SKUs/register codes for isolation tests; no public registration endpoint.
- [ ] **M1-17:** Verify invalid duplicate SKU/barcode/payment/refund/request keys, broken references, invalid prices/quantities, and conflicting open shifts are rejected using isolated fixtures.
- [ ] **M1-18:** Rebuild a blank test database from migrations/seeds and verify all tables in the approved schema and initial records exist (the current 17-table baseline still needs pricing/deletion extensions) without manual fixes. Products must still have zero stock until posting routines exist.

### 1D. Tenant isolation foundation

- [ ] **M1-19:** Create tenant ownership keys on every business table, scoped business identifier indexes, and composite foreign keys preventing cross-tenant relationships. Retain database-wide UUID/email/token uniqueness within each installation as documented.
- [ ] **M1-20:** Apply reviewed RLS policies and least-privilege runtime/credential-lookup identities; verify default-deny missing context, write checks, tenant-registry visibility, and inability to bypass policies.
- [ ] **M1-21:** Build synthetic two-tenant fixtures with overlapping business identifiers and different totals. Prove cross-tenant links fail while matching identifiers in separate tenants succeed.
- [ ] **M1-22:** Verify tenant settings, first Owner, and built-in cash method are provisioned atomically per business through the approved restricted onboarding path with protected operational audit evidence.

- [ ] **M1-23:** After M0 approval and explicit implementation authorization, implement the reviewed pricing/deletion schema changes, snapshots, keys/checks/privileges, and synthetic fixtures. Replace baseline-only pricing checks with the approved model; verify ordinary append-only protections and restricted tenant deletion remain distinct.

## Milestone 2 — Staff Access, Inventory, Shifts, and Checkout

### 2A. Staff and authorized writes

- [ ] **M2-01:** Implement login/logout, secure password hashes, hashed session tokens, session expiry/revocation, own-session history/revocation, and authentication rate limiting.
- [ ] **M2-02:** Implement profile/name/password changes and the server permission rules for Owner, Manager, and Cashier. Verify cashiers cannot change catalog prices, self-authorize checkout price overrides, adjust stock, or change staff roles; and Owner/Manager permissions match the approved role rules.
- [ ] **M2-03:** Implement an atomic append-only audit helper, with safe before/after values, for use by every privileged action and draft void.
- [ ] **M2-04:** Implement owner-provisioned staff, role/access changes with reasons, and session revocation. Block role/access changes for staff with open shifts and protect the last active owner under concurrent changes.
- [ ] **M2-05:** Implement owner settings/register/payment-method management and manager catalog/category management. Preserve referenced records, immutable method kinds, and completed snapshots; prevent register disablement while a shift is open.

### 2B. Request handling and stock

- [ ] **M2-06:** Implement exact PHP/whole-quantity validation, rejecting excess decimals, nonpositive sale prices/quantities, and overflow in line, sale, or cash totals.
- [ ] **M2-07:** Implement durable requester-scoped idempotency claims, canonical immutable payload/fingerprint storage, processing status, safe outcome lookup, and same-key replay/conflict handling across posting action types.
- [ ] **M2-08:** Define the global lock order and recovery/retry policy for authorization, requests, shifts, sales, methods/registers, and product rows. Revalidate eligibility after acquiring relevant locks.
- [ ] **M2-09:** Implement the stock-on-hand query as summed signed movements, with zero for products with no movements. Lock product rows and read fresh stock before every posting.
- [ ] **M2-10:** Implement multi-product stock receipts and signed adjustments, with required delivery notes/reasons, audit events, and atomic operation completion; reject corrections causing negative stock.
- [ ] **M2-11:** Implement product search by SKU/barcode text/name/category, current price/stock, low-stock listing, and scoped inventory-movement history. Keep zero-stock products visible but disable Add; warn without blocking valid low-stock purchases. Limit total per-SKU cart quantity to latest known availability, allow reduction/removal, flag newly insufficient lines, block checkout until explicit correction, and re-enable valid actions after refreshed replenishment. Server posting still rechecks stock under concurrency controls.

### 2C. Open shifts, build carts, and sell

- [ ] **M2-12:** Implement shift opening with nonnegative float, active operator, enabled register, one-open-shift constraints, idempotency, and atomic shift/request creation.
- [ ] **M2-13:** Implement own-shift draft creation, consolidated product lines, quantity edits/removal, calculated totals, and sale locking/version increments. Verify drafts do not reserve/deduct stock.
- [ ] **M2-14:** Implement checkout preview/confirmation with expected cart version/prices and server-computed total; reject changed prices or edited drafts for reconfirmation.
- [ ] **M2-15:** Implement cash tender/change validation and manual noncash confirmation/reference recording for exactly one full payment per sale. Do not call or imply payment-provider processing.
- [ ] **M2-16:** Implement atomic sale completion, receipt/header/item/payment snapshots, one payment, exact per-line deductions, unique receipt number, and successful request finalization.
- [ ] **M2-17:** Implement own-draft void with reason, audit event, and no money/stock effect; completed sales must reject voiding.
- [ ] **M2-18:** Coordinate checkout acceptance with the shift lock and persist target shift/sale references; later close-shift logic must be able to identify unresolved processing work.
- [ ] **M2-19:** Restrict runtime writes to prevent direct edits/deletions of posted stock, completed sale/payment fields, terminal requests, and audit history. Verify enforcement with the actual runtime database role.
- [ ] **M2-20:** Implement rejected/failed outcomes only after rollback, interrupted-request recovery, and unknown-response status lookup. A retry must reuse its key and never ask the operator to collect payment again automatically.

### 2D. Verify the core

- [ ] **M2-21:** Through real posting routines receive 10 hammers, sell two at PHP 150 with PHP 500 tender, and verify stock 8, payment PHP 300, change PHP 200, and float-plus-PHP-300 expected drawer cash.
- [ ] **M2-22:** Race two cashiers at different registers to sell the last unit; assert only one completes, the other records a rejection, and stock ends at zero.
- [ ] **M2-23:** Race identical requests under one key, changed payload under one key, and distinct checkout keys for the same draft; assert at most one sale/payment/deduction set.
- [ ] **M2-24:** Inject failure between payment creation and stock posting; verify no partial sale/payment/movement survives. Test crash-before-posting and lost-response-after-commit recovery.
- [ ] **M2-25:** Race checkout with a price/product/method/access change, draft edit, stock correction, and draft void; verify a valid serial outcome or safe reconfirmation/retry.
- [ ] **M2-26:** Verify two users cannot open the same register concurrently, one user cannot open two shifts, unauthorized users cannot post into another cashier's shift, and role/access changes cannot bypass an open shift.

### 2E. Tenant-aware access and posting

- [ ] **M2-27:** Derive tenant scope from authenticated staff/session, enforce immutable account/business ownership, and install/clear transaction-local context for every request, background job, and recovery action.
- [ ] **M2-28:** Test two-tenant read/write/related-ID/operation-replay isolation, last-Owner/cash-method separation, request-field tampering, missing context, failed transactions, and pooled-connection reuse under the actual runtime role.
- [ ] **M2-29:** Prove tenant B's checkout, stock changes, and idempotency keys cannot affect tenant A; apply tenant scope to optional caches, logs, and safe errors as well as database queries.

- [ ] **M2-30:** Implement and verify approved discount/promotion/tax computation and override authorization with server-calculated previews, stale-approval/rule-change handling, safe retry/concurrency, immutable snapshots, and permission/rounding/limit fixtures.

## Milestone 3 — Refunds, Cash Reconciliation, Reports, and Audit

### 3A. Full refunds

- [ ] **M3-01:** Implement full-refund eligibility and request-specific Manager/Owner approval for cashier execution; record approver separately from current-shift executor. Support direct Manager/Owner own-shift execution, original paid amount/method, approval invalidation, and one successful refund maximum.
- [ ] **M3-02:** Require every original line and one restock decision per line; restore full original quantity only for restocked lines, including inactive products. Require a reason and externally verified noncash refund reference when applicable.
- [ ] **M3-03:** Implement sufficient expected drawer cash checks for cash refunds and no drawer-cash effect for noncash refunds; allow the original shift to be closed.
- [ ] **M3-04:** Commit refund/header/lines/restock entries/audit/successful request together with immutable references and timestamps; keep failed attempts only as operation outcomes.
- [ ] **M3-05:** Race two full refunds under distinct keys and test rollback mid-return; verify one successful refund maximum, no partial restoration, and eligibility for a newly keyed retry after rejection/failure.

### 3B. Close and reconcile shifts

- [ ] **M3-06:** Derive current expected cash as float plus cash sales minus cash refunds in that shift; verify cash change, noncash sales/refunds, stock receipts, and corrections do not inflate it.
- [ ] **M3-07:** Implement own-shift closing with counted cash, computed expected/variance snapshots, required nonzero-variance reason, immutable closure, audit, and atomic request completion.
- [ ] **M3-08:** Reject closure with drafts or other processing checkout/refund requests; serialize request acceptance/draft creation/posting with closure and reject later work against a closed shift.
- [ ] **M3-09:** Test posting-versus-closing races, processing-request recovery before close, nonzero variances, denied reopening/count rewriting, and attempts to disable an occupied register.

### 3C. History, receipts, and SQL reports

- [ ] **M3-10:** Build cashier own-shift and manager/owner store-wide histories with reference/date/operator/register/status/method filters and stable time-plus-ID pagination capped at 100 rows.
- [ ] **M3-11:** Build receipt reprint/download using frozen business/operator/register/item/payment labels and linked refund details; verify later renames/price changes cannot rewrite a receipt.
- [ ] **M3-12:** Implement scoped CSV exports with proper quoting and formula-like text protection; test that receipt/reference searches and exports cannot bypass role visibility.
- [ ] **M3-13:** Create known-result fixtures spanning days/months, empty periods, cash/noncash, varied operators/registers, full refunds in a later month, and mixed restock choices.
- [ ] **M3-14:** Write daily/monthly gross/discount/tax/refund/net-sales and collected-total SQL under the reviewed pricing definitions using posting dates, plus top products and units sold/returned. Do not label net sales as profit or full-business cash flow.
- [ ] **M3-15:** Write payment-method and shift cash-variance reports; avoid multiplying totals when joining sale items, payments, refunds, and movements.
- [ ] **M3-16:** Write current/as-of stock, low-stock, and movement-history SQL, including zero-stock products and Asia/Manila date-boundary tests. Separate returned units from actually restocked units.
- [ ] **M3-17:** Verify reports against manually calculated fixtures, including refunds after the original shift closes, later-month refunds, no-activity periods, and cash tender larger than sale amount.

### 3D. Integrity and audit review

- [ ] **M3-18:** Write integrity queries for sale/line/payment totals, missing/duplicate/wrong product deductions, refund coverage/amount/restock patterns, orphan/inconsistent operation links, and negative stock.
- [ ] **M3-19:** Compare immutable closed-shift cash snapshots with recomputed sale/refund activity and flag mismatches using one consistent database snapshot.
- [ ] **M3-20:** In an isolated disposable database, use controlled privileged corruption fixtures to prove each integrity query detects its target; runtime credentials must still be unable to create these inconsistencies.
- [ ] **M3-21:** Build scoped operation-outcome lookup and read-only audit review. Verify all privileged actions/voids/closures have atomic safe audit events and cashiers cannot self-authorize refunds or other privileged actions; specifically approved refund execution is allowed.

### 3E. Tenant-aware oversight

- [ ] **M3-22:** Verify refunds, receipts/reference searches, reports, CSV exports, integrity queries, audit targets, and outcome lookup cannot expose or modify another business. Compare known totals for two tenants with overlapping identifiers.

- [ ] **M3-23:** Verify full refunds return the original paid amount after discounts/promotions/tax and later rule changes; receipts/exports/reports reconcile base amounts, discounts, tax, final paid totals, and refunded components under the approved definitions.

## Milestone 4 — Frontend, Performance, Recovery, and Delivery

### 4A. Build the operator interface

- [ ] **M4-01:** Implement the approved mobile-POS/desktop-back-office navigation with role-aware screens and authoritative server permissions; use the reviewed platform/framework decisions.
- [ ] **M4-02:** Build appropriate login/profile/session screens in both clients, with Owner staff/settings/register/payment-method administration on desktop.
- [ ] **M4-03:** Build desktop Manager/Owner catalog/category maintenance, stock receipt/adjustment forms, and stock/low-stock/history views.
- [ ] **M4-04:** Build mobile register/shift selection and float entry, then touch-usable POS product lookup/cart/payment/receipt screens, including keyboard/text entry where available, cash tender/change, and manual noncash confirmation/reference.
- [ ] **M4-05:** Preserve one idempotency key per deliberate posting action across network retries, show processing/outcome lookup, and handle reconfirmation without silently changing the cart or requesting a second external payment.
- [ ] **M4-06:** Build mobile receipt/own-sales lookup and owned draft voiding, with desktop store-wide history/export and the reviewed receipt delivery options.
- [ ] **M4-07:** Build cashier mobile refund request/execution and desktop Manager/Owner approval, plus direct Manager/Owner mobile own-shift execution. Include preview/reason/restock/original-method confirmation and distinct approval/execution evidence under the approved state model.
- [ ] **M4-08:** Build mobile operator shift activity/count/close/variance explanation and desktop Manager/Owner closed-shift review. Explain unresolved actions/drafts blocking closure.
- [ ] **M4-09:** Build desktop Manager/Owner sales, refund, stock, cash-variance, integrity, audit, and operation-outcome views with stated date ranges/timezone and correct permissions; expose own-operation retry status in mobile POS too.
- [ ] **M4-10:** Verify mobile touch/text/keyboard input where applicable, desktop keyboard navigation, supported screen sizes, loading/empty/error states, stock/price errors, and unknown-response recovery in both clients. Verify low-stock warnings, zero-stock Add disabling, typed/repeated-add quantity limits, cart invalidation after a stock drop, replenishment re-enabling, and safe server rejection after stale availability.

### 4B. Measure the database

- [ ] **M4-11:** Generate at least 100,000 valid inventory movements with reproducible product/date/sale/refund distributions through validated posting paths or equivalent verified fixture generation; run integrity checks first.
- [ ] **M4-12:** Record hardware, database/software versions, dataset parameters/counts, cache conditions, and baseline history/search/report query plans with execution/buffer statistics.
- [ ] **M4-13:** Run five minutes of filtered sale-history requests with 20 concurrent readers and up to 100 rows per page; record p95 latency against the proposed 500 ms target, throughput, and errors.
- [ ] **M4-14:** Tune SQL/indexes from observed plans, repeat the same benchmark, and save before/after results plus write/storage trade-offs. Verify top-product and cash/stock reports as well.
- [ ] **M4-15:** Decide whether measured results justify Redis. If introduced, verify invalidation and keep stock, idempotency, permissions, and posting correctness in PostgreSQL; otherwise mark caching deferred.

### 4C. Recovery and final demonstration evidence

- [ ] **M4-16:** Document/script backup and restore with role/extension/schema requirements and secret-free configuration; restore into a separate fresh database.
- [ ] **M4-17:** Compare restored row counts, sale/refund totals, stock balances, shift cash snapshots, and integrity results with the source snapshot; record commands and measured recovery time.
- [ ] **M4-18:** Run the complete authorization/integration/concurrency suite from a clean test database and repeat affected benchmarks after any relevant changes.
- [ ] **M4-19:** Demonstrate stock receipt → cashier shift → two-item cash sale with change → receipt/retry → Manager-approved cashier-executed full refund with restock choices → cash reconciliation/closure → report review.
- [ ] **M4-20:** Demonstrate manual noncash recording, insufficient stock rejection, two cashiers racing for the last unit, duplicate checkout prevention, failed posting recovery, and denied unapproved cashier refunds/staff administration.
- [ ] **M4-21:** Update the operating instructions in document 04 with actual prerequisites/setup, migrations/seeds, first-owner bootstrap, app startup, tests, benchmark, demo, and backup/restore commands once implemented. Keep the repository README empty unless the user explicitly changes that preference.
- [ ] **M4-22:** Reproduce the project from a fresh checkout using the operating instructions in document 04. Check each requirements completion criterion with evidence; record any unmet target or deliberately revised scope.

### 4D. Hosted and business-local demonstrations

- [ ] **M4-23:** Deploy the approved service to the selected non-AWS host with protected transport, private PostgreSQL access, secrets, migration/release/rollback procedure, health checks, and logs. Demonstrate independent mobile/desktop sessions for two businesses with visible business identity; public signup/billing remain excluded.
- [ ] **M4-24:** Benchmark with skewed two-tenant data and restore a protected hosted-installation backup into an isolated target. Verify each tenant's counts/totals and RLS privileges before reconnection; document operator-only onboarding/backup access and deployment recovery evidence.

- [ ] **M4-25:** Install the same approved backend/schema on the chosen business-local server with one tenant. Demonstrate desktop/mobile LAN access, visible business identity, persistent storage after server restart, and cash selling with internet disconnected; verify external-provider connectivity is handled separately.
- [ ] **M4-26:** Verify local server/LAN loss blocks posting safely, uncertain outcomes recover against the original server, and client endpoint changes cannot leak tokens/cache or replay pending work to another installation. Record safe handling of active drafts/shifts before switching.
- [ ] **M4-27:** Exercise business-local backup/restore, operator-owned secrets, startup/shutdown, update/rollback, and documented recovery targets. Complete separate hosted/local runbooks with prerequisites and responsibility ownership; keep database/backup administration separate from staff API permissions.

- [ ] **M4-28:** Implement and demonstrate the approved Owner-initiated business deletion workflow in hosted/local modes, including verification/export/confirmation, active-work checks, access revocation, restricted purge/recovery/status, retention/backup/restore evidence, and proof that other tenants remain unchanged.

## Start Here

Start in [01 — Requirements](01-requirements.md), with purpose, scope, client responsibilities, and roles (M0-01–M0-02). Follow the numbered review order while tracking the relevant M0 items in GitHub, then complete the planning gate before any implementation. The next executable setup task after explicit implementation authorization will be M1-01. Your first end-to-end implementation target is a stock receipt and cash sale with correct change and inventory—not a dashboard.
