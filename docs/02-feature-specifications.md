# Vault Feature Specifications — Planning Workbook

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Review WF-01: first-owner setup, staff provisioning, and individual login.
- [ ] Review WF-02–WF-03: desktop catalog, stock receipts, and corrections.
- [ ] Review WF-04–WF-06: mobile register/shift opening, cart, payment, receipts, history, and draft void.
- [ ] Review WF-07: full refunds, restock decisions, permissions, and cash/shift attribution.
- [ ] Review WF-08–WF-09: mobile shift closure and desktop reporting/audit review.
- [ ] Resolve the cross-client questions, including register/device binding, receipt output, and externally collected payments after a rejected posting.
- [ ] Complete the specification template for each feature with inputs, screen steps, validation, failures/retries, acceptance examples, and requirement/schema links.
- [ ] Record the agreed changes and confirm the workflows/specifications are complete before moving to 03.

Review position **02 of 07**. Finalize user workflows, screen behavior, validation, failures, and acceptance examples after the scope review. See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Status: draft workbook, not completed feature specifications. Fill and review every in-scope feature in M0-03–M0-05 before code/setup. The [requirements](01-requirements.md) state required behavior; this document contains the draft client workflows below and will hold the detailed feature specifications and acceptance scenarios. The [schema](06-conceptual-schema.md) must be reviewed against these specifications.

Only one learning-project deliverable is planned. Implementing features in sequence does not imply additional product releases.

## Feature Inventory and Traceability

- **FS-01 — Identity and staff:** POS-FR-01–POS-FR-03. First-owner bootstrap, owner-created staff, credentials/reset, role changes, deactivation, profile, login/logout, session history/revocation. Roles: Owner, Manager, Cashier.
- **FS-02 — Store/register/payment setup:** POS-FR-04. Configuration, allowed edits/disablement, historical labels, open-register restrictions.
- **FS-03 — Catalog and inventory:** POS-FR-05–POS-FR-09. Product/category/SKU/barcode/unit/price rules, search, receipts, corrections, low stock, movement history.
- **FS-04 — Drafts and confirmation:** POS-FR-10–POS-FR-11, POS-FR-17. Own-shift draft ownership, cart edits/version counter, price reconfirmation, draft void.
- **FS-05 — Payment, checkout, receipts, retry:** POS-FR-12–POS-FR-16, POS-FR-29. Cash/change, externally confirmed noncash, atomic completion, snapshots, status lookup, failure recovery.
- **FS-06 — Full refunds:** POS-FR-18–POS-FR-20. Full original amount/method, all original lines, per-line restock decision, current-shift drawer checks, one-success limit.
- **FS-07 — Shifts and reconciliation:** POS-FR-21–POS-FR-24. Opening float, exclusivity, activity, closure, unresolved drafts/requests, counts and variances.
- **FS-08 — Reports/history/export:** POS-FR-25–POS-FR-26. Visibility, posting dates, gross/refund/net formulas, quantity versus restock, filters, pagination, CSV, empty periods.
- **FS-09 — Integrity and audit:** POS-FR-27–POS-FR-28. Detectable invariant failures, audit coverage, immutable history, safe log values and review access.
- **FS-10 — Client responsibility and navigation:** mobile is the POS workspace; desktop is the Owner/Manager back office. Define mobile register/shift binding, role-aware navigation, receipt delivery, and refund handoff. This maps existing role/business requirements to clients; it does not create separate business data per platform.

Each group may contain multiple feature specifications. Referencing requirements here establishes coverage intent, not completed acceptance criteria.

## Workflow Drafts by Client

Status: initial workflow drafts for the user's mobile-POS/desktop-back-office direction. These are review material, not finalized feature specifications or screen designs. Keep M0-03–M0-05 unchecked until inputs, errors, acceptance scenarios, and open questions are resolved for each feature.

The requirements are the source for business rules; these steps explain their execution order and client location. If a workflow needs a different business rule, amend its requirement explicitly rather than maintaining contradictory copies.

### WF-01 — Initial Business and Staff Setup

**Client/actor:** desktop; initial Owner setup followed by authenticated Owner actions. **Links:** FS-01–FS-02, POS-FR-01–POS-FR-04.

1. An authorized setup operator provisions the first Owner through the controlled bootstrap procedure; this is not a public registration screen.
2. The Owner logs into the desktop back office and checks business identity, registers, and payment-method configuration.
3. The Owner creates individual Manager/Cashier accounts and assigns roles; define secure initial credential delivery/change during the identity specification.
4. Each staff member logs in using their own credentials. The API checks current role/record permissions for subsequent actions.

**Result/exceptions:** configuration and attributed staff accounts, with audit records for privileged changes. No shared cashier account; no fourth review-only role. Protect the last active Owner. Exact first-login/reset screens remain to be specified.

### WF-02 — Catalog Maintenance and Stock Receipt

**Client/actor:** desktop; Manager/Owner. **Links:** FS-03, POS-FR-05–POS-FR-07.

