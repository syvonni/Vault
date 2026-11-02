# Vault POS Implementation Checklist

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Use this document to track the matching M0 tasks while reviewing documents 01–06 in order.
- [ ] Check M0 tasks only when their review evidence exists; keep implementation tasks unchecked during planning.
- [ ] Confirm every requirement has a reviewed specification, schema mapping, acceptance scenario, and implementation task.
- [ ] Reconcile the milestones with the approved mobile-POS/desktop-back-office scope and chosen tools.
- [ ] Resolve remaining planning questions and complete the Milestone 0 review gate.
- [ ] Record explicit implementation authorization only if the user later gives it; completing documentation alone does not authorize building.

Review position **07 of 07**. Use this as a tracker throughout; after reviewing 01–06, reconcile the tasks and confirm the planning gate. See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Complete Milestone 0 first, then follow the four implementation milestones in [01-requirements.md](01-requirements.md), using the [DBML schema](06-conceptual-schema.md). This replaces the simulated-wallet checklist; old task IDs and their meanings are retired. Implementation is not authorized. All implementation tasks remain unchecked; planning checkmarks require reviewed evidence, not just a document existing.

## Document Review Order

**Current step: 01 — Requirements.** Review one document at a time; you do not need to finalize all of them at once. Every document is still a draft, and implementation remains unauthorized.

1. [01 — Requirements](01-requirements.md): decide what Vault does, who uses it, the mobile-POS/desktop-back-office split, and what is out of scope.
2. [02 — Feature specifications and workflows](02-feature-specifications.md): walk through what each person does, screen steps, validation, failures, retries, and acceptance examples.
3. [03 — Database design goals](03-database-design-goals.md): agree on integrity, performance, security, maintainability, and backup/recovery expectations.
4. [04 — Tech stack](04-tech-stack.md): settle the remaining technology, authentication, local-running, and delivery choices that support those workflows.
5. [05 — Architecture planning](05-architecture-planning.md): decide how mobile, desktop, API, application services, and PostgreSQL fit together and enforce the rules.
6. [06 — Conceptual schema](06-conceptual-schema.md): review tables, fields, relationships, and constraints against the preceding decisions.
7. [07 — Progress checklist](07-progress-checklist.md): reconcile the plan with its tasks and confirm the planning gate before any implementation.

Keep **07 open as a tracker throughout**, rather than waiting until the end to use it. Its existing task IDs are stable; the filename prefixes give the document review order. If a later review exposes a gap, revisit only the affected earlier decisions.

At each step, resolve the document's open questions, record agreed changes, and explicitly confirm that review before moving on. Draft checkmarks mean a document exists, not that you have approved it. After all planning reviews, coding/setup still requires your explicit authorization.

For now, open **01** and review only the purpose, scope, client responsibilities, and roles. We can handle its functional requirements next, before moving to **02**. Runtime setup, test, benchmark, and recovery commands will be added only when implementation is authorized and those commands exist.

## What You Are Building

Version one serves a single-location hardware store selling whole-number quantities. The shared retail core is products, stock, sales, recorded payments, internal receipts, full refunds, cashier shifts, and operational reports. Multiple registers may operate concurrently.

