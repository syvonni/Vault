# Vault POS Conceptual Schema

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Review every entity and explain why it is needed for an approved feature.
- [ ] Review fields, types, nullability, primary/unique/foreign keys, and relationship cardinalities.
- [ ] Confirm Owner/Manager/Cashier identity/session requirements and the planned Django/auth-library mapping.
- [ ] Check normalization and justify stored sale/payment snapshots versus derived stock and cash totals.
- [ ] Walk through a sale, failed/retried checkout, refund/restock, stock correction, and shift closure using the proposed records.
- [ ] Map each business invariant to constraints, transactions/locks, routines, permissions, or integrity queries; verify retention/deletion rules.
- [ ] Complete the data dictionary and review candidate indexes against expected query patterns.
- [ ] Resolve schema questions and confirm it matches documents 01–05; parser success alone does not finalize the design.

Review position **06 of 07**. Review tables, fields, relationships, and invariants against the preceding planning decisions. See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

This model follows the [POS requirements](01-requirements.md) for one business at one location, multiple registers, whole-unit stock, and manually recorded payments. It replaces the previous wallet/account schema. Roles are Owner, Manager, and Cashier; audit logs remain available to authorized Owner/Manager users. This schema is a draft: Milestone 0 must review it against feature specifications, database design goals, and the chosen Django identity/session approach before implementation.

The DBML includes logical keys, checks, and candidate indexes, but is not a complete PostgreSQL migration. Its reusable core is catalog → draft sale → completed sale/payment → inventory movement, with refunds and cashier shifts. There is no financial-account or general-ledger subsystem in this release.

## Modeling Decisions

- Each product is one sellable SKU. Unit labels are descriptive; quantities are whole numbers. Variants may later be modeled separately without adding a table for every business type.
- Inventory movements form an append-only quantity ledger. On-hand stock is their signed sum; product rows provide the stable lock target even before the first movement exists.
- Draft sales are editable and do not reserve inventory. Completed lines freeze product identifiers/names/prices; sale headers freeze receipt labels. Catalog edits do not rewrite receipts.
- A completed sale has one payment. Payment records describe cash or externally verified noncash collection; they do not cause a provider charge.
- A full refund is a new immutable document. Refund lines link all original sale lines and specify whether their full quantities return to stock. Failed attempts live in operation requests rather than occupying the successful-refund slot.
- Shift cash is derived from opening float plus cash sale amounts minus cash refunds posted in that shift. Closure stores the checked result, counted cash, and variance.
- Operation requests provide durable idempotency for all posting actions. Their payload/result JSON records request processing; normalized business tables remain authoritative for sales, refunds, and stock.

## Planning Dependencies

Review this model against [database design goals](03-database-design-goals.md), [feature specifications](02-feature-specifications.md), and [architecture planning](05-architecture-planning.md). The current `staff_users`/`staff_sessions` layout expresses conceptual needs; Django user internals and the proposed token/session library require an explicit mapping before this becomes an approved logical schema. No migrations are authorized.

The sale `version` field is an edit counter for detecting stale carts, not a product release number. Database migrations later track schema changes within this one learning project.

## DBML