1. Create/select a category and product with unique SKU, optional barcode, unit label, selling price, and low-stock threshold.
2. Open stock receipt, select products and positive whole quantities, and provide a delivery note/reference.
3. Review and submit once using the action's idempotency key.
4. The backend validates permission/data and atomically appends stock movements, audit evidence, and the successful operation outcome.
5. Desktop shows updated stock/movement history; later mobile lookups read that same backend stock.

**Result/exceptions:** products start at zero before stock posting. Invalid/duplicate SKU and malformed quantities are rejected. A lost response uses status lookup/same-key replay, not a second receipt posting. This does not create supplier bills or purchase accounting.

### WF-03 — Stock Correction

**Client/actor:** desktop; Manager/Owner. **Links:** FS-03, POS-FR-08–POS-FR-09.

1. Inspect a product's recorded quantity and movement history, then enter a signed correction with its reason.
2. Review the proposed adjustment and submit once.
3. Backend locks/rechecks the product, rejects a negative resulting balance, and atomically records the correction/audit/outcome.
4. Show the resulting stock and attributed movement; do not overwrite earlier stock history.

**Result/exceptions:** correction is a new movement, not editing an old sale. Recheck against concurrent sales; precise stale-count handling and physical-count input semantics require review.

### WF-04 — Start the Mobile Cashier Session

**Client/actor:** mobile; Cashier, or Manager/Owner acting under their own identity. **Links:** FS-01, FS-07, POS-FR-01, POS-FR-21–POS-FR-22.

1. Log in, then select the configured register/cash responsibility being operated.
2. If the operator already has an open shift, resume that shift after validation instead of creating another.
3. Otherwise enter opening cash float and submit the shift-opening action.
4. Backend checks active staff, enabled register, and one-open-shift-per-operator/register; it creates the shift and completes the operation atomically.
5. Open the mobile selling workspace with visible register, operator, shift, and connection status.

**Result/exceptions:** session login and business shift are separate. Expired login requires reauthentication; it does not close the shift. A mobile device is not automatically a new register. Device changes, allowed register binding, and physical drawer sharing remain review decisions.

### WF-05 — Mobile Sale, Payment, and Receipt

**Client/actor:** mobile; operator of the open shift. **Links:** FS-04–FS-05, POS-FR-06, POS-FR-10–POS-FR-16, POS-FR-29.

1. Search active products by SKU/barcode text/name/category and add whole quantities to the cart. Repeated selection consolidates the same SKU.
2. Review server-calculated lines, current prices, stock, and total. Draft edits update the cart's edit counter without reserving stock.
3. Choose cash or a configured manual noncash method. Cash records tender/change; noncash requires external verification/reference. Confirm what the cashier should do before externally collecting money in the detailed payment specification.
4. Submit the confirmed cart/version/payment data with one stable action key.
5. Backend rechecks permission, shift/register/method/product eligibility, current prices, and stock under the appropriate locks.
6. On success, complete the sale, one payment, receipt snapshots, exact stock deductions, and the request outcome atomically.
7. Show the receipt on mobile and begin a new deliberate sale only with a new draft/action. Desktop reports can read the same completed records.

**Result/exceptions:** changed price/cart requires reconfirmation; insufficient stock rejects checkout; no partial recorded payment/deduction survives failure. A lost response shows an unresolved state and looks up/replays the same request. Vault does not collect/provider-refund money automatically. Handling money already collected externally when final checkout is rejected remains an explicit planning decision; database rollback cannot undo an external payment.

Receipt viewing is in scope; mobile download/share/printing and the output format must be specified before implementation. Camera barcode scanning and printer hardware are not added by mentioning barcode text lookup.

### WF-06 — Mobile Draft Void and Sales Lookup

**Client/actor:** mobile; operator accessing their own shifts. **Links:** FS-04–FS-05, POS-FR-16–POS-FR-17, POS-FR-26, POS-FR-29.

1. Resume/search an accessible draft or receipt by allowed filters/reference.
2. To abandon an uncompleted draft, record a reason and submit void; serialize it against checkout.
3. To inspect a completed sale, display its immutable receipt and any linked refund; do not offer completed-sale voiding.
4. Resolve uncertain operation outcomes through the status flow before attempting a new posting.

**Result/exceptions:** cashiers cannot inspect other operators' records merely by knowing a reference. Desktop Owner/Manager search has the broader authorized store scope.

### WF-07 — Full Refund and Return

**Draft client/actor:** mobile execution by Manager/Owner under their own open shift; desktop offers history/audit inspection. **Links:** FS-06, POS-FR-18–POS-FR-20, POS-FR-24.

1. Authorized Manager/Owner locates the completed sale and checks full-refund eligibility and original payment.
2. Enter a reason and review every original line with a restock/do-not-restock choice.
3. Verify the external refund as required and record its reference for noncash, following the reviewed payment/refund ordering procedure.
4. Submit once; backend validates original amount/method, one-success limit, current shift, expected cash where relevant, and exact restoration entries.
5. Atomically record the full refund, line decisions, restoration/audit entries, and operation outcome. Show linked refund confirmation; desktop history reflects it without rewriting the original sale/shift.