The working platform split is mobile POS and desktop Owner/Manager back office; see the [workflow drafts](02-feature-specifications.md#workflow-drafts-by-client). A useful first demonstration is: receive 10 hammers, open a shift, sell two hammers with cash and change, verify stock and expected drawer cash, record a manager-authorized full refund with restock choices, and close the shifts with counted cash and variance.

You are not implementing wallets, bank transfers, payment processing, payroll, supplier debt, full accounting, restaurant kitchens, or profit reporting. Keep future extensions outside the completion gate until the requirements change deliberately.

## How to Work Through This Guide

- During planning, follow the [numbered document review order](07-progress-checklist.md#document-review-order), using M0 task IDs as stable references rather than a strict reading sequence. After the planning gate and explicit implementation authorization, work through implementation milestones in order. Check each task only with the corresponding reviewed document or implementation/verification evidence.
- Complete the remaining stack decisions in M1-01; Python, Django, PostgreSQL, DRF, and Vue are selected in [04-tech-stack.md](04-tech-stack.md). The working split is Vue desktop back office and React Native mobile POS; shell/tooling, detailed workflows, refund handoff, and OS acceptance remain under review.
- Validate backend flows through SQL/API before their final screens. Drafting the schema does not mean the operations are implemented.
- Privileged actions need atomic audit writing from Milestone 2 onward. Audit viewing/reporting comes in Milestone 3.
- Milestone 2 establishes sale/stock/shift-opening primitives. Milestone 3 adds full refunds and closure; do not claim those workflows work before then.
- When changing scope, update the requirements first, then synchronize the schema and this checklist.

Current documentation status (drafted does not mean reviewed or approved):

- [x] POS purpose, scope, roles, requirements, and milestones are documented.
- [x] Proposed POS entities and relationships are documented in DBML.
- [ ] Feature specifications, architecture, database goals, and conceptual schema have completed Milestone 0 review.
- [ ] The user has explicitly authorized implementation after planning review.
- [ ] Runtime application, migrations, tests, and setup commands are implemented.

## Milestone 0 — Feature Specifications, Architecture, and Design Review

**Outcome:** A coherent, reviewed plan for one learning-project deliverable. This milestone produces documentation only. Do not create app scaffolding, environments, dependencies, migration/configuration files, CI workflows, containers, or deployments during planning.

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
- [ ] **M0-12:** Review the chosen stack/version baseline and developer tools in [04-tech-stack.md](04-tech-stack.md); resolve proposals for Tauri/Expo/auth and local Docker topology, leaving deployment provider undecided if desired.
- [ ] **M0-13:** Specify the verification plan using known fixtures, permission/rollback/concurrency cases, frontend/native/mobile smoke checks, load-test parameters, query plans, and isolated restore evidence.
- [ ] **M0-14:** Plan CI stages, installation/running instructions, desktop/mobile build/distribution/signing expectations, hosting alternatives, and secrets/backups ownership without provisioning anything.
- [ ] **M0-15:** Check traceability from each approved requirement to its specification, schema entities, acceptance scenarios, and implementation task. Resolve contradictory rules and record remaining decisions.
- [ ] **M0-16:** Review the complete documentation with the user. Start implementation only after the planning items are resolved and the user explicitly authorizes code/setup work; the existing do-not-build instruction remains in effect until then.

**Planning completion gate:** Feature specifications, screen flows, architecture, database goals, and conceptual schema are reviewed together; outstanding scope/design decisions are resolved; the user explicitly authorizes implementation. A generated document or valid DBML syntax alone cannot satisfy this gate. The current documents are drafts, so this gate is open.

## Milestone 1 — Design and Database Foundation

**Outcome:** A clean database can be created with the POS schema, initial configuration, staff roles, and synthetic reference data.

**Supports:** POS-FR-01–POS-FR-29 foundations; POS-NFR-01, POS-NFR-03.

### 1A. Decide and prepare

- [ ] **M1-01:** Complete/document compatible Python/Django/PostgreSQL versions, Django migrations/ORM and driver setup, test runner, and local startup method. Apply the documented planning baseline and selected developer tools only after Milestone 0 and explicit implementation authorization; recheck patch security/compatibility before locking dependencies.
- [ ] **M1-02:** Create directories for application code, migrations, posting SQL/routines, seeds, tests, and operational documentation using the chosen conventions.
- [ ] **M1-03:** Add configuration examples with placeholders; exclude real secrets, local environment files, and generated backups from version control.
- [ ] **M1-04:** Create separate development/test databases and protect reset/fixture commands against non-test targets.
- [ ] **M1-05:** Recheck the approved Milestone 0 sale/refund flows and version-one decisions: one location, whole units, PHP, single tender, full refunds, and manual external payment verification.
- [ ] **M1-06:** Render the approved DBML and confirm it matches reviewed feature/state flows for drafts, completed sales, payments, refunds, stock, and shifts. Preserve the reviewed rationale for derived stock and receipt snapshots.

### 1B. Implement schema and constraints

- [ ] **M1-07:** Create migrations for `store_settings`, `staff_users`, `staff_sessions`, `registers`, and `payment_methods` with normalized staff-email uniqueness and session validity checks.
- [ ] **M1-08:** Create `categories`, `products`, and `inventory_movements`; enforce unique SKU/optional barcode, positive price, whole quantities, and valid movement shape.
- [ ] **M1-09:** Create `operation_requests`, `shifts`, `sales`, `sale_items`, and `payments`, including draft versioning, receipt uniqueness, one product line per sale, cash tender/change checks, and one payment per sale.
- [ ] **M1-10:** Create `refunds`, `refund_items`, and `audit_events`, including one successful full refund per sale and one refund decision per original line.
- [ ] **M1-11:** Add all foreign keys/deletion restrictions, introducing circular request/shift/sale references only after their target tables exist.
- [ ] **M1-12:** Add partial unique indexes for one open shift per register/operator and exactly one built-in cash-method kind. Enforce the cash method's immutable/enabled behavior through the controlled configuration path.
- [ ] **M1-13:** Add the DBML local money, timestamp, quantity, sale-state, operation-state, and shift-closure checks, plus candidate lookup/history indexes.
- [ ] **M1-14:** Separate migration-owner and runtime PostgreSQL credentials. Document which invariants need posting routines/privileges/triggers beyond foreign keys and row checks.

### 1C. Seed and verify

- [ ] **M1-15:** Seed the singleton store settings, cash plus example manual noncash methods, two registers, hardware categories, and whole-unit products such as hammers, screws sold by box, and locks.
- [ ] **M1-16:** Provide controlled initial-owner bootstrap without committing a real password. Seed synthetic Owner, Manager, and two Cashier identities for tests; no public registration endpoint.
- [ ] **M1-17:** Verify invalid duplicate SKU/barcode/payment/refund/request keys, broken references, invalid prices/quantities, and conflicting open shifts are rejected using isolated fixtures.
- [ ] **M1-18:** Rebuild a blank test database from migrations/seeds and verify all 16 proposed tables and initial records exist without manual fixes. Products must still have zero stock until posting routines exist.

**Completion gate:** Repeatable setup creates the POS database, constraints demonstrably reject invalid rows, and schema relationships match the documentation. No selling workflow is marked complete yet.

## Milestone 2 — Staff Access, Inventory, Shifts, and Checkout

**Outcome:** A cashier can complete cash/manual noncash sales without duplicate posting or overselling; managers can maintain the catalog and receive/correct stock.

**Requirements:** POS-FR-01–POS-FR-17, POS-FR-21, POS-FR-22, POS-FR-28, POS-FR-29; POS-NFR-01–POS-NFR-03, POS-NFR-05.

### 2A. Staff and authorized writes

- [ ] **M2-01:** Implement login/logout, secure password hashes, hashed session tokens, session expiry/revocation, own-session history/revocation, and authentication rate limiting.
- [ ] **M2-02:** Implement profile/name/password changes and the server permission rules for Owner, Manager, and Cashier. Verify cashiers cannot change prices/stock/staff roles, and Owner/Manager permissions match the approved role rules.
- [ ] **M2-03:** Implement an atomic append-only audit helper, with safe before/after values, for use by every privileged action and draft void.
- [ ] **M2-04:** Implement owner-provisioned staff, role/access changes with reasons, and session revocation. Block role/access changes for staff with open shifts and protect the last active owner under concurrent changes.
- [ ] **M2-05:** Implement owner settings/register/payment-method management and manager catalog/category management. Preserve referenced records, immutable method kinds, and completed snapshots; prevent register disablement while a shift is open.

### 2B. Request handling and stock

- [ ] **M2-06:** Implement exact PHP/whole-quantity validation, rejecting excess decimals, nonpositive sale prices/quantities, and overflow in line, sale, or cash totals.
- [ ] **M2-07:** Implement durable requester-scoped idempotency claims, canonical immutable payload/fingerprint storage, processing status, safe outcome lookup, and same-key replay/conflict handling across posting action types.
- [ ] **M2-08:** Define the global lock order and recovery/retry policy for authorization, requests, shifts, sales, methods/registers, and product rows. Revalidate eligibility after acquiring relevant locks.
- [ ] **M2-09:** Implement the stock-on-hand query as summed signed movements, with zero for products with no movements. Lock product rows and read fresh stock before every posting.
- [ ] **M2-10:** Implement multi-product stock receipts and signed adjustments, with required delivery notes/reasons, audit events, and atomic operation completion; reject corrections causing negative stock.
- [ ] **M2-11:** Implement product search by SKU/barcode text/name/category, current price/stock, low-stock listing, and scoped inventory-movement history.

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

**Completion gate:** Backend tests demonstrate valid cash/noncash selling, accurate stock, immutable receipts, authorized catalog/stock actions, idempotency, rollback, and last-item concurrency protection. Refund and close-shift completion are still pending Milestone 3.

## Milestone 3 — Refunds, Cash Reconciliation, Reports, and Audit

**Outcome:** Staff can handle returns and finish shifts; owners can inspect operational reports and detect inconsistent records.

**Requirements:** POS-FR-16, POS-FR-18–POS-FR-20, POS-FR-22–POS-FR-29; POS-NFR-01–POS-NFR-03, POS-NFR-05.

### 3A. Full refunds

- [ ] **M3-01:** Implement manager/owner refund eligibility: completed original sale, no prior successful refund, authorized current-shift operator, full original amount, and original payment method.
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
- [ ] **M3-14:** Write daily/monthly gross/refund/net-sales SQL using posting dates, plus top products and units sold/returned. Do not label net sales as profit or full-business cash flow.
- [ ] **M3-15:** Write payment-method and shift cash-variance reports; avoid multiplying totals when joining sale items, payments, refunds, and movements.
- [ ] **M3-16:** Write current/as-of stock, low-stock, and movement-history SQL, including zero-stock products and Asia/Manila date-boundary tests. Separate returned units from actually restocked units.
- [ ] **M3-17:** Verify reports against manually calculated fixtures, including refunds after the original shift closes, later-month refunds, no-activity periods, and cash tender larger than sale amount.

### 3D. Integrity and audit review

- [ ] **M3-18:** Write integrity queries for sale/line/payment totals, missing/duplicate/wrong product deductions, refund coverage/amount/restock patterns, orphan/inconsistent operation links, and negative stock.
- [ ] **M3-19:** Compare immutable closed-shift cash snapshots with recomputed sale/refund activity and flag mismatches using one consistent database snapshot.
- [ ] **M3-20:** In an isolated disposable database, use controlled privileged corruption fixtures to prove each integrity query detects its target; runtime credentials must still be unable to create these inconsistencies.
- [ ] **M3-21:** Build scoped operation-outcome lookup and read-only audit review. Verify all privileged actions/voids/closures have atomic safe audit events and cashiers cannot invoke privileged write actions.

**Completion gate:** Full refunds, restock choices, cash closure, receipt/history visibility, reports, and integrity queries match known fixtures and pass concurrency/rollback tests. Original sales/shifts remain unchanged by later returns.

## Milestone 4 — Frontend, Performance, Recovery, and Delivery

**Outcome:** A usable POS demonstrates the verified workflows, handles retries clearly, meets a measured performance target, and can be restored/reproduced.

**Requirements:** All POS functional requirements in the UI; POS-NFR-04, POS-NFR-06–POS-NFR-08 and final regression checks.

### 4A. Build the operator interface

- [ ] **M4-01:** Implement the approved mobile-POS/desktop-back-office navigation with role-aware screens and authoritative server permissions; use the reviewed platform/framework decisions.
- [ ] **M4-02:** Build appropriate login/profile/session screens in both clients, with Owner staff/settings/register/payment-method administration on desktop.
- [ ] **M4-03:** Build desktop Manager/Owner catalog/category maintenance, stock receipt/adjustment forms, and stock/low-stock/history views.
- [ ] **M4-04:** Build mobile register/shift selection and float entry, then touch-usable POS product lookup/cart/payment/receipt screens, including keyboard/text entry where available, cash tender/change, and manual noncash confirmation/reference.
- [ ] **M4-05:** Preserve one idempotency key per deliberate posting action across network retries, show processing/outcome lookup, and handle reconfirmation without silently changing the cart or requesting a second external payment.
- [ ] **M4-06:** Build mobile receipt/own-sales lookup and owned draft voiding, with desktop store-wide history/export and the reviewed receipt delivery options.
- [ ] **M4-07:** Build the reviewed Manager/Owner full-refund workflow with preview/reason/restock decisions and original-method confirmation. Keep current own-shift rules unless M0 explicitly approves a different handoff/schema.
- [ ] **M4-08:** Build mobile operator shift activity/count/close/variance explanation and desktop Manager/Owner closed-shift review. Explain unresolved actions/drafts blocking closure.
- [ ] **M4-09:** Build desktop Manager/Owner sales, refund, stock, cash-variance, integrity, audit, and operation-outcome views with stated date ranges/timezone and correct permissions; expose own-operation retry status in mobile POS too.
- [ ] **M4-10:** Verify mobile touch/text/keyboard input where applicable, desktop keyboard navigation, supported screen sizes, loading/empty/error states, stock/price errors, and unknown-response recovery in both clients.

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
- [ ] **M4-19:** Demonstrate stock receipt → cashier shift → two-item cash sale with change → receipt/retry → manager-shift full refund with restock choices → cash reconciliation/closure → report review.
- [ ] **M4-20:** Demonstrate manual noncash recording, insufficient stock rejection, two cashiers racing for the last unit, duplicate checkout prevention, failed posting recovery, and denied cashier refunds/staff administration.
- [ ] **M4-21:** Update the operating instructions in document 04 with actual prerequisites/setup, migrations/seeds, first-owner bootstrap, app startup, tests, benchmark, demo, and backup/restore commands once implemented. Keep the repository README empty unless the user explicitly changes that preference.
- [ ] **M4-22:** Reproduce the project from a fresh checkout using the operating instructions in document 04. Check each requirements completion criterion with evidence; record any unmet target or deliberately revised scope.

**Completion gate:** Required role flows work through the frontend, correctness/concurrency checks pass, benchmark results meet the agreed target or document an explicitly revised target, restore verification succeeds, and a fresh checkout reproduces the demo.

## Start Here

Start in [01 — Requirements](01-requirements.md), with purpose, scope, client responsibilities, and roles (M0-01–M0-02). Follow the numbered review order while tracking the relevant M0 items here, then complete the planning gate before any implementation. The next executable setup task after explicit implementation authorization will be M1-01. Your first end-to-end implementation target is a stock receipt and cash sale with correct change and inventory—not a dashboard.

## Deferred Work

Restaurant tables/modifiers/kitchen/ingredient workflows; fractional units/package conversion; integrated payments; split tender/credit/partial payments; partial refunds/exchanges; suppliers/purchasing/payables; expenses/payroll/full accounting/profit; tax/statutory invoicing; offline sync; multiple locations/businesses; and emergency shift takeover remain outside this learning project.
