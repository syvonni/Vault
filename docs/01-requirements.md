# Vault POS and Inventory System Requirements

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

* [x] Confirm the business problem, target store, and purpose of Vault.

* [ ] Confirm the single-project scope and explicitly excluded features.

* [ ] Confirm mobile POS versus desktop inventory/admin/reporting/audit responsibilities.

* [ ] Confirm Owner, Manager, and Cashier permissions and first-owner/staff account creation.

* [ ] Review each functional-requirement module and clarify any unfamiliar or ambiguous behavior.

* [ ] Resolve the refund handoff, mobile/register assignment, receipt delivery, and external-payment failure rules with document 02.

* [ ] Confirm the business rules, non-functional targets, and project completion criteria.

* [ ] Record the agreed changes and confirm this requirements review is complete before moving to 02.

Review position **01 of 07**. Agree on purpose, scope, mobile/desktop responsibilities, roles, and required behavior. See the [review order](../README.md#review-order--start-here). This document remains a draft; numbering does not mark it approved.

## Purpose and Business Problem

Vault is a point-of-sale and inventory system for a small, single-location hardware store. It helps staff sell products, record payments, maintain stock counts, issue internal sales receipts, process full-sale refunds, and reconcile cashier shifts.

A store needs to answer: What sold today? What is left on the shelf? Can two cashiers sell the same last item? Was a sale recorded twice after a retry? Does the counted cash match what the shift should contain? Who changed a price, adjusted stock, or authorized a refund?

Vault supports these daily operations. It records payments received outside the software; it does not move money through banks or payment networks.

This revision replaces the previous simulated-wallet requirements. All `POS-FR-*` identifiers are new and supersede the old `FR-*` identifiers; they are not a renumbering of the old features.

## Single-Project Scope and Assumptions

* **Business:** One business at one physical location per installation. Multiple registers and concurrent cashiers are supported. Multi-business hosting, branches, warehouses, and inter-location transfers are deferred.

* **Products:** Each sellable product has a unique SKU, a category, a name, a package/unit label, a selling price, a low-stock threshold, and active/inactive status. Each SKU is sold and stocked in positive whole quantities. A box and an individual item are separate SKUs; automatic package conversion and fractional measurements are deferred.

* **Catalog flexibility:** Categories, unit labels, products, registers, and manual noncash payment methods are configurable records. The initial seed catalog is hardware-specific, while the sales, payment, inventory, and shift model uses general retail concepts. Future business modules require deliberate extensions rather than arbitrary custom fields.

* **Currency/pricing:** PHP only, at most two decimal places, fixed unit prices, and no discounts, tax computation, promotions, or price overrides at checkout initially. Printed/downloaded receipts are internal sales records; statutory invoicing/certification is outside this project scope.

* **Stock:** On-hand quantity derives from an append-only inventory-movement ledger. No negative stock, reservations, backorders, recipes, batches, serial tracking, or product bundles initially. Draft carts do not reserve stock.

* **Checkout:** One complete payment per sale. Cash tender and change are supported; manual noncash payments record confirmation/reference information. Split tender, credit sales, customer invoices, partial payments, and integrated payment terminals are deferred.

* **Refunds:** A manager/owner can record one full refund per completed sale, using the original payment method. Each returned line is marked restock or do-not-restock. Partial refunds, exchanges, and refund cancellation are deferred.

* **Cash shifts:** One cashier and one register per shift; neither may have multiple open shifts. A shift records opening float, activity, counted closing cash, and variance. Other cash-in/cash-out events, petty cash, and bank deposits are deferred.

* **Connectivity:** The learning project requires a reachable central backend/database. Offline synchronization is deferred.

* **Finance boundary:** Track sales, refunds, payments, and drawer cash. Payroll, supplier payables, purchase orders, expenses, general-ledger accounting, inventory valuation, cost of goods sold, and profit reporting are deferred. Stock receipts record quantities and a delivery note/reference, not supplier debt or purchase accounting.

## Client Responsibilities — Working Workflow Split

The current planning direction is **mobile POS plus desktop back office**, both using the same Django REST API/PostgreSQL database. This replaces the earlier desktop-POS/mobile-manager-companion proposal. Detailed workflow drafts and open questions live in [02-feature-specifications.md](02-feature-specifications.md#workflow-drafts-by-client).

* **Mobile (React Native):** cashier login, register/shift opening, product lookup, cart, cash/manual-noncash checkout, receipt viewing, own sales/shift activity, draft void, and shift cash count/closure. Manager/Owner refund execution is a draft mobile workflow under the existing role/own-shift rules; its exact handoff needs review.

* **Desktop (Vue, shell undecided):** Owner/Manager business/staff/register/payment setup, product/category/price maintenance, stock receipt/adjustment, movement/low-stock review, sales and shift history, operational metrics/reports/exports, integrity checks, and audit-log review.

* **Shared behavior:** individual staff identity, current permissions, authoritative product/stock/payment outcomes, and retry/status handling are server responsibilities. A mobile client does not grant a cashier Manager permissions; a platform split does not create separate inventories or user tables.

No desktop checkout or read-only mobile-companion requirement is assumed. Resolve refund authorization flow, mobile device/register assignment, receipt delivery, and noncash rejection handling in Milestone 0 before approving the client workflows or schema. Functional requirement IDs below remain unchanged because they primarily define role/business behavior, not screen location.

## Objectives

1. Provide a useful checkout and stock-control workflow for the first target business.
2. Separate reusable retail entities from future restaurant-specific workflows.
3. Use normalized PostgreSQL data, exact monetary calculations, and transactional posting.
4. Prevent duplicate sales, concurrent overselling, duplicate refunds, and shift/posting races.
5. Preserve price snapshots, stock history, authorization, and audit evidence.
6. Practice advanced SQL, indexing, performance measurement, and backup recovery against a concrete business workflow.

## Users and Permissions

* **Owner:** All manager/cashier capabilities plus staff provisioning, role changes, business settings, and register/payment-method configuration. The last active owner cannot be deactivated or demoted.

* **Manager:** Product/category management, stock receipts/adjustments, full refunds, store reports, and audit review. May operate checkout when using a shift opened under their own identity.

* **Cashier:** Manage their own draft sales, operate their own register shift, record checkout, reprint accessible receipts, and view their own shift activity. Cannot change catalog prices, adjust stock, refund sales, manage staff, or read store-wide reports/audit logs.

The only staff roles are Owner, Manager, and Cashier. Each staff user has one role. Owner/manager roles include cashier capabilities, but an open shift must belong to the actual operator. There is no public staff registration. First-owner creation is a controlled bootstrap operation.

## Functional Requirements

### Module A — Staff and Store Setup

* **POS-FR-01 — Authentication:** Active staff shall log in/out securely, view their sessions, and revoke their own other sessions. Expired, revoked, or deactivated-user sessions shall be denied.

* **POS-FR-02 — Staff administration:** Owners shall create staff accounts, set roles, deactivate/reactivate staff, and revoke their sessions. Role/access changes require a reason, revoke affected sessions, preserve attribution, and cannot remove the last active owner. Initial credential delivery/reset is an owner-controlled setup procedure; public password recovery is deferred.

* **POS-FR-03 — Profile:** Staff shall view their profile, change their display name, and change their password after verifying the current password.

* **POS-FR-04 — Configuration:** Owners shall maintain the business display name, registers, and manual noncash payment-method names. Cash is a built-in method. Disabling a register/method prevents new use, preserves history, and does not prevent a refund using the original method. Registers cannot be disabled while their shift is open.

### Module B — Product Catalog and Inventory

* **POS-FR-05 — Catalog maintenance:** Managers/owners shall create and update categories and product names, unique SKUs, optional unique barcodes, unit labels, selling prices, and low-stock thresholds, and activate/deactivate categories/products. Disabled categories shall prevent new product assignment; existing products shall retain their category and follow their own active status. Historical sale snapshots shall remain unchanged.

* **POS-FR-06 — Product lookup:** Cashiers shall search active products by SKU, barcode text, name, or category and see current price and on-hand quantity. Keyboard entry is sufficient initially; scanner/printer hardware integration is deferred.

* **POS-FR-07 — Stock receipts:** Managers/owners shall record positive whole-number stock receipts for one or more products with a delivery reference/note. Submission shall append receipt movements atomically and shall not create purchase/payable records.

* **POS-FR-08 — Stock adjustments:** Managers/owners shall record signed whole-number stock corrections with a required reason. An adjustment must not make stock negative and cannot rewrite prior movements.

* **POS-FR-09 — Stock visibility:** Authorized staff shall view current on-hand stock. Managers/owners shall view movement history and active products at or below their low-stock thresholds. Products start at zero until a receipt/adjustment is posted.

### Module C — Draft Sales and Checkout

* **POS-FR-10 — Draft sale:** Cashiers shall create a draft under their open shift, add/update/remove product lines, and see quantity, unit price, line total, and sale total. Duplicate selections of a SKU shall consolidate to one line. Draft creation/editing shall not post payment or stock movement.

* **POS-FR-11 — Server validation:** Checkout shall validate active staff, ownership of the draft/open shift, enabled register/payment method, active products, positive whole quantities, current prices, and sufficient stock. The server shall calculate totals. A changed price/cart version shall require review and reconfirmation rather than silently charging a different amount.

* **POS-FR-12 — Cash checkout:** A cashier shall record cash tender at least equal to the total, calculate change, and apply exactly the sale total as the payment. Expected drawer cash increases by the sale total, not the tender before change.

* **POS-FR-13 — Manual noncash checkout:** A cashier shall select an enabled noncash method and record its external confirmation/reference after verifying payment outside Vault. Vault shall record the exact sale amount and confirmation; it shall not initiate or verify provider settlement automatically.

* **POS-FR-14 — Atomic completion:** Checkout shall atomically complete the sale, freeze its item/price/name snapshots, create exactly one payment, append the stock deductions, assign a unique immutable receipt number, and finalize the operation outcome. A failed attempt shall leave the draft without payment or deductions.

* **POS-FR-15 — Retry safety:** Checkout and all other posting actions shall use staff-scoped idempotency keys and request fingerprints. The same key/payload shall return the existing operation/outcome; a different payload under that key shall conflict. Concurrent checkout attempts on one draft shall produce at most one completed sale/payment even with different keys.

* **POS-FR-16 — Receipt/history:** A cashier shall view/search receipts for their own shifts, including closed shifts, and reprint/download them. Managers/owners shall access store-wide history. Completed receipts shall show business identity, receipt number, completion time, operator/register, item snapshots, quantities/prices/totals, payment method snapshot, cash tender/change when applicable, and any linked refund.

* **POS-FR-17 — Draft void:** A cashier shall void an owned draft with a reason, retaining its lines/history without posting stock/payment. A completed sale cannot be voided; correction requires a refund. Checkout and void shall serialize so only one succeeds.

### Module D — Full Refunds and Stock Returns

* **POS-FR-18 — Full-sale refund:** Managers/owners shall refund a completed sale once, under their own current open shift, recording reason, actor, time, original payment method, original sale total, and external refund reference for noncash methods. They shall verify the physical/provider refund externally; Vault records the event. A prior closed original shift does not prevent refunding in a new shift.

* **POS-FR-19 — Restock decision:** A full refund shall include every original sale line and a restock decision per line. A restocked line restores its original sold quantity; a damaged/nonreturned line restores none. Inactive products may be restocked through a valid refund.

* **POS-FR-20 — Atomic refund:** Refund completion, unique refund reference, refund-line decisions, stock-restoration entries, audit event, and operation completion shall commit together. No second completed refund, over-refund, or completed-refund edit/delete shall be allowed. Failed attempts shall remain retryable with a deliberately new key.

### Module E — Register Shifts and Cash Reconciliation

* **POS-FR-21 — Open shift:** A cashier shall open an enabled register with a nonnegative opening cash float. A register and a staff member may each have at most one open shift. Opening shall not count as sales revenue.

* **POS-FR-22 — Shift activity:** A cashier shall view sales, refunds, and payment-method totals for their own shifts. Noncash activity shall remain separate from expected physical drawer cash.

* **POS-FR-23 — Close shift:** The shift operator shall close their open shift with counted cash. The system shall freeze expected cash, counted cash, variance, and closing time; nonzero variance requires an explanation. Closing shall reject unresolved processing checkout/refund operations and uncompleted draft sales; cashiers must complete or void drafts first. Subsequent operations cannot post into a closed shift.

* **POS-FR-24 — Shift review:** Managers/owners shall inspect closed-shift totals and variances. Initial shifts cannot be reopened or have their closing counts rewritten. Cash refunds require sufficient expected cash in the current drawer; noncash refunds do not reduce drawer cash.

### Module F — Reports, Integrity, and Audit

* **POS-FR-25 — Sales reports:** Managers/owners shall view daily/monthly gross completed sales, refunds by refund date, net sales, units sold/returned, payment-method totals, and top products by sales amount/quantity. These are sales measures, not profit or complete business cash flow.

* **POS-FR-26 — History and export:** Accessible sale/refund/shift histories shall support reference, date, operator, register, and relevant status/method filters with stable pagination up to 100 rows. Authorized staff shall export the same accessible filtered data as CSV with safe formula-like text handling.

* **POS-FR-27 — Integrity report:** Managers/owners shall inspect mismatched sale totals, missing/duplicate payments, missing/incorrect sale or refund stock entries, invalid refund totals/lines, negative stock, and shift expected-cash mismatches. Reports shall use a consistent database snapshot.

* **POS-FR-28 — Audit trail:** Privileged actions and draft voids shall append actor/action/target/time/reason and safe before/after values. Covered changes include staff access/roles, settings, catalogs, stock receipts/adjustments, refunds, and shift closure/variance. Events shall commit atomically with the action and shall exclude credentials/session tokens.

* **POS-FR-29 — Posting outcomes:** Authorized staff shall inspect their operation status and safe rejection/failure reasons; managers/owners may inspect store-wide outcomes. Rejected business requests and failed execution attempts shall have no committed business postings. Interrupted/unknown responses shall be resolved by checking existing state rather than repeating collection/refund externally.

## Business Rules and Reporting Definitions

1. Completed sale total equals the sum of positive integer quantities multiplied by exact PHP unit prices. Zero-priced products/sales are deferred. Sale lines snapshot SKU, name, unit label, and price at confirmation; completed snapshots and payment-method labels remain immutable.
2. Current stock is the sum of committed signed inventory movements. Receipts/restocked refunds add units; completed sales subtract units; adjustments may add/subtract units. Drafts/voids/rejected/failed operations do not change stock. Every SKU must remain nonnegative.
3. Checkout, stock corrections/receipts, and refunds lock all affected product rows in a consistent order and recheck stock under that lock. Draft edits also use sale/version checks. A database row check alone cannot prevent concurrent overselling.
4. Refund amount equals the original sale total; no new pricing applies. Each original sale has at most one completed refund. Refund eligibility is independent of current product/payment-method enablement, but the operator/current register shift must be valid.
5. Expected drawer cash = opening float + completed cash sale amounts in that shift − completed cash refunds recorded in that shift. Counted cash and opening float must be nonnegative. Variance = counted − expected. Noncash activity, stock receipts, and stock corrections do not enter that formula.
6. Sale completion and refund completion use their actual posting timestamps, not draft creation time. Refunds reduce net sales on their refund date without rewriting prior periods. Gross sales still includes subsequently refunded original sales.
7. Net sales = gross completed sale totals − completed refund totals in the period. Units sold and units returned are reported separately; restocked quantity is a separate stock measure. There is no profit/margin report without cost accounting.
8. Store/report currency is PHP and initial reporting timezone is Asia/Manila. Persist timezone-aware instants; convert inclusive calendar-date filters to half-open timestamp intervals. Months/days without activity appear as zero where relevant.
9. All posting actions (checkout, refund, receipt, adjustment, shift opening/closure, draft void) require a staff-scoped idempotency key across action types. Persist the immutable validated payload/fingerprint and a processing outcome before execution. Complete business effects and the terminal outcome atomically; on business rejection/execution failure retain the outcome without effects. Terminal outcomes do not transition again. A new attempt after a rejection/failure needs a new key.
10. A crash may leave a processing request; recovery must inspect committed state before resolving it. Posting uses a locked processing row. A timeout is not proof that payment/refund was unrecorded. Idempotency protects Vault records; manual external payment/refund handling requires operator verification and cannot promise provider-side exactly-once behavior.
11. Sale completion/refund, shift close, role deactivation, product changes, and register/method changes must coordinate locks so eligibility cannot change unnoticed midway. A processing checkout/refund operation accepted for a shift prevents closure until resolved. Lock order and bounded deadlock/serialization retries must be specified during implementation.
12. Deactivated staff cannot authenticate/start actions; existing attribution remains. Role changes revoke sessions and take effect on future actions. Before deactivating or changing the role of a staff member with an open shift, the owner must have that operator close it; emergency manager takeover is deferred.
13. Referenced products, categories, methods, registers, staff, sales, payments, refunds, shifts, inventory movements, and audit events are retained. Ordinary application credentials cannot edit/delete posted movements, completed documents, payments, refunds, or audit history.

## Database Design Goals

Review [03-database-design-goals.md](03-database-design-goals.md) before approving the schema. Goals cover correctness/integrity, atomicity/isolation/durable outcomes, controlled scalability/flexibility, measured performance, maintainability/reproducibility, security/least privilege, backup/recovery/contingency planning, and traceability/privacy. Each goal names future verification evidence; proposed RPO/RTO and retention targets remain subject to planning review.

## Non-Functional Requirements

* **POS-NFR-01 — Integrity:** Use foreign keys, unique keys, local checks, and controlled posting routines. Enforce cross-row sale/payment/stock/refund invariants and append-only history using transactional logic and database privileges/triggers as appropriate.

* **POS-NFR-02 — Atomicity/concurrency:** Integration tests shall prove no partial checkout/refund survives errors; no stock overselling, duplicate sale/refund, last-owner removal, or posting after shift close can occur under competing requests.

* **POS-NFR-03 — Precision/security:** Use `numeric(20,2)` and decimal/integer-centavo arithmetic, reject excess precision/overflow, hash passwords and session tokens securely, enforce server permissions, rate-limit authentication, and never log secrets. Public-network deployments must use secure transport.

* **POS-NFR-04 — Performance:** Proposed target: p95 server latency at most 500 ms for 100-row filtered sale-history requests over five minutes with 20 concurrent readers and at least 100,000 inventory movements. Document hardware, dataset distribution, query plans, cache conditions, throughput/errors, and whether the target was met.

* **POS-NFR-05 — Testability:** Use synthetic fixtures with known totals for selling, stock, returns, cash change, reconciliation, permissions, operation recovery, and concurrency. A diagram alone does not satisfy correctness.

* **POS-NFR-06 — Recovery:** Restore a backup into a separate database and verify row counts, completed sale/refund totals, stock balances, shift cash totals, and integrity reports. Document procedure and measured time.

* **POS-NFR-07 — Usability:** Checkout shall support keyboard-driven product lookup and clear quantity/price/payment errors. Unknown-response states must offer status lookup/retry of the same request without prompting another external payment.

* **POS-NFR-08 — Caching:** Redis is optional after measurement; authoritative stock, operation keys, permissions, and checkout/refund correctness stay in PostgreSQL.

## Planning Gate and Project Stages

This is one learning-project deliverable. Milestones describe the order of work, not a roadmap for additional product releases. No application code, environment setup, dependency installation, migrations, or deployment begins until the Milestone 0 planning gate in [07-progress-checklist.md](07-progress-checklist.md) is complete and the user explicitly authorizes implementation. Draft documents and a passing DBML parser are not approval.

Before implementation, complete [feature specifications](02-feature-specifications.md), [architecture planning](05-architecture-planning.md), the [database design goals](03-database-design-goals.md) review, and the conceptual schema review. Proposed desktop/mobile scope and authentication choices must be resolved there.

## Delivery Milestones

1. **Design and foundation:** Finalize retail workflows and the [DBML schema](06-conceptual-schema.md); implement PostgreSQL migrations, permissions, catalog/setup seeds, and baseline constraints.
2. **Core selling and stock:** Implement staff access, catalog, stock receipts/adjustments, shift opening, drafts, cash/noncash checkout, inventory posting, idempotency, and correctness/concurrency tests. Establish atomic audit writing before privileged actions.
3. **Returns and oversight:** Implement full refunds/restock choices, shift closure/reconciliation, accessible history/receipts/exports, sales/stock reports, audit review, and integrity queries with concurrency tests.
4. **Usability and operations:** Build the frontend around validated backend flows; benchmark at least 100,000 inventory movements, tune SQL/indexes, verify restore, and document reproducible setup/demo.

## Completion Criteria

* [ ] Owner provisions staff/catalog/registers without public role escalation.

* [ ] Cashier opens a shift, builds a cart, and completes cash/manual noncash sales with immutable receipts and accurate stock.

* [ ] Two cashiers competing for the last item cannot both sell it; retries cannot duplicate a sale/payment.

* [ ] Manager records one full refund with line-specific restock choices and correct current-shift cash impact.

* [ ] A shift closes with computed expected cash, counted cash, variance, and explanation, and rejects subsequent posting.

* [ ] Reports and integrity queries match known fixtures and enforce role visibility.

* [ ] Posted records and audit history resist unauthorized edits/deletion.

* [ ] The documented performance benchmark and backup/restore verification run successfully.

* [ ] A fresh checkout can reproduce setup, tests, benchmark, and the hardware-store demo from the README.

## Outside the Learning-Project Scope

Add features only after a demonstrated workflow requires them: product variants, fractional units/package conversion, suppliers/purchasing, credit sales, split payments, integrated payments, multi-location stock, and restaurant tables/modifiers/kitchen/ingredient workflows. None is part of the single learning-project deliverable; listing them is not a promise of another release.