**Open handoff decision:** the existing rule requires the refund operator's own shift. Do not silently let a Manager approve on desktop while attributing the refund to a Cashier's shift. Decide whether to keep mobile Manager execution or explicitly redesign approval/execution actors, cash-drawer attribution, and schema/requirements. Cashiers cannot perform refunds under the current rules.

### WF-08 — Mobile Shift Close and Desktop Review

**Clients/actors:** mobile operator closes; desktop Owner/Manager reviews. **Links:** FS-07–FS-08, POS-FR-21–POS-FR-26.

1. On mobile, resolve processing actions and complete/void remaining drafts.
2. Review own shift sales/refunds/method totals, count physical cash, and enter the counted amount.
3. Backend locks/checks the shift, derives expected cash and variance, and requires an explanation for nonzero variance.
4. Commit immutable closure/audit/outcome together, then prevent new postings in that shift.
5. Owner/Manager uses desktop to inspect closed shifts, differences, and related sales/refunds.

**Result/exceptions:** a failed connection does not establish that closing failed. Recheck the same request. Refunds posted in another shift affect that later shift, not the original closed drawer.

### WF-09 — Desktop Metrics, Reports, and Audit Review

**Client/actor:** desktop; Manager/Owner, with Owner-only staff administration. **Links:** FS-08–FS-09, POS-FR-25–POS-FR-29.

1. Choose date range, timezone-aware filters, operator/register/method, or a specific reference.
2. Inspect gross sales, refunds, net sales, units, stock/low-stock information, or shift variance using the defined formulas.
3. Drill into the underlying accessible sale/payment/refund/stock/shift records; export the permitted filtered data if needed.
4. Review privileged action audit history and integrity-report findings; investigate inconsistencies through controlled actions rather than editing historical records.

**Result/exceptions:** metrics mean operational sales/stock/cash measures, not profit or full accounting. Audit review is read-only history access within Owner/Manager capabilities; it does not restore the removed role. Exact dashboard charts/layouts remain a screen-design task.

## Cross-Client Questions Before Workflow Approval

- How does a phone/tablet select or bind to a logical register, and what physical drawer/cash responsibility does it represent? Can the same operator resume that shift on a replacement device?
- Does refund execution stay on an authorized Manager/Owner mobile shift, or is a separate approval/execution flow required? The latter changes business attribution and may change the schema.
- What should the operator do if manual noncash payment/refund has happened externally but the final Vault posting is rejected? Specify recovery evidence and staff actions without pretending the database can undo provider actions.
- What is the supported receipt delivery format on mobile? Text barcode lookup is already in scope; camera scanning and printer integration require separate decisions.
- Which phone/tablet platforms and screen sizes must be demonstrated? Neither a read-only mobile companion nor desktop checkout is an assumed required deliverable now.

## Required Specification Template

For each concrete feature, write:

1. **ID, title, status, and requirement links.** Identify whether drafted/reviewed, and which POS-FR rules it implements.
2. **Purpose and actor.** Explain the business problem and permitted role/record scope.
3. **Preconditions.** Include login, active status, shift/document states, and required existing records.
4. **Inputs.** Define fields, types, required/optional rules, precision/limits, examples, and server-derived values.
5. **Screen/navigation flow.** Describe entry point, steps, confirmation, loading/empty/error states, keyboard behavior, and completion view.
6. **Validation and authorization.** Specify rules, when they are rechecked, safe error codes, and prohibited operations.
7. **State and data effects.** Name affected entities, snapshots, atomic boundaries, audit events, and records that must remain unchanged.
8. **API contract outline.** Describe request/response fields and errors without implementing endpoints. Include idempotency/version expectations where relevant.
9. **Failure/retry/concurrency.** Cover validation rejection, database rollback, lost responses, simultaneous requests, recovery, and whether a deliberate new attempt needs a new key.
10. **Acceptance scenarios.** Use Given/When/Then examples with known amounts/quantities, including happy paths, boundaries, permission failures, and concurrency where relevant.
11. **Schema mapping and open questions.** Trace required fields/relationships/invariants and flag missing/ambiguous decisions before final schema approval.

## Example Acceptance Scenarios — Drafts to Expand

- Given 10 hammers priced PHP 150 and a valid cashier shift, when two are sold for PHP 300 with PHP 500 tender, then change is PHP 200, stock becomes 8, and expected drawer cash increases by PHP 300.
- Given one remaining unit and two independent cashier shifts, when checkout competes concurrently, then at most one sale completes and stock cannot become negative.
- Given a committed checkout with a lost response, when the same action/key is retried, then the existing receipt/outcome returns and there is no second payment or stock deduction.
- Given a confirmed draft whose product price changes before checkout, when the cashier submits it, then checkout requires review/reconfirmation instead of silently changing the amount.
- Given a completed sale, when a cashier attempts a refund, then authorization denies it. A Manager/Owner may use the separately specified full-refund flow.
- Given an open shift with a processing checkout request, when closure is attempted, then closure is blocked until that request is resolved; a timeout alone does not establish failure.

These examples do not replace the per-feature specification or its error/UI details. Completion remains unchecked until all groups have reviewable specifications and open decisions are resolved.
