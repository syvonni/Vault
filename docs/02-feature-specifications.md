# Vault Feature Specifications — Planning Workbook

Review position **02 of 07**. Finalize user workflows, screen behavior, validation, failures, and acceptance examples after the scope review. Track this review in [GitHub issue #2](https://github.com/syvonni/Vault/issues/2). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Status: draft workbook, not completed feature specifications. Fill and review every in-scope feature in M0-03–M0-05 before code/setup. The [requirements](01-requirements.md) state required behavior; this document contains the draft client workflows below and will hold the detailed feature specifications and acceptance scenarios. The [schema](06-conceptual-schema.md) must be reviewed against these specifications.

Vault as a whole is a learning project. The feature groups organize its work; implementing them in sequence does not imply additional product releases.

## Feature Inventory and Traceability

- **FS-01 — Identity and staff:** POS-FR-01–POS-FR-03. Per-business controlled onboarding/first-owner bootstrap, owner-created staff, credentials/reset, role changes, deactivation, profile, login/logout, session history/revocation. Roles: Owner, Manager, Cashier.
- **FS-02 — Store/register/payment setup:** POS-FR-04. Configuration, allowed edits/disablement, historical labels, open-register restrictions.
- **FS-03 — Catalog and inventory:** POS-FR-05–POS-FR-09. Product/category/SKU/barcode/unit/price rules, search, receipts, corrections, low stock, movement history.
- **FS-04 — Drafts and confirmation:** POS-FR-10–POS-FR-11, POS-FR-17. Own-shift draft ownership, cart edits/version counter, price reconfirmation, draft void.
- **FS-05 — Payment, checkout, receipts, retry:** POS-FR-12–POS-FR-16, POS-FR-29. Cash/change, externally confirmed noncash, atomic completion, snapshots, status lookup, failure recovery.
- **FS-06 — Full refunds:** POS-FR-18–POS-FR-20. Full original amount/method, cashier request plus Manager/Owner approval, separate approver/executor, all original lines, per-line restock decision, executing-shift drawer checks, one-success limit.
- **FS-07 — Shifts and reconciliation:** POS-FR-21–POS-FR-24. Opening float, exclusivity, activity, closure, unresolved drafts/requests, counts and variances.
- **FS-08 — Reports/history/export:** POS-FR-25–POS-FR-26. Visibility, posting dates, gross/refund/net formulas, quantity versus restock, filters, pagination, CSV, empty periods.
- **FS-09 — Integrity and audit:** POS-FR-27–POS-FR-28. Detectable invariant failures, audit coverage, immutable history, safe log values and review access.
- **FS-10 — Client responsibility and navigation:** mobile is the POS workspace; desktop is the Owner/Manager back office. Define mobile register/shift binding, role-aware navigation, receipt delivery, and refund handoff. This maps existing role/business requirements to clients; it does not create separate business data per platform.

- **FS-11 — Tenant onboarding and isolation:** POS-FR-30–POS-FR-31, POS-NFR-09–POS-NFR-10. Controlled business creation, per-business staff/setup, server-derived tenant context, hosted and business-local operation, RLS, same-tenant relationship checks, and isolation for every feature above.

- **FS-12 — Retail pricing:** POS-FR-32–POS-FR-35. Discounts, supported promotion rules, tax configuration/calculation, authorized overrides, calculation order/rounding, immutable snapshots, receipts, refunds, reports, and concurrent changes. Exact rule semantics remain pending review.
- **FS-13 — Business deletion:** POS-FR-36. Owner verification/confirmation, export opportunity, active work, lifecycle/cancellation decisions, access revocation, controlled tenant-scoped purge, safe outcomes, retention/backup handling, and hosted/local execution responsibilities.

Each group may contain multiple feature specifications. Referencing requirements here establishes coverage intent, not completed acceptance criteria.

## Deployment Context for Every Workflow

The same workflows run against either a shared hosted API or a business-local API. Clients connect only to the chosen API; PostgreSQL is reachable only by its backend. A local installation contains one business but keeps the same tenant-aware schema and permission checks. No cloud/local data synchronization or automatic failover is planned.

Before login, the client uses a configured installation endpoint. Specify a deliberate endpoint-change flow: clear the previous token/cache and reauthenticate; never send old tokens to a new server. Resolve active carts/shifts and uncertain posting outcomes against their original installation before switching. The endpoint identifies an installation, not permission to select a tenant; tenant context still comes from the authenticated account.

Local cash checkout must work during an internet outage if the store server and LAN are reachable. Losing the API/LAN blocks new posting and uses existing unknown-response recovery. A manual external payment provider may still need internet independently. Specify clear connection errors without silently redirecting work to a cloud or per-device database.

## Workflow Drafts by Client

Status: initial workflow drafts for the user's mobile-POS/desktop-back-office direction. These are review material, not finalized feature specifications or screen designs. Record completion of M0-03–M0-05 in the corresponding GitHub issues only after inputs, errors, acceptance scenarios, and open questions are resolved for each feature.

The requirements are the source for business rules; these steps explain their execution order and client location. If a workflow needs a different business rule, amend its requirement explicitly rather than maintaining contradictory copies.

### WF-01 — Initial Business and Staff Setup

**Client/actor:** desktop; initial Owner setup followed by authenticated Owner actions. **Links:** FS-01–FS-02, POS-FR-01–POS-FR-04, POS-FR-30–POS-FR-31.

1. An authorized installation operator (hosted service operator or designated business-local administrator) atomically provisions a business, its settings, built-in cash method, and first Owner through controlled onboarding. This is not a public registration screen or a fourth staff role; the operator retains a protected operational audit record.
2. The Owner logs into the desktop back office, sees their business identity, and checks registers and payment-method configuration.
3. The Owner creates individual Manager/Cashier accounts and assigns roles; define secure initial credential delivery/change during the identity specification.
4. Each staff member logs in using email credentials unique within their installation for their single business. The API derives tenant context from that account and checks current role/record permissions for subsequent actions. Clients cannot select a different tenant by changing a request field.

**Result/exceptions:** configuration and attributed staff accounts, with audit records for privileged changes. No shared cashier account; no fourth review-only role. Protect the last active Owner in each business. Owners cannot create staff for another business or access service-operator tooling. Exact first-login/reset screens remain to be specified.

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

1. Log in, then select an enabled register representing the checkout station and its cash drawer, not the phone itself.
2. If the operator already has an open shift, resume that shift after validation instead of creating another. Replacement-device resume requires explicit same-operator authentication and atomically invalidates the previous mobile control session.
3. Otherwise enter opening cash float and submit the shift-opening action.
4. Backend checks active staff, enabled register, and one-open-shift-per-operator/register; it creates the shift and completes the operation atomically.
5. Open the mobile selling workspace with visible register, operator, shift, and connection status.

**Result/exceptions:** session login and business shift are separate. Expired login requires reauthentication; it does not close the shift. A mobile device is not automatically a new register. One mobile control session may operate a shift at a time; the server rechecks it for every posting. Device handoff does not transfer cashier identity or drawer ownership. Specify race/recovery behavior before implementation.

### WF-05 — Mobile Sale, Payment, and Receipt

**Client/actor:** mobile; operator of the open shift. **Links:** FS-04–FS-05, POS-FR-06, POS-FR-10–POS-FR-16, POS-FR-29.

1. Search active products by SKU/barcode text/name/category and add whole quantities to the cart. Repeated selection consolidates the same SKU.
2. Review server-calculated lines, catalog/effective prices, eligible discounts/promotions, authorized overrides, tax breakdown, stock, and final payable total. Specify the calculation/approval rules before implementing this expanded flow. Draft edits update the cart's edit counter without reserving stock.
3. Choose cash or a configured manual noncash method. Cash records tender/change; noncash requires external verification/reference. Confirm what the cashier should do before externally collecting money in the detailed payment specification.
4. Submit the confirmed cart/version/payment data with one stable action key.
5. Backend rechecks permission, shift/register/method/product eligibility, current prices, and stock under the appropriate locks.
6. On success, complete the sale, one payment, receipt snapshots, exact stock deductions, and the request outcome atomically.
7. Show the receipt on mobile and begin a new deliberate sale only with a new draft/action. Desktop reports can read the same completed records.

**Result/exceptions:** changed price/cart requires reconfirmation; insufficient stock rejects checkout; no partial recorded payment/deduction survives failure. A lost response shows an unresolved state and looks up/replays the same request. Vault does not collect/provider-refund money automatically. If payment was collected externally but Vault definitively rejected checkout, mark reconciliation required and involve a Manager/Owner. Preserve original references, amount, actors, and outcome/resolution evidence; no automatic second collection/refund. Exact reconciliation states and recovery actions remain a required design task; database rollback cannot undo an external payment.

Receipt viewing plus PDF download/share is confirmed for mobile and accessible desktop history. Layout, file generation/access control, and platform share/save behavior require specification. Dedicated printer hardware and camera barcode scanning remain deferred.

### WF-06 — Cancel a Draft Sale and Look Up Sales

**Client/actor:** mobile; operator accessing their own shifts. **Links:** FS-04–FS-05, POS-FR-16–POS-FR-17, POS-FR-26, POS-FR-29.

1. Resume/search an accessible draft or receipt by allowed filters/reference.
2. To abandon an uncompleted draft, record a reason and submit void; serialize it against checkout.
3. To inspect a completed sale, display its immutable receipt and any linked refund; do not offer completed-sale voiding.
4. Resolve uncertain operation outcomes through the status flow before attempting a new posting.

**Result/exceptions:** cashiers cannot inspect other operators' records merely by knowing a reference. Desktop Owner/Manager search has the broader authorized store scope.

### WF-07 — Full Refund and Return

**Confirmed client/actor split:** mobile operator executes under their own shift; a separately authenticated Manager/Owner authorizes cashier requests from the desktop back office. Managers/Owners operating their own mobile shift may authorize and execute directly. **Links:** FS-06, POS-FR-18–POS-FR-20, POS-FR-24, POS-FR-29.

1. The mobile operator finds a completed sale through a tenant-scoped refund lookup, which reveals only information needed for the return. A cashier's ordinary sales history remains limited to their own shifts; refund lookup does not grant store-wide browsing.
2. Enter the reason and full set of restock/do-not-restock decisions. The server previews the original paid total/method, eligibility, and executing-shift cash requirements.
3. A cashier submits the request for Manager/Owner approval. The approver uses their own authenticated desktop session; credentials are never shared. Approval binds the sale, amount, decisions/reason, executor/shift, and request version. Changes require approval again. A Manager/Owner on their own mobile shift may approve their own execution.
4. Immediately before execution, revalidate current approval/roles, sale eligibility, active device authority, current shift, and drawer cash. The executing operator performs the external cash/provider refund and records its confirmation/reference; Vault does not process the provider transfer.
5. Submit once with the stable operation key. The server revalidates and atomically commits the successful full refund, distinct approver/executor evidence, line decisions, stock restoration, audit, and operation outcome. Cash impact belongs to the executing shift; no second successful full refund is allowed.
6. Mobile displays the refund confirmation; desktop fetches the same result. The original sale/shift and pricing snapshots remain unchanged.

**Recovery:** server prechecks cannot guarantee an external action and database commit succeed together. Missing responses use original outcome lookup/retry; an external refund followed by definitive posting rejection creates a reconciliation-required case for Manager/Owner investigation. Never return money twice automatically. Specify approval expiry/revocation, execution claims preventing duplicate external payouts, interrupted approval/execution, reconciliation states and drawer/close-shift behavior before implementing. This confirmed direction supersedes the earlier Manager-only execution draft and requires DBML changes.

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

## Tenant Boundary Across All Workflows

Every workflow above is confined to the authenticated staff account's business. Display its identity in both clients; filter lookups and validate related IDs before posting. A Manager/Owner's store-wide visibility means their own tenant only. Staff identities cannot belong to or switch between businesses in this scope. Business-specific identifiers can repeat across tenants; canonical email unique within the installation keeps authentication lookup unambiguous. Separate installations have independent accounts and tokens.

Plan authentication as a narrowly scoped credential lookup before tenant context is established; it must expose no business data and must not grant the general API an RLS bypass. Specify how tokens resolve to one account/tenant, how missing/revoked context fails, and how pooled connections and background recovery obtain and clear context. Review that design alongside the auth library choice.

## Newly Included Workflows Requiring Specification

Pricing configuration belongs in the desktop back office; mobile checkout must show applied pricing and tax before confirmation. Review discount permissions, supported promotion types, calculation order/rounding/allocation, tax-inclusive versus exclusive treatment, and override authorization/handoff. Preserve the original charged amounts and applied rules for receipts, reports, and full refunds. Existing simple cash-sale examples remain useful with no discount/tax configured; they do not validate the expanded pricing behavior.

Business deletion begins with the business Owner in the desktop back office. Specify identity verification, impact disclosure, export opportunity, active shifts/drafts/processing operations, confirmation, lifecycle/grace/cancellation choices, access revocation, purge/retry outcome, and backup/restore/retention evidence. Decide operator/local-administrator responsibilities without giving normal staff unrestricted database privileges. No deletion period or tax/promotion algorithm is selected by this scope addition.

## Confirmed Client Decisions

- **Desktop application container:** Vue + Tauri for Windows/macOS; Django remains the backend.
- **Refunds:** cashier mobile execution under their own shift requires specific Manager/Owner approval; record both identities. Direct Manager/Owner own-shift execution is allowed.
- **Registers/devices:** select an enabled logical register; one operator and one active mobile control session per shift. Same-operator replacement-device resume invalidates old control authority.
- **Receipts:** on-screen plus PDF download/share; dedicated printer integration remains deferred.
- **Payment uncertainty:** unknown response means check original outcome; confirmed external payment/refund with definitive server rejection requires Manager/Owner reconciliation, never automatic repeated money movement.

## Remaining Client Specification Questions

- Specify approval expiry/revocation, restricted refund lookup, request/execution claims and race cases, approval version binding, and the receipt/refund confirmation layout. The approval/execution client split is settled.
- Specify how one-active-device control is represented and invalidated, how replacement-device resume races with in-flight actions, and how the old device displays lost authority.
- Specify reconciliation case states, evidence, resolution permissions, safe completion/compensation, and open-shift/closure handling when external money movement occurred but posting failed. Never assume database rollback undoes provider actions.
- Specify PDF layout, server/client generation boundary, safe download/share, and tenant/role access; printer hardware remains deferred.
- Confirm mobile OS/device/screen-size acceptance and client endpoint/connection screens. Desktop targets Windows/macOS with Tauri; neither desktop checkout nor read-only mobile companion is required.

## Required Specification Template

For each concrete feature, write:

1. **ID, title, status, and requirement links.** Identify whether drafted/reviewed, and which POS-FR rules it implements.
2. **Purpose and actor.** Explain the business problem, authenticated tenant, and permitted role/record scope.
3. **Preconditions.** Include login, active status, shift/document states, and required existing records.
4. **Inputs.** Define fields, types, required/optional rules, precision/limits, examples, and server-derived values.
5. **Screen/navigation flow.** Describe entry point, steps, confirmation, loading/empty/error states, keyboard behavior, and completion view.
6. **Validation and authorization.** Specify rules, when they are rechecked, safe error codes, and prohibited operations.
7. **State and data effects.** Name affected entities, snapshots, atomic boundaries, audit events, and records that must remain unchanged.
8. **API contract outline.** Describe request/response fields and errors without implementing endpoints. Include idempotency/version expectations where relevant.
9. **Failure/retry/concurrency.** Cover validation rejection, database rollback, lost responses, simultaneous requests, recovery, and whether a deliberate new attempt needs a new key.
10. **Acceptance scenarios.** Use Given/When/Then examples with known amounts/quantities, including happy paths, boundaries, permission failures, and concurrency where relevant.
11. **Schema mapping and open questions.** Trace required fields/relationships/invariants and flag missing/ambiguous decisions before final schema approval.

## Additional Acceptance Coverage to Plan

- Pricing: fixed/percentage and line/sale discounts once selected, eligibility/stacking limits, promotion start/end and simultaneous edits, override authorization and stale approvals, tax-inclusive/exclusive cases once selected, rounding/allocation, excessive discounts/overflow, receipt snapshots, full refunds after rule changes, and consistent tax/discount reports. Use reviewed known-value fixtures rather than assuming formulas now.
- Deletion: wrong-role/foreign-tenant requests, Owner reauthentication/confirmation, concurrent requests, pending shifts/postings, export access, cancellation if selected, revoked sessions, interrupted purge/retries, other tenants unchanged, protected retained evidence, and restoring a backup containing a deleted business. Cover hosted and business-local installations.

## Example Acceptance Scenarios — Drafts to Expand

- Given Businesses A and B with the same SKU/register code but different stock/prices, when A searches, sells, reports, exports, or checks an operation, then only A's records and effects appear.
- Given A's session and a record ID from B, when it is used in a URL, request body, related record, receipt lookup, or operation replay, then access is denied without exposing B's data or committing effects.
- Given a database connection previously used for A, when B or a request without tenant context reuses it, then A's context/data cannot leak; verify under the actual runtime database role.
- Given independent businesses, when the last Owner is changed or the built-in cash method is seeded, then the rule is enforced separately per business.
- Given a controlled onboarding failure midway, then no partially configured business/Owner remains and the operational failure is recorded safely.
- Given a business-local installation with a reachable server/LAN and no internet, when staff complete a cash sale from mobile and review it on desktop, then both use that local database and stock/shift totals agree.
- Given an unreachable local server or LAN, when checkout is attempted, then no client invents a local posting or switches to the cloud; uncertain outcomes are checked on the original server after recovery.
- Given a deliberate change to an independent API endpoint, when the client connects, then it discards old tokens/cached business data and requires login; it never transfers drafts or request replays automatically. Resolve active shifts/pending outcomes against the old installation first.
- Given separate hosted/local backups, when either is restored into an isolated target, then its own records, privileges, and tenant context remain correct; the databases are not merged.
- Given two businesses on the hosted service, when each uses mobile and desktop, then its clients show its own business identity and see consistent business-specific data.
- Given 10 hammers priced PHP 150 and a valid cashier shift, when two are sold for PHP 300 with PHP 500 tender, then change is PHP 200, stock becomes 8, and expected drawer cash increases by PHP 300.
- Given one remaining unit and two independent cashier shifts, when checkout competes concurrently, then at most one sale completes and stock cannot become negative.
- Given a committed checkout with a lost response, when the same action/key is retried, then the existing receipt/outcome returns and there is no second payment or stock deduction.
- Given a confirmed draft whose product price changes before checkout, when the cashier submits it, then checkout requires review/reconfirmation instead of silently changing the amount.
- Given a completed sale, when a cashier attempts to execute without valid Manager/Owner approval, then authorization denies it. A valid approved request executes only under its bound operator/shift; changing the request invalidates approval.
- Given an open shift with a processing checkout request, when closure is attempted, then closure is blocked until that request is resolved; a timeout alone does not establish failure.

These examples do not replace the per-feature specification or its error/UI details. Keep the feature-specification review issue open until all groups have reviewable specifications and open decisions are resolved.