Paste this block into a current DBML-compatible diagram tool. Syntax: [official DBML documentation](https://dbml.dbdiagram.io/docs/). The DBML v2 parser supports the table-level checks used here.

```dbml
Project vault_pos {
  database_type: 'PostgreSQL'
  Note: 'Single-location retail POS. Payments are recorded, not processed. See enforcement notes for cross-row and concurrency rules.'
}

Enum staff_role {
  owner
  manager
  cashier
}

Enum payment_kind {
  cash
  manual_noncash
}

Enum shift_status {
  open
  closed
}

Enum sale_status {
  draft
  completed
  voided
}

Enum operation_type {
  checkout
  refund
  stock_receipt
  stock_adjustment
  shift_open
  shift_close
  void_sale
}

Enum operation_status {
  processing
  completed
  rejected
  failed
}

Enum inventory_movement_kind {
  receipt
  adjustment
  sale
  refund
}

Table store_settings {
  id smallint [pk, note: 'Singleton row with id = 1.']
  display_name varchar(160) [not null]
  currency_code char(3) [not null, default: 'PHP']
  reporting_timezone varchar(60) [not null, default: 'Asia/Manila']
  updated_at timestamptz [not null, default: `now()`]

  checks {
    `id = 1` [name: 'single_store_settings']
    `currency_code = 'PHP'` [name: 'store_php_only']
    `reporting_timezone = 'Asia/Manila'` [name: 'initial_reporting_timezone']
  }
}

Table staff_users {
  id uuid [pk]
  email varchar(254) [not null, unique, note: 'Canonical normalized email.']
  display_name varchar(100) [not null]
  password_hash text [not null]
  role staff_role [not null]
  is_active boolean [not null, default: true]
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  Note: 'One role per staff member initially. No public registration. Retain attribution after deactivation.'
}

Table staff_sessions {
  id uuid [pk]
  staff_user_id uuid [not null]
  token_hash varchar(255) [not null, unique]
  client_description text
  created_at timestamptz [not null, default: `now()`]
  expires_at timestamptz [not null]
  revoked_at timestamptz

  checks {
    `expires_at > created_at` [name: 'session_expiry_after_creation']
    `revoked_at IS NULL OR revoked_at >= created_at` [name: 'session_revocation_after_creation']
  }

  indexes {
    (staff_user_id, created_at, id)
  }
}

Table categories {
  id uuid [pk]
  name varchar(100) [not null, unique]
  is_active boolean [not null, default: true]
}

Table products {
  id uuid [pk]
  category_id uuid [not null]
  sku varchar(60) [not null, unique]
  barcode varchar(80) [unique]
  name varchar(160) [not null]
  unit_label varchar(40) [not null, note: 'Descriptive whole-package label such as piece or box; no unit conversion.']
  selling_price numeric(20,2) [not null]
  low_stock_threshold integer [not null, default: 0]
  is_active boolean [not null, default: true]
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  checks {
    `selling_price > 0` [name: 'product_positive_price']
    `low_stock_threshold >= 0` [name: 'product_nonnegative_threshold']
  }

  indexes {
    (category_id, is_active)
  }

  Note: 'No editable stock column. Sum signed movements for stock; lock this row before validating/posting stock changes. Disabled categories prevent new product assignment; existing product eligibility follows product status.'
}

Table payment_methods {
  id uuid [pk]
  code varchar(40) [not null, unique]
  name varchar(80) [not null]
  kind payment_kind [not null]
  is_enabled boolean [not null, default: true]

  Note: 'Seed exactly one built-in cash method; kind/code immutable. Other methods are manual noncash labels. Disabled methods still support recording a refund against an original payment.'
}

Table registers {
  id uuid [pk]
  code varchar(40) [not null, unique]
  name varchar(80) [not null]
  is_enabled boolean [not null, default: true]
}

Table operation_requests {
  id uuid [pk]
  requested_by_user_id uuid [not null]
  idempotency_key varchar(128) [not null]
  request_fingerprint char(64) [not null]
  type operation_type [not null]
  status operation_status [not null, default: 'processing']
  target_sale_id uuid
  target_shift_id uuid
  request_payload jsonb [not null, note: 'Immutable validated canonical action data, including expected sale version/prices where applicable. No credentials.']
  result_payload jsonb [note: 'Safe stored response/created references, not a substitute for normalized business data.']
  outcome_code varchar(80)
  created_at timestamptz [not null, default: `now()`]
  finalized_at timestamptz

  checks {
    `((status = 'processing' AND finalized_at IS NULL) OR (status <> 'processing' AND finalized_at IS NOT NULL))` [name: 'operation_finalization_state']
    `finalized_at IS NULL OR finalized_at >= created_at` [name: 'operation_finalized_after_creation']
    `status NOT IN ('rejected', 'failed') OR outcome_code IS NOT NULL` [name: 'operation_failure_reason']
  }

  indexes {
    (requested_by_user_id, idempotency_key) [unique]
    (target_shift_id, status, created_at)
    target_sale_id
    (status, created_at, id)
  }

  Note: 'Checkout/refund requests identify their target shift before posting, allowing close-shift to detect unresolved processing work. FK targets may be null for early business rejection or shift opening. Terminal outcomes are immutable.'
}

Table shifts {
  id uuid [pk]
  register_id uuid [not null]
  operator_user_id uuid [not null]
  status shift_status [not null, default: 'open']
  opening_float numeric(20,2) [not null]
  opened_at timestamptz [not null]
  open_operation_id uuid [not null, unique]
  closed_at timestamptz
  close_operation_id uuid [unique]
  expected_closing_cash numeric(20,2)
  counted_closing_cash numeric(20,2)
  cash_variance numeric(20,2)
  variance_reason text

  checks {
    `opening_float >= 0` [name: 'shift_nonnegative_float']
    `((status = 'open' AND closed_at IS NULL AND close_operation_id IS NULL AND expected_closing_cash IS NULL AND counted_closing_cash IS NULL AND cash_variance IS NULL) OR (status = 'closed' AND closed_at IS NOT NULL AND close_operation_id IS NOT NULL AND expected_closing_cash IS NOT NULL AND counted_closing_cash IS NOT NULL AND cash_variance IS NOT NULL))` [name: 'shift_closure_fields']
    `expected_closing_cash IS NULL OR expected_closing_cash >= 0` [name: 'shift_nonnegative_expected_cash']
    `counted_closing_cash IS NULL OR counted_closing_cash >= 0` [name: 'shift_nonnegative_counted_cash']
    `cash_variance IS NULL OR cash_variance = counted_closing_cash - expected_closing_cash` [name: 'shift_variance_formula']
    `cash_variance IS NULL OR cash_variance = 0 OR variance_reason IS NOT NULL` [name: 'shift_variance_reason']
    `closed_at IS NULL OR closed_at >= opened_at` [name: 'shift_closed_after_opening']
  }

  indexes {
    (operator_user_id, opened_at, id)
    (register_id, opened_at, id)
  }

  Note: 'Partial unique indexes must permit at most one open shift per register and operator. Closing snapshots become immutable. Current expected cash is derived from recorded cash payments/refunds.'
}

Table sales {
  id uuid [pk]
  shift_id uuid [not null]
  status sale_status [not null, default: 'draft']
  version integer [not null, default: 1]
  total_amount numeric(20,2) [not null, default: 0]
  receipt_number varchar(50) [unique]
  receipt_business_name varchar(160)
  receipt_operator_name varchar(100)
  receipt_register_name varchar(80)
  completion_operation_id uuid [unique]
  void_operation_id uuid [unique]
  void_reason text
  created_at timestamptz [not null, default: `now()`]
  completed_at timestamptz
  voided_at timestamptz

  checks {
    `version > 0` [name: 'sale_positive_version']
    `total_amount >= 0` [name: 'sale_nonnegative_total']
    `((status = 'completed' AND completed_at IS NOT NULL AND completion_operation_id IS NOT NULL AND receipt_number IS NOT NULL AND total_amount > 0 AND receipt_business_name IS NOT NULL AND receipt_operator_name IS NOT NULL AND receipt_register_name IS NOT NULL) OR (status <> 'completed' AND completed_at IS NULL AND completion_operation_id IS NULL AND receipt_number IS NULL))` [name: 'sale_completion_fields']
    `((status = 'voided' AND voided_at IS NOT NULL AND void_operation_id IS NOT NULL AND void_reason IS NOT NULL) OR (status <> 'voided' AND voided_at IS NULL AND void_operation_id IS NULL AND void_reason IS NULL))` [name: 'sale_void_fields']
    `completed_at IS NULL OR completed_at >= created_at` [name: 'sale_completed_after_creation']
    `voided_at IS NULL OR voided_at >= created_at` [name: 'sale_voided_after_creation']
  }

  indexes {
    (shift_id, created_at, id)
    (status, completed_at, id)
  }

  Note: 'One draft owner via its shift operator. Total must match line totals. Completed sale remains completed after refund; derive refunded indicator from the linked refund. No customer/credit-sale model initially.'
}

Table sale_items {
  id uuid [pk]
  sale_id uuid [not null]
  product_id uuid [not null]
  sku_snapshot varchar(60) [not null]
  name_snapshot varchar(160) [not null]
  unit_label_snapshot varchar(40) [not null]
  quantity integer [not null]
  unit_price numeric(20,2) [not null]
  line_total numeric(20,2) [not null]

  checks {
    `quantity > 0` [name: 'sale_item_positive_quantity']
    `unit_price > 0` [name: 'sale_item_positive_price']
    `line_total = quantity * unit_price` [name: 'sale_item_total_formula']
  }

  indexes {
    (sale_id, product_id) [unique]
    product_id
  }

  Note: 'Consolidate each SKU once per sale. Draft lines may change under sale lock/version checks; completed/voided document lines may not.'
}

Table payments {
  id uuid [pk]
  sale_id uuid [not null, unique]
  payment_method_id uuid [not null]
  method_name_snapshot varchar(80) [not null]
  kind_snapshot payment_kind [not null]
  amount numeric(20,2) [not null]
  cash_tendered numeric(20,2)
  cash_change numeric(20,2)
  external_reference varchar(160)

  checks {
    `amount > 0` [name: 'payment_positive_amount']
    `((kind_snapshot = 'cash' AND cash_tendered IS NOT NULL AND cash_tendered >= amount AND cash_change IS NOT NULL AND cash_change = cash_tendered - amount AND external_reference IS NULL) OR (kind_snapshot = 'manual_noncash' AND cash_tendered IS NULL AND cash_change IS NULL AND external_reference IS NOT NULL))` [name: 'payment_method_fields']
  }

  indexes {
    payment_method_id
  }

  Note: 'Only a successfully completed sale has a payment, equal to sale total. Sale completion time supplies posting time. Reference is operator-entered evidence, not automatic settlement verification or a globally unique provider key.'
}

Table refunds {
  id uuid [pk]
  sale_id uuid [not null, unique]
  shift_id uuid [not null]
  operation_request_id uuid [not null, unique]
  reference varchar(50) [not null, unique]
  amount numeric(20,2) [not null]
  reason text [not null]
  external_reference varchar(160)
  completed_at timestamptz [not null]

  checks {
    `amount > 0` [name: 'refund_positive_amount']
  }

  indexes {
    (shift_id, completed_at, id)
    (completed_at, id)
  }

  Note: 'Successful full refunds only. Amount equals original sale total; payment method comes from its original payment. Actor is the current shift operator/requester. Refund-time product/method disablement does not block a valid return.'
}

Table refund_items {
  id uuid [pk]
  refund_id uuid [not null]
  sale_item_id uuid [not null, unique]
  restock boolean [not null]

  indexes {
    (refund_id, sale_item_id) [unique]
  }

  Note: 'Exactly one row for every original sale line. Quantity/value derive from immutable original line. Restock=true produces one movement restoring the entire sold quantity; false produces none.'
}

Table inventory_movements {
  id uuid [pk]
  product_id uuid [not null]
  operation_request_id uuid [not null]
  kind inventory_movement_kind [not null]
  quantity_delta integer [not null]
  sale_item_id uuid [unique]
  refund_item_id uuid [unique]
  note text
  posted_at timestamptz [not null]

  checks {
    `quantity_delta <> 0` [name: 'inventory_nonzero_movement']
    `((kind = 'sale' AND quantity_delta < 0 AND sale_item_id IS NOT NULL AND refund_item_id IS NULL) OR (kind = 'refund' AND quantity_delta > 0 AND refund_item_id IS NOT NULL AND sale_item_id IS NULL) OR (kind = 'receipt' AND quantity_delta > 0 AND sale_item_id IS NULL AND refund_item_id IS NULL AND note IS NOT NULL) OR (kind = 'adjustment' AND sale_item_id IS NULL AND refund_item_id IS NULL AND note IS NOT NULL))` [name: 'inventory_movement_shape']
  }

  indexes {
    (product_id, posted_at, id)
    (operation_request_id, product_id) [unique]
  }

  Note: 'Append-only quantity ledger. On-hand is SUM(quantity_delta), accumulated as bigint. Cross-row validation must check linked line product/quantity, operation type/outcome, posting timestamps, and no negative product balance.'
}

Table audit_events {
  id uuid [pk]
  actor_user_id uuid [not null]
  action varchar(80) [not null]
  entity_type varchar(80) [not null]
  entity_id varchar(128) [not null]
  reason text
  before_values jsonb
  after_values jsonb
  created_at timestamptz [not null, default: `now()`]

  indexes {
    (entity_type, entity_id, created_at, id)
    (actor_user_id, created_at, id)
  }

  Note: 'Append-only safe privileged/void/shift history, committed with the related action. Polymorphic target validated by the authorized write path; secrets are excluded.'
}

Ref: staff_sessions.staff_user_id > staff_users.id [delete: restrict]
Ref: products.category_id > categories.id [delete: restrict]
Ref: operation_requests.requested_by_user_id > staff_users.id [delete: restrict]
Ref: operation_requests.target_sale_id > sales.id [delete: restrict]
Ref: operation_requests.target_shift_id > shifts.id [delete: restrict]
Ref: shifts.register_id > registers.id [delete: restrict]
Ref: shifts.operator_user_id > staff_users.id [delete: restrict]
Ref: shifts.open_operation_id > operation_requests.id [delete: restrict]
Ref: shifts.close_operation_id > operation_requests.id [delete: restrict]
Ref: sales.shift_id > shifts.id [delete: restrict]
Ref: sales.completion_operation_id > operation_requests.id [delete: restrict]
Ref: sales.void_operation_id > operation_requests.id [delete: restrict]
Ref: sale_items.sale_id > sales.id [delete: restrict]
Ref: sale_items.product_id > products.id [delete: restrict]
Ref: sales.id - payments.sale_id [delete: restrict]
Ref: payments.payment_method_id > payment_methods.id [delete: restrict]
Ref: sales.id - refunds.sale_id [delete: restrict]
Ref: refunds.shift_id > shifts.id [delete: restrict]
Ref: refunds.operation_request_id > operation_requests.id [delete: restrict]
Ref: refund_items.refund_id > refunds.id [delete: restrict]
Ref: refund_items.sale_item_id > sale_items.id [delete: restrict]
Ref: inventory_movements.product_id > products.id [delete: restrict]
Ref: inventory_movements.operation_request_id > operation_requests.id [delete: restrict]
Ref: inventory_movements.sale_item_id > sale_items.id [delete: restrict]
Ref: inventory_movements.refund_item_id > refund_items.id [delete: restrict]
Ref: audit_events.actor_user_id > staff_users.id [delete: restrict]

TableGroup staff_and_setup {
  store_settings
  staff_users
  staff_sessions
  registers
  payment_methods
}

TableGroup catalog_and_stock {
  categories
  products
  inventory_movements
}

TableGroup selling_and_returns {
  shifts
  sales
  sale_items
  payments
  refunds
  refund_items
}

TableGroup operations_and_audit {
  operation_requests
  audit_events
}
```

## Worked Example

Receive 10 hammers, each priced at PHP 150. Open a shift with PHP 1,000 float. Sell two hammers for PHP 300, accept PHP 500, and return PHP 200 change. Stock becomes 8; expected drawer cash becomes PHP 1,300.

A manager opens their own shift at another register with PHP 500 float and records a full PHP 300 cash refund. Restocking both hammers restores stock to 10. The original sale shift still expects PHP 1,300; the refund shift now expects PHP 200. On that date the reports show gross sales PHP 300, refunds PHP 300, and net sales zero. If the returned line is not restocked, stock stays at 8. None of these figures states profit.

## Enforcement Beyond DBML

1. **Access and configuration:** Enforce active staff, role hierarchy, draft/shift ownership, the last active owner rule, and session revocation. Serialize concurrent owner changes using a stable settings/user lock protocol. Block access/role changes for users with open shifts; preserve attribution. Seed exactly one immutable enabled cash method using partial uniqueness and controlled setup. Category disablement affects new product assignment, not existing product eligibility. Validate nonblank names, reasons, and external references.
2. **Shift exclusivity:** Add PostgreSQL partial unique indexes on register and operator where shift status is open. Open/close requests have correct types, actors, and timestamps. Opening creates the shift and finalizes its request atomically. Never trust caller-supplied operator identities or cash totals.
3. **Drafts and snapshots:** Lock the sale on every edit/checkout/void and check its version. Add or update a SKU as a single line; increment version for each edit. Checkout compares the confirmed cart/version/prices with current locked product prices, revalidates eligibility, and freezes SKU/name/unit/price plus receipt business/operator/register labels. Compute totals with bounded decimal arithmetic. Prohibit edits/deletions of completed/voided headers/lines except no further business mutation; refunds are separate records.
4. **Posting and stock locks:** Specify a global lock order for authorization/settings, request, shift, sale, method/register, and ordered product rows, with bounded retries where necessary. Compute stock in a fresh statement after product locks at READ COMMITTED, or use a correctly retried alternative isolation strategy. The product row is the lock target even at zero stock. Price/activation edits, receipts, adjustments, checkout, and returns must coordinate with those locks. Posting must not overdraw stock.
5. **Atomic sale patterns:** At completion require at least one line, matching header/line totals, exactly one matching payment, and exactly one deduction per sold line with the same product and negative quantity. Amount/tender/change and method snapshots must match confirmed data. Sale, payment, movements, and successful operation commit together; no payment/movement survives a rejected/failed checkout. Deduction posted_at equals sale completed_at. A single completed draft is protected even if two request keys compete.
6. **Durable requests and recovery:** Uniquely claim requester/key and persist canonical payload/fingerprint before execution. Replays compare the hash and return the saved outcome; changed data conflicts. Business effects and a terminal successful outcome commit atomically under a locked processing request. After rollback, retain rejected/failed outcomes separately; allow a newly keyed deliberate attempt. Recovery competes for that same request lock and verifies committed state before resolving interrupted requests. Disable uncontrolled application writes that bypass this path.
7. **Request acceptance versus shift close:** Validate and attach checkout/refund requests to their current shift while holding that shift's lock during request acceptance. Closing takes the same lock and checks for other processing checkout/refund requests and remaining drafts. New acceptance/draft creation requires an open shift. Closure itself is excluded from the unresolved-work check. This prevents closing between request acceptance and execution. A processing request that has lost its client still needs recovery before closure.
8. **Full refunds:** Lock the original sale and current shift, verify role/ownership and one full successful refund, and include exactly its full set of lines. Method derives from the original payment, amount from the sale; current pricing/enablement does not change them. Require externally verified reference for noncash and no external reference for cash. Cash refunds require enough expected current-shift cash. Post one positive stock movement per restocked line, none for non-restocked lines, with exact original quantity/product and refund completion time. Refund/header/lines/movements/audit/request outcome commit together. Unique refunds.sale_id excludes second successes while failed attempts remain in operation_requests.
9. **Cash reconciliation:** Close under shift lock, derive expected cash from opening float and completed cash sales/refunds, compare against nonnegative counted cash, compute variance, and require an explanation if nonzero. Close timestamp/cash snapshots/audit and request completion commit together. Refunds use their current shift, never rewrite the original shift, and never masquerade as a void or negative sale payment.
10. **Append-only posting:** Restrict runtime privileges and use controlled routines/deferred triggers as appropriate for cross-row completion checks. No direct writes/deletes to payments, refunds, movements, closed shifts, completed documents, or audit events. Quantity receipts/adjustments also commit their movements/audit/successful request atomically. JSON request history is not an alternate editable inventory/sales store. Numeric scale checks at the API/routine boundary must reject excess decimals rather than relying on column rounding.
11. **Reporting and integrity:** Report sales by completed_at and refunds by their own completed_at; use payment kind snapshots and the original payment for refund method. Cashier scope is their own shifts; manager/owner store-wide reads differ from write permissions. Integrity SQL must check sale/line/payment equality, exact stock patterns, full refund coverage/value, no negative stock, operation/document links, and closed-shift cash snapshots against the derived formula. Use one consistent snapshot; as-of stock uses only movements posted at/before cutoff. Sales/refund filters should apply their respective posting times. Audit targets are intentionally polymorphic and need authorized target checks, not fictitious universal foreign keys.

## Requirement Coverage

- **POS-FR-01–POS-FR-04:** staff/settings/sessions/registers/payment methods, plus audit and server permissions.
- **POS-FR-05–POS-FR-09:** categories/products/inventory movements and operation requests.
- **POS-FR-10–POS-FR-17:** sales/items/payments/shifts and durable requests, including receipt snapshots and immutable completion.
- **POS-FR-18–POS-FR-20:** refunds/items, original payments, stock restoration, current shifts, and audit.
- **POS-FR-21–POS-FR-24:** shifts plus cash-payment/refund formulas and posting/closure coordination.
- **POS-FR-25–POS-FR-29:** derived SQL reports/histories/integrity queries, scoped exports, audit events, and operation outcomes.

Candidate indexes are starting points, not measured optimization claims. Full accounting, inventory valuation, suppliers/payables, restaurant extensions, and multiple locations do not have placeholder tables in this release.
