# Vault POS and Inventory System Requirements

Review position **01 of 07**. Agree on purpose, scope, mobile/desktop responsibilities, roles, and required behavior. Track this review in [GitHub issue #1](https://github.com/syvonni/Vault/issues/1). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

## Purpose and Business Problem

Vault is a point-of-sale and inventory system supporting two deployment modes: a shared hosted service for independent businesses, or a business-local installation running on the store’s own server. Each business (tenant) has one physical store; the first supported workflows target hardware stores and similar retail businesses. It helps staff sell products, record payments, maintain stock counts, issue internal sales receipts, process full-sale refunds, and reconcile cashier shifts.

A store needs to answer: What sold today? What is left on the shelf? Can two cashiers sell the same last item? Was a sale recorded twice after a retry? Does the counted cash match what the shift should contain? Who changed a price, adjusted stock, or authorized a refund?

Vault supports these daily operations. It records payments received outside the software; it does not move money through banks or payment networks.

This revision replaces the previous simulated-wallet requirements. All `POS-FR-*` identifiers are new and supersede the old `FR-*` identifiers; they are not a renumbering of the old features.

## Single-Project Scope and Assumptions

### Included Scope

| Area | Included behavior |
| --- | --- |
| Business and registers | Hosted mode supports multiple independent businesses sharing a Django API and PostgreSQL database. Business-local mode runs the same backend and schema on a designated store server with a separate database containing one business. Each business has one physical location, multiple registers, and concurrent cashiers; its records are isolated from other businesses. |
| Deployment modes | Support hosted and business-local installations with identical business features. Each installation has one authoritative PostgreSQL database; desktop/mobile clients access it through the Django API. |
| Business onboarding | A controlled setup process creates each business, its settings, built-in cash method, and first Owner. Owners provision staff within their own business. |
| Product catalog | Products have a SKU unique within their business, category, name, package/unit label, selling price, low-stock threshold, and active/inactive status. Categories, unit labels, products, registers, and manual noncash payment methods are configurable records. |
| Pricing | PHP catalog prices with discounts, tax computation, promotions, and authorized checkout price overrides. Final payable amounts use at most two decimal places; calculation precision, rounding, eligibility, and combinations require planning review. |
| Business deletion | An Owner can initiate a verified deletion workflow for their business in either deployment mode, with confirmation, an export opportunity, safe handling of active work, and clear live-data/backup deletion expectations. |
| Inventory | On-hand quantity derives from an append-only inventory-movement ledger. Stock cannot become negative. Stock receipts record quantities and a delivery note/reference. |
| Checkout and payment records | One complete payment per sale. Cash checkout supports tender and change; manual noncash checkout records external confirmation/reference information. |
| Receipts | Printed/downloaded receipts serve as internal sales records. |
| Refunds | A Manager/Owner can record one full refund per completed sale using the original payment method. Each returned line is marked restock or do-not-restock. |
| Cash shifts | One cashier and one register per shift; neither may have multiple open shifts. Each shift records opening float, activity, counted closing cash, and variance. |
| Financial tracking | Track sales, refunds, recorded payments, and drawer cash. |

### Operating Assumptions and Constraints

| Area | Assumption or constraint |
| --- | --- |
| Project boundary | Vault is a learning project supporting hosted and business-local deployment. Each business has one physical store, with workflows initially focused on hardware stores and similar retailers. |
| Staff identity | Each staff account belongs to exactly one business and has one business role. Canonical login emails are unique across all tenants within an installation; independent installations do not share an identity directory. For this learning scope, shared identities across businesses and account switching are not included. |
| Business isolation | Every business-owned record carries a tenant identifier. The server derives business context from the authenticated account, never trusts a submitted tenant ID, and applies isolation to reads, writes, reports, exports, and operation outcomes. |
| Quantities and packaging | Each SKU is sold and stocked in positive whole quantities. A box and an individual item are separate SKUs. |
| Catalog flexibility | The initial seed catalog is hardware-specific. Sales, payment, inventory, and shift records use general retail concepts; future business modules require deliberate extensions rather than arbitrary custom fields. |
| Cart availability | Draft carts do not reserve stock. Availability must be validated at checkout. |
| Connectivity | Both clients require their selected installation’s API to be reachable; only the backend connects to PostgreSQL. Hosted operation needs internet access. Business-local operation can continue during an internet outage while its store server and LAN remain available; individual devices do not post independently when disconnected. |
| Payment handling | Staff handle and verify external payments/refunds outside Vault; Vault records them. |
| Installation boundary | A business uses one authoritative installation. Hosted and local databases do not replicate, synchronize, or merge; accounts/tokens belong to their installation. Changing installations requires reauthentication and discarding cached business data; active carts, shifts, and uncertain postings must be resolved against their original server first. |
| Financial interpretation | Stock receipts do not create supplier debt or purchase-accounting records. Sales and drawer-cash measures do not represent full business accounts or profit. |

### Limitations and Deferred Work

These capabilities are outside the current deliverable. Listing them here does not commit this learning project to a later release.

| Area | Excluded or deferred capabilities |
| --- | --- |
| Business structure | Multiple branches/locations within one business, warehouses, and inter-location transfers. |
| Service administration | Turnkey server installers, a universal host-OS support guarantee, automated installation-to-installation migration, public business signup, subscription billing, plans/quotas, cross-business staff accounts, and a platform-administration UI. Controlled onboarding remains in scope. |
| Quantities and packaging | Automatic package conversion and fractional measurements. |
| Statutory documents | Statutory invoicing and certification. Internal sales receipts do not fulfill this scope. |
| Inventory operations | Reservations, backorders, recipes, batches, serial tracking, and product bundles. |
| Payments and credit | Split tender, credit sales, customer invoices, partial payments, and integrated payment terminals. Vault does not process payments or move money through banks/payment networks. |
| Refunds | Partial refunds, exchanges, and refund cancellation. |
| Cash management | Other cash-in/cash-out events, petty cash, and bank deposits. |
| Connectivity | Disconnected per-device posting, cloud/local synchronization, replication, and automatic failover between installations. Internet-independent store-LAN operation is included; it still requires a reachable local server. |
| Accounting and purchasing | Payroll, supplier payables, purchase orders, expenses, general-ledger accounting, inventory valuation, cost of goods sold, and profit reporting. |

## Client Responsibilities

The agreed client split is **mobile POS plus desktop back office**, both using the same Django REST API/PostgreSQL database. This replaces the earlier desktop-POS/mobile-manager-companion proposal. Detailed workflow drafts and open questions live in [02-feature-specifications.md](02-feature-specifications.md#workflow-drafts-by-client).

* **Mobile (React Native):** cashier login, register/shift opening, product lookup, cart, cash/manual-noncash checkout, receipt viewing, own sales/shift activity, cancellation of unfinished draft sales, and shift cash count/closure. A cashier may request and execute a full refund from their own open shift only after a Manager/Owner authorizes that specific refund. A Manager/Owner operating their own shift may execute directly; approval and execution identities are recorded separately.

* **Desktop (Vue + Tauri):** Owner/Manager business/staff/register/payment setup, product/category/price and pricing-rule maintenance, Owner-only business deletion, stock receipt/adjustment, movement/low-stock review, sales and shift history, operational metrics/reports/exports, integrity checks, and audit-log review. Managers/Owners review and approve cashier refund requests under their own authenticated identity. Desktop does not execute cash payouts from a cashier’s drawer. Tauri packages the Vue interface as a Windows/macOS application; it is the desktop application container, not the Django backend.

* **Shared behavior:** individual staff identity, current permissions, authoritative product/stock/payment outcomes, and retry/status handling are server responsibilities. A mobile client does not grant a cashier Manager permissions; a platform split does not create separate inventories or user tables.

A register represents a checkout station/cash drawer, not a phone. At shift opening, the operator selects an enabled register. One active operator owns each register shift; the server permits only one active mobile control session for that shift. After reauthentication, the same operator can explicitly resume it on a replacement device; the old device loses shift-control authority. No staff identity sharing or automatic transfer of drawer responsibility is allowed.

Receipts are available on screen and as PDF downloads/shares, subject to the same role/tenant access rules. Dedicated receipt-printer and camera-scanner integration remain deferred. A wrong cart line can be edited; **voiding a draft sale** cancels an unfinished sale without posting payment or stock effects. Completed sales require refunds.

Mobile submits sales to its selected Django API; desktop fetches those saved records from the same server. This communication is included in both deployment modes. Only independent local/cloud database synchronization and disconnected per-device posting are excluded.

For a missing response, check the original operation outcome before retrying. If external payment/refund occurred but Vault definitively rejected the posting, show **reconciliation required** and involve a Manager/Owner; never automatically collect or return money again. Preserve the original external reference, amount, action and staff attribution, and resolution evidence in a tenant-scoped recovery workflow. Its detailed state model, safe subsequent posting/compensation, and handling of drawer/shift closure must be specified before implementation.

No desktop checkout or read-only mobile-companion requirement is assumed. The choices above are agreed direction; detailed authorization, active-device handoff, reconciliation states, receipt layout, and platform acceptance still require specification and schema review in Milestone 0.

## Objectives

1. Provide a useful checkout and stock-control workflow for the first target business.
2. Separate reusable retail entities from future restaurant-specific workflows.
3. Use normalized PostgreSQL data, exact monetary calculations, and transactional posting.
4. Prevent duplicate sales, concurrent overselling, duplicate refunds, and shift/posting races.
5. Preserve price snapshots, stock history, authorization, and audit evidence.
6. Practice advanced SQL, indexing, performance measurement, and backup recovery against a concrete business workflow.

## Users and Permissions

* **Owner:** All manager/cashier capabilities plus staff provisioning, role changes, business settings, and register/payment-method configuration. The last active owner cannot be deactivated or demoted.

* **Manager:** Product/category management, stock receipts/adjustments, refund authorization/direct own-shift execution, store reports, and audit review. May operate checkout when using a shift opened under their own identity.

* **Cashier:** Manage their own draft sales, operate their own register shift, record checkout, reprint accessible receipts, and view their own shift activity. May execute only specifically Manager/Owner-authorized full refunds from their own shift. Cannot change catalog prices, adjust stock, self-authorize refunds, manage staff, or read store-wide reports/audit logs.

The only staff roles are Owner, Manager, and Cashier. Each staff user has one role. Owner/manager roles include cashier capabilities, but an open shift must belong to the actual operator. There is no public staff registration. First-owner creation is a controlled onboarding operation for each business. An authorized installation operator performs onboarding outside the business UI (the hosted-service operator or the business’s designated local administrator); this is an operational responsibility, not a fourth staff role. Owners can administer only their own business. Staff cannot reassign their account or business records to another tenant.

## Functional Requirements

### Module A — Staff and Store Setup

* **POS-FR-01 — Authentication:** Active staff shall log in/out securely, view their sessions, and revoke their own other sessions. Expired, revoked, or deactivated-user sessions shall be denied.

* **POS-FR-02 — Staff administration:** Owners shall create staff accounts, set roles, deactivate/reactivate staff, and revoke their sessions. Role/access changes require a reason, revoke affected sessions, preserve attribution, and cannot remove the last active owner. Initial credential delivery/reset is an owner-controlled setup procedure; public password recovery is deferred.

* **POS-FR-03 — Profile:** Staff shall view their profile, change their display name, and change their password after verifying the current password.

* **POS-FR-04 — Configuration:** Owners shall maintain the business display name, registers, and manual noncash payment-method names. Cash is a built-in method. Disabling a register/method prevents new use, preserves history, and does not prevent a refund using the original method. Registers cannot be disabled while their shift is open.

### Module B — Product Catalog and Inventory

* **POS-FR-05 — Catalog maintenance:** Managers/owners shall create and update categories and product names, SKUs and optional barcodes unique within their business, unit labels, selling prices, and low-stock thresholds, and activate/deactivate categories/products. Disabled categories shall prevent new product assignment; existing products shall retain their category and follow their own active status. Historical sale snapshots shall remain unchanged.

* **POS-FR-06 — Product lookup:** Cashiers shall search active products by SKU, barcode text, name, or category and see current price and on-hand quantity. Active products with zero stock shall remain visible as “Out of stock” with Add to cart disabled. Positive stock at or below the configured low-stock threshold shall show a warning (for example, “Only 2 left”) without preventing a valid purchase. Keyboard entry is sufficient initially; scanner/printer hardware integration is deferred.

* **POS-FR-07 — Stock receipts:** Managers/owners shall record positive whole-number stock receipts for one or more products with a delivery reference/note. Submission shall append receipt movements atomically and shall not create purchase/payable records.

* **POS-FR-08 — Stock adjustments:** Managers/owners shall record signed whole-number stock corrections with a required reason. An adjustment must not make stock negative and cannot rewrite prior movements.

* **POS-FR-09 — Stock visibility:** Authorized staff shall view current on-hand stock. Managers/owners shall view movement history and active products at or below their low-stock thresholds. Products start at zero until a receipt/adjustment is posted.

### Module C — Draft Sales and Checkout

* **POS-FR-10 — Draft sale:** Cashiers shall create a draft under their open shift, add/update/remove product lines, and see quantity, unit price, line total, and sale total. Duplicate selections of a SKU shall consolidate to one line. Draft creation/editing shall not post payment or stock movement. Quantity entry and Add/increase controls shall prevent the total cart quantity for a SKU from exceeding its latest known availability, with an explanatory message. Reducing quantity or removing the line remains possible. If refreshed stock drops below an existing cart quantity, flag the affected line and block checkout until the operator explicitly reduces/removes it; do not silently alter the cart. Refreshed replenishment shall update availability and re-enable valid actions.

* **POS-FR-11 — Server validation:** Checkout shall validate active staff, ownership of the draft/open shift, enabled register/payment method, active products, positive whole quantities, current prices, and sufficient stock. UI warnings and disabled controls are guidance, not reservations or authorization: the server shall recheck stock under transactional concurrency controls even if the client last showed availability. Rejection shall identify the affected lines and require correction without partial posting. The server shall calculate and validate base amounts, authorized price overrides, applicable discounts/promotions, taxes, and the final payable total using the reviewed pricing rules. A changed price/cart version shall require review and reconfirmation rather than silently charging a different amount.

* **POS-FR-12 — Cash checkout:** A cashier shall record cash tender at least equal to the total, calculate change, and apply exactly the sale total as the payment. Expected drawer cash increases by the sale total, not the tender before change.

* **POS-FR-13 — Manual noncash checkout:** A cashier shall select an enabled noncash method and record its external confirmation/reference after verifying payment outside Vault. Vault shall record the exact sale amount and confirmation; it shall not initiate or verify provider settlement automatically.

* **POS-FR-14 — Atomic completion:** Checkout shall atomically complete the sale, freeze its item/price/name snapshots, create exactly one payment, append the stock deductions, assign an immutable receipt number unique within the business, and finalize the operation outcome. A failed attempt shall leave the draft without payment or deductions.

* **POS-FR-15 — Retry safety:** Checkout and all other posting actions shall use staff-scoped idempotency keys and request fingerprints. The same key/payload shall return the existing operation/outcome; a different payload under that key shall conflict. Concurrent checkout attempts on one draft shall produce at most one completed sale/payment even with different keys.

* **POS-FR-16 — Receipt/history:** A cashier shall view/search receipts for their own shifts, including closed shifts, and view them on screen and download/share a PDF. Managers/owners shall access store-wide history. Completed receipts shall show business identity, receipt number, completion time, operator/register, item snapshots, quantities, original/effective prices, applied discounts/promotions, tax breakdown, final totals, payment method snapshot, cash tender/change when applicable, and any linked refund.

* **POS-FR-17 — Draft void:** A cashier shall void an owned draft with a reason, retaining its lines/history without posting stock/payment. A completed sale cannot be voided; correction requires a refund. Checkout and void shall serialize so only one succeeds.

### Module D — Full Refunds and Stock Returns

* **POS-FR-18 — Full-sale refund:** An operator shall request a full refund of a completed sale from their own current open shift. A Cashier requires a Manager/Owner’s separately authenticated approval bound to the exact sale, amount, restock decisions, reason, executing operator/shift, and request version; changes invalidate approval. A Manager/Owner operating their own shift may authorize and execute directly. Record approver, executor, reason, times, original payment method/paid total, and external refund reference for noncash. The executor handles the physical/provider refund; Vault records it. The executing shift bears the cash impact, never the approver’s unrelated shift. A closed original sale shift does not prevent refunding in a new shift.

* **POS-FR-19 — Restock decision:** A full refund shall include every original sale line and a restock decision per line. A restocked line restores its original sold quantity; a damaged/nonreturned line restores none. Inactive products may be restocked through a valid refund.

* **POS-FR-20 — Atomic refund:** Refund completion, unique refund reference, refund-line decisions, stock-restoration entries, audit event, and operation completion shall commit together. No second completed refund, over-refund, or completed-refund edit/delete shall be allowed. Failed attempts shall remain retryable with a deliberately new key.

### Module E — Register Shifts and Cash Reconciliation

* **POS-FR-21 — Open shift:** A cashier shall open an enabled register with a nonnegative opening cash float. A register and a staff member may each have at most one open shift. Opening shall not count as sales revenue. A shift has one active mobile control session; same-operator replacement-device resume atomically invalidates the old control session. Check that authority during posting, not only in the client UI.

* **POS-FR-22 — Shift activity:** A cashier shall view sales, refunds, and payment-method totals for their own shifts. Noncash activity shall remain separate from expected physical drawer cash.

* **POS-FR-23 — Close shift:** The shift operator shall close their open shift with counted cash. The system shall freeze expected cash, counted cash, variance, and closing time; nonzero variance requires an explanation. Closing shall reject unresolved processing checkout/refund operations and uncompleted draft sales; cashiers must complete or void drafts first. Subsequent operations cannot post into a closed shift.

* **POS-FR-24 — Shift review:** Managers/owners shall inspect closed-shift totals and variances. Initial shifts cannot be reopened or have their closing counts rewritten. Cash refunds require sufficient expected cash in the current drawer; noncash refunds do not reduce drawer cash.

### Module F — Reports, Integrity, and Audit

* **POS-FR-25 — Sales reports:** Managers/owners shall view daily/monthly gross completed sales, applied discounts/promotions, tax collected/refunded, refunds by refund date, net sales excluding tax, total collected/refunded, and units sold/returned, payment-method totals, and top products by sales amount/quantity. These are sales measures, not profit or complete business cash flow.

* **POS-FR-26 — History and export:** Accessible sale/refund/shift histories shall support reference, date, operator, register, and relevant status/method filters with stable pagination up to 100 rows. Authorized staff shall export the same accessible filtered data as CSV with safe formula-like text handling.

* **POS-FR-27 — Integrity report:** Managers/owners shall inspect mismatched sale totals, missing/duplicate payments, missing/incorrect sale or refund stock entries, invalid refund totals/lines, negative stock, and shift expected-cash mismatches. Reports shall use a consistent database snapshot.

* **POS-FR-28 — Audit trail:** Privileged actions and draft voids shall append actor/action/target/time/reason and safe before/after values. Covered changes include staff access/roles, settings, catalogs, stock receipts/adjustments, refunds, and shift closure/variance. Events shall commit atomically with the action and shall exclude credentials/session tokens.

* **POS-FR-29 — Posting outcomes:** Authorized staff shall inspect their operation status and safe rejection/failure reasons; managers/owners may inspect store-wide outcomes. Rejected business requests and failed execution attempts shall have no committed business postings. Interrupted/unknown responses shall be resolved by checking existing state rather than repeating collection/refund externally. Definitively rejected postings after confirmed external collection/refund require a tenant-scoped Manager/Owner reconciliation case with original references, amounts, actors, outcome links, and resolution evidence. Define the full lifecycle and cash/shift effects during planning; do not silently repeat or discard the external event.

### Module G — Business Onboarding and Isolation

* **POS-FR-30 — Controlled business onboarding:** An authorized installation operator shall provision a business, one settings record, its built-in cash method, and its first Owner as an all-or-nothing setup operation. Normal staff endpoints cannot create businesses or provision another business's staff. Onboarding outcomes shall retain a protected operational audit record without secrets; exact operational tooling is decided before implementation.

* **POS-FR-31 — Business-scoped access:** Every authenticated request shall use the business associated with the current staff account. Reads, writes, related-record selection, receipt/reference searches, reports, exports, audit review, and operation status/replay shall remain inside that business and its role/ownership permissions. Foreign-business IDs and tenant-field tampering shall be denied without exposing records or creating effects. Each client shall display the signed-in business identity.

### Module H — Pricing and Business Deletion

* **POS-FR-32 — Discounts:** Authorized staff shall apply eligible discounts to a sale using server-validated rules. Plan fixed/percentage options, line/sale scope, permissions, reasons, limits, allocation, rounding, and combination rules before implementation. Cashiers cannot invent unrestricted discounts.

* **POS-FR-33 — Tax computation:** Managers/Owners shall configure supported business tax rules; checkout shall calculate and snapshot the applied tax treatment and amounts. Review inclusive/exclusive pricing, rate precision, applicability/exemptions, calculation order, rounding, refunds, and receipt/report presentation. Tax computation is in scope; statutory invoicing/certification and a claim of automatic legal compliance remain outside scope.

* **POS-FR-34 — Promotions:** Managers/Owners shall configure promotions; checkout shall validate eligibility and apply the reviewed rules. Promotion types, qualifying products/quantities, effective periods/timezone, stacking/priority, changes during checkout, and refund behavior require feature/schema review. Do not assume every promotion type is supported.

* **POS-FR-35 — Checkout price overrides:** A Manager/Owner shall authorize a checkout price override with a reason and attributable evidence. Preserve catalog price and effective charged price separately; a cashier cannot self-authorize an override. Review whether authorization is direct or a separate approval handoff, how it binds to cart/version, and when edits invalidate approval.

* **POS-FR-36 — Owner-initiated business deletion:** Only an authorized Owner of that business shall initiate deletion after identity/ownership verification and explicit confirmation. Provide an export opportunity and explain effects on all staff, live records, operational history, retained exceptions, and backup copies. Plan pending work/open shifts, any cancellation/grace period, execution/status/retry safety, and session revocation. A restricted tenant-scoped purge path must not bypass other tenants’ isolation. Hosted operators and local installation administrators have deployment-specific execution/backup duties; an Owner’s request cannot imply automatic erasure of independent backup copies. Exact lifecycle, retention exceptions, and backup expiry/restore handling require review before implementation.

## Business Rules and Reporting Definitions

1. Completed sale total is the server-calculated payable amount after authorized pricing changes, allocated discounts/promotions, and applicable tax under the reviewed calculation rules. The old quantity × unit-price-only total is insufficient. Zero-priced products/sales remain deferred unless explicitly approved during pricing review. Snapshot SKU, name, unit label, catalog/effective price, discount/promotion/tax inputs and amounts, authorization evidence, and final line/sale totals; completed snapshots and payment-method labels remain immutable.
2. Current stock is the sum of committed signed inventory movements. Receipts/restocked refunds add units; completed sales subtract units; adjustments may add/subtract units. Drafts/voids/rejected/failed operations do not change stock. Every SKU must remain nonnegative.
3. Checkout, stock corrections/receipts, and refunds lock all affected product rows in a consistent order and recheck stock under that lock. Draft edits also use sale/version checks. A database row check alone cannot prevent concurrent overselling.
4. Refund amount equals the original sale total; no new pricing applies. Each original sale has at most one completed refund. Refund eligibility is independent of current product/payment-method enablement, but the operator/current register shift must be valid.
5. Expected drawer cash = opening float + completed cash sale amounts in that shift − completed cash refunds recorded in that shift. Counted cash and opening float must be nonnegative. Variance = counted − expected. Noncash activity, stock receipts, and stock corrections do not enter that formula.
6. Sale completion and refund completion use their actual posting timestamps, not draft creation time. Refunds reduce net sales on their refund date without rewriting prior periods. Gross sales still includes subsequently refunded original sales.
7. Pricing review must define gross sales, discounts/promotions, tax collected/refunded, net sales excluding tax, and total collected/refunded separately. The earlier gross-minus-refund-only reporting definition is a baseline pending that review; it must not label tax collected as sales revenue. Units sold and units returned are reported separately; restocked quantity is a separate stock measure. There is no profit/margin report without cost accounting.
8. Store/report currency is PHP and initial reporting timezone is Asia/Manila. Persist timezone-aware instants; convert inclusive calendar-date filters to half-open timestamp intervals. Months/days without activity appear as zero where relevant.
9. All posting actions (checkout, refund, receipt, adjustment, shift opening/closure, draft void) require a staff-scoped idempotency key across action types. Persist the immutable validated payload/fingerprint and a processing outcome before execution. Complete business effects and the terminal outcome atomically; on business rejection/execution failure retain the outcome without effects. Terminal outcomes do not transition again. A new attempt after a rejection/failure needs a new key.
10. A crash may leave a processing request; recovery must inspect committed state before resolving it. Posting uses a locked processing row. A timeout is not proof that payment/refund was unrecorded. Idempotency protects Vault records; manual external payment/refund handling requires operator verification and cannot promise provider-side exactly-once behavior.
11. Sale completion/refund, shift close, role deactivation, product changes, and register/method changes must coordinate locks so eligibility cannot change unnoticed midway. A processing checkout/refund operation accepted for a shift prevents closure until resolved. Lock order and bounded deadlock/serialization retries must be specified during implementation.
12. Deactivated staff cannot authenticate/start actions; existing attribution remains. Role changes revoke sessions and take effect on future actions. Before deactivating or changing the role of a staff member with an open shift, the owner must have that operator close it; emergency manager takeover is deferred.
13. Referenced products, categories, methods, registers, staff, sales, payments, refunds, shifts, inventory movements, and audit events are retained. Ordinary business actions cannot edit/delete posted movements, completed documents, payments, refunds, or audit history. POS-FR-36 is a distinct controlled whole-business deletion lifecycle; review its scoped purge privileges, retention exceptions, and backup behavior rather than allowing ordinary record deletion.

14. All business-owned records belong to exactly one tenant. Parent/child records must have the same tenant, enforced with composite foreign keys as well as application validation. Tenant ownership identifiers are immutable through normal staff actions.
15. SKUs, optional barcodes, category names, register codes, payment-method codes, receipt numbers, and refund references are unique within a business, not across businesses. Login emails and token hashes are unique across tenants within an installation; generated UUID primary keys remain unique within its database. Separate installations have independent identity and token namespaces. Idempotency keys are scoped to tenant and requester; last-owner and built-in cash-method rules apply per tenant.
16. Tenant isolation uses server-side scoping plus PostgreSQL row-level security (RLS) as defense in depth. Missing business context denies business-data access. Database roles, authentication lookup, connection reuse, background work, and privileged routines require explicit isolation design and verification before approval.

17. Both deployment modes use the same application code, PostgreSQL schema/migration history, tenant isolation, business rules, and role checks. A local database has one tenant; the hosted database supports several. No alternate embedded database or direct client-to-PostgreSQL access is included.
18. Each installation has one authoritative database. Client sessions, cached data, operation outcomes, and pending work cannot be carried automatically between independent installations. Internet failure alone does not stop store-LAN operation; local server/LAN failure does. External manual payment providers may still require internet access independently of Vault.

19. Pricing edits/configuration must not rewrite completed transactions. Full refunds return the actual original paid total, including its recorded pricing/tax effects; do not reprice using current rules. Review reporting allocations and refund snapshots before approving formulas or DBML.
20. Business deletion differs from staff deactivation and ordinary document deletion. No request may delete another tenant; the last-Owner rule remains for normal staff administration, while authorized whole-business deletion needs its own lifecycle. Do not claim immediate erasure from backups or permit a restore to silently reactivate a deleted business; review protected deletion evidence and restore procedures.

## Database Design Goals

Review [03-database-design-goals.md](03-database-design-goals.md) before approving the schema. Goals cover correctness/integrity, atomicity/isolation/durable outcomes, controlled scalability/flexibility, measured performance, maintainability/reproducibility, security/least privilege, backup/recovery/contingency planning, and traceability/privacy. Each goal names future verification evidence; proposed RPO/RTO and retention targets remain subject to planning review.

## Non-Functional Requirements

* **POS-NFR-01 — Integrity:** Use foreign keys, unique keys, local checks, and controlled posting routines. Enforce cross-row sale/payment/stock/refund invariants and append-only history using transactional logic and database privileges/triggers as appropriate.

* **POS-NFR-02 — Atomicity/concurrency:** Integration tests shall prove no partial checkout/refund survives errors; no stock overselling, duplicate sale/refund, last-owner removal, or posting after shift close can occur under competing requests.

* **POS-NFR-03 — Precision/security:** Use `numeric(20,2)` and decimal/integer-centavo arithmetic, reject excess precision/overflow, hash passwords and session tokens securely, enforce server permissions, rate-limit authentication, and never log secrets. Both hosted and business-local operational deployments must use protected transport; plain HTTP is limited to an explicitly controlled development exception. Plan local hostname/certificate trust and firewall rules before implementation.

* **POS-NFR-04 — Performance:** Proposed target: p95 server latency at most 500 ms for 100-row filtered sale-history requests over five minutes with 20 concurrent readers and at least 100,000 inventory movements. Document hardware, dataset distribution, query plans, cache conditions, throughput/errors, and whether the target was met.

* **POS-NFR-05 — Testability:** Use synthetic fixtures with known totals for selling, stock, returns, cash change, reconciliation, permissions, operation recovery, and concurrency. A diagram alone does not satisfy correctness.

* **POS-NFR-06 — Recovery:** Restore a backup into a separate database and verify row counts, completed sale/refund totals, stock balances, shift cash totals, and integrity reports. Document procedure and measured time.

* **POS-NFR-07 — Usability:** Checkout shall support keyboard-driven product lookup and clear quantity/price/payment errors. Unknown-response states must offer status lookup/retry of the same request without prompting another external payment.

* **POS-NFR-08 — Caching:** Redis is optional after measurement; authoritative stock, operation keys, permissions, and checkout/refund correctness stay in PostgreSQL.

* **POS-NFR-09 — Tenant isolation:** Verify at least two businesses with overlapping business identifiers and deliberately different totals. Test API, direct database access under the actual runtime role, cross-tenant relationship attempts, authentication/session boundaries, task/recovery execution, reports/exports, and connection-pool reuse. Runtime database access shall not own tables or have superuser/BYPASSRLS privileges. Missing/stale context must not expose another tenant.

* **POS-NFR-10 — Deployment portability:** Demonstrate both (a) desktop/mobile access to a non-AWS hosted multi-tenant installation and (b) desktop/mobile access over the LAN to a business-local single-tenant installation. Use the same code and schema with deployment-specific configuration. Document setup, protected transport/certificate trust, private database access, persistent storage, startup/restart, secrets, migrations, logs, health checks, rollback, backups/restore, onboarding, and responsible operators for each mode. Verify local selling during an internet outage and safe refusal when its server/LAN is unreachable. Provider and local-server platform selection remain pending; this requirement authorizes planning only.

## Planning Gate and Project Stages

Vault is a learning project. Milestones organize the planning, implementation, and verification work; they do not imply separate product releases. No application code, environment setup, dependency installation, migrations, or deployment begins until the Milestone 0 planning gate in [07-progress-checklist.md](07-progress-checklist.md) is complete and the user explicitly authorizes implementation. Draft documents and a passing DBML parser are not approval.

Before implementation, complete [feature specifications](02-feature-specifications.md), [architecture planning](05-architecture-planning.md), the [database design goals](03-database-design-goals.md) review, and the conceptual schema review. Proposed desktop/mobile scope and authentication choices must be resolved there.

## Delivery Milestones

1. **Design and foundation:** Finalize retail workflows and the [DBML schema](06-conceptual-schema.md); implement PostgreSQL migrations, permissions, tenant-aware catalog/setup seeds, baseline constraints, and RLS isolation.
2. **Core selling and stock:** Implement staff access, catalog, stock receipts/adjustments, shift opening, drafts, cash/noncash checkout, inventory posting, idempotency, and correctness/concurrency tests. Establish atomic audit writing before privileged actions.
3. **Returns and oversight:** Implement full refunds/restock choices, shift closure/reconciliation, accessible history/receipts/exports, sales/stock reports, audit review, and integrity queries with concurrency tests.
4. **Usability and operations:** Build the frontend around validated backend flows; benchmark at least 100,000 inventory movements, tune SQL/indexes, verify restore, and document reproducible setup/demo and demonstrate both the shared hosted service and a business-local installation.

## Completion Criteria

* [ ] A controlled setup creates two independent businesses and their first Owners; each Owner provisions only their own staff/catalog/registers without public role escalation.

* [ ] Businesses can reuse SKUs/register codes/receipt numbers while cross-business API access, foreign-key linking, reports/exports, and runtime-role database access remain isolated.

* [ ] Mobile and desktop demonstrate the same feature flows against both hosted and business-local APIs with the signed-in business clearly displayed; installation-specific deployment, startup/restart, and recovery evidence is recorded.

* [ ] Local cash selling continues with internet disconnected while the server/LAN is available; server/LAN loss blocks posting safely. Switching independent installations cannot reuse credentials/tokens or carry cached data/pending operations across servers.

* [ ] Cashier opens a shift, builds a cart, and completes cash/manual noncash sales with immutable receipts and accurate stock.

* [ ] Two cashiers competing for the last item cannot both sell it; retries cannot duplicate a sale/payment.

* [ ] Manager records one full refund with line-specific restock choices and correct current-shift cash impact.

* [ ] A shift closes with computed expected cash, counted cash, variance, and explanation, and rejects subsequent posting.

* [ ] Reports and integrity queries match known fixtures and enforce role visibility.

* [ ] Posted records and audit history resist unauthorized edits/deletion; only the reviewed Owner-initiated business deletion lifecycle can execute its permitted tenant-scoped removal.

* [ ] Discounts, supported promotions, taxes, and authorized price overrides yield the reviewed payable totals, immutable receipts, original-paid-value full refunds, and separate sales/discount/tax reporting.

* [ ] Business deletion verifies Owner identity/confirmation, offers export, handles active work and sessions safely, isolates other tenants, and reports live-data and backup/retention outcomes in both deployment modes.

* [ ] The documented performance benchmark and backup/restore verification run successfully.

* [ ] A fresh checkout can reproduce setup, tests, benchmark, and the hardware-store demo from the operating instructions in document 04.

## Outside the Learning-Project Scope

Add features only after a demonstrated workflow requires them: product variants, fractional units/package conversion, suppliers/purchasing, credit sales, split payments, integrated payments, multi-location stock, and restaurant tables/modifiers/kitchen/ingredient workflows. These features are outside the current project scope; listing them is not a promise of another release.
