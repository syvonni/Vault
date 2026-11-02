# Vault POS Conceptual Schema

Review position **06 of 07**. Review tables, fields, relationships, and invariants against the preceding planning decisions. Track this review in [GitHub issue #6](https://github.com/syvonni/Vault/issues/6). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

This model follows the [POS requirements](01-requirements.md) for shared hosted multi-tenant deployment and business-local single-tenant deployment using the same schema, one location per business, multiple registers, whole-unit stock, and manually recorded payments. It replaces the previous wallet/account schema. Roles are Owner, Manager, and Cashier; audit logs remain available to authorized Owner/Manager users. This schema is a draft: Milestone 0 must review it against feature specifications, database design goals, and the chosen Django identity/session approach before implementation.

The DBML includes logical keys, checks, and candidate indexes, but is not a complete PostgreSQL migration. Its reusable core is catalog → draft sale → completed sale/payment → inventory movement, with refunds and cashier shifts. There is no financial-account or general-ledger subsystem in this release.

## Modeling Decisions

- Each product is one sellable SKU. Unit labels are descriptive; quantities are whole numbers. Variants may later be modeled separately without adding a table for every business type.
- Inventory movements form an append-only quantity ledger. On-hand stock is their signed sum; product rows provide the stable lock target even before the first movement exists.
- Draft sales are editable and do not reserve inventory. Completed lines freeze product identifiers/names/prices; sale headers freeze receipt labels. Catalog edits do not rewrite receipts.
- A completed sale has one payment. Payment records describe cash or externally verified noncash collection; they do not cause a provider charge.
- A full refund is a new immutable document. Refund lines link all original sale lines and specify whether their full quantities return to stock. Failed attempts live in operation requests rather than occupying the successful-refund slot.
- Shift cash is derived from opening float plus cash sale amounts minus cash refunds posted in that shift. Closure stores the checked result, counted cash, and variance.
- Operation requests provide durable idempotency for all posting actions. Their payload/result JSON records request processing; normalized business tables remain authoritative for sales, refunds, and stock.

## Tenant Modeling Decisions

- `tenants` identifies each business. All other tables are business-owned and carry a non-null `tenant_id`; settings use it as their primary key, yielding at most one settings row per business. Controlled onboarding must create the required settings row.
- Staff accounts belong to one immutable tenant. Canonical email and token hashes stay unique across tenants within each installation; separate installations have independent identities/tokens. A person working for two businesses in the same installation would need separate accounts/emails in this limited scope. A membership/switching model is deferred.
- Business identifiers (SKU, optional barcode, category name, register/payment-method codes, receipt number, refund reference) are unique per tenant. UUID primary keys remain unique across tenants within each table, and `(tenant_id, id)` candidate keys support composite foreign keys.
- Every relationship between business records includes the tenant key, preventing cross-tenant links even if a UUID from another business is known. Nullable relationship columns use normal nullable composite-FK semantics; tenant ownership itself is never nullable. Audit polymorphic targets and IDs embedded in JSON also require same-tenant validation in the write path.
- All candidate access indexes lead with tenant context. The shared `tenants` registry exposes only the current business to normal staff; operational onboarding is separate. The last-active-Owner rule, exactly-one-cash-method rule, open-shift rules, and idempotency namespace apply per tenant.
- DBML cannot express the complete RLS/authentication privilege design. Before implementation, specify default-deny tenant policies, write checks, context lifecycle, privileged credential lookup, and runtime-role verification in the architecture/invariant mapping.

## Deployment Interpretation

This DBML is shared by both deployment modes. The hosted database contains several tenants; a business-local database contains one tenant with the same tenant keys, constraints, and planned RLS protections. Local deployment does not introduce an embedded database, a separate schema variant, or a database on each client.

Each installation has one authoritative PostgreSQL database and independent account/token records. Clients connect through that installation’s Django API. Cloud/local synchronization, automatic failover, and live business transfer/merge tooling are outside scope; no replication or synchronization tables are added.

## Planning Dependencies

Review this model against [database design goals](03-database-design-goals.md), [feature specifications](02-feature-specifications.md), and [architecture planning](05-architecture-planning.md). The current `staff_users`/`staff_sessions` layout expresses conceptual needs; Django user internals and the proposed token/session library require an explicit mapping before this becomes an approved logical schema. No migrations are authorized.

The sale `version` field is an edit counter for detecting stale carts, not a product release number. Database migrations later track schema changes within this one learning project.

## New Scope Gap — Pricing and Business Deletion

**The DBML below is the earlier baseline, not a complete model of POS-FR-32–POS-FR-36. Do not approve or implement it until these gaps are resolved.**

Pricing review must add the approved discount/promotion/tax configuration, effective periods/eligibility, catalog-versus-charged prices, calculation/rounding/allocation snapshots, override actor/reason/approval validity, and reconciled line/header totals. Existing `line_total = quantity * unit_price`, positive-price assumptions, receipt snapshots, and reporting formulas must be reviewed against those decisions; no new pricing tables or formulas are fabricated here before their semantics are agreed.

Business deletion review must model the approved request/lifecycle/confirmation/outcome and operational evidence, then map safe tenant-scoped purge ordering, privileges, active-work handling, retained exceptions, and backup expiry/restore handling. Current restrictive foreign keys and append-only rules protect normal operations; they are not a complete deletion design. Exact fields/tables and purge mechanisms are pending M0 review.

## Additional Schema Gap — Confirmed Client Workflows

**The DBML baseline also predates the confirmed cashier-execution/Manager-approval refund flow, active-device handoff, and external-money reconciliation.** Current refund/request/shift attribution alone is insufficient. Before approval, model same-tenant approver/executor separately, request content/version binding, authorization/expiry/revocation/claim/consumption and failed attempts, restricted refund lookup, one-active-device shift authority, and reconciliation records for actual external movement after rejected posting. Specify state transitions, race/retry/close-shift behavior, retained evidence, and enforcement mapping before choosing final fields/tables. No migrations are authorized.

## DBML

Paste this block into a current DBML-compatible diagram tool. Syntax: [official DBML documentation](https://dbml.dbdiagram.io/docs/). The DBML v2 parser supports the table-level checks used here.

```dbml
Project vault_pos {
  database_type: 'PostgreSQL'
  Note: 'Shared-schema multi-tenant retail POS; one location per business. Payments are recorded, not processed. See enforcement notes for cross-row and concurrency rules.'
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

Table tenants {
  id uuid [pk]
  slug varchar(80) [not null, unique, note: 'Stable service-wide business identifier; not an authorization credential.']
  created_at timestamptz [not null, default: `now()`]

  Note: 'One business with one physical location. Controlled onboarding creates settings, cash method, and first Owner atomically; no public signup.'
}

Table store_settings {
  tenant_id uuid [pk]
  display_name varchar(160) [not null]
  currency_code char(3) [not null, default: 'PHP']
  reporting_timezone varchar(60) [not null, default: 'Asia/Manila']
  updated_at timestamptz [not null, default: `now()`]

  checks {
    `currency_code = 'PHP'` [name: 'store_php_only']
    `reporting_timezone = 'Asia/Manila'` [name: 'initial_reporting_timezone']
  }

  Note: 'One settings row per business; controlled onboarding creates it.'
}

Table staff_users {
  id uuid [pk]
  tenant_id uuid [not null]
  email varchar(254) [not null, unique, note: 'Canonical normalized email.']
  display_name varchar(100) [not null]
  password_hash text [not null]
  role staff_role [not null]
  is_active boolean [not null, default: true]
  created_at timestamptz [not null, default: `now()`]
  updated_at timestamptz [not null, default: `now()`]

  Note: 'One immutable tenant and one role per staff account; email is globally unique. No public registration. Retain attribution after deactivation.'

  indexes {
    (tenant_id, id) [unique]
  }
}

Table staff_sessions {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, staff_user_id, created_at, id)
  }
}

Table categories {
  id uuid [pk]
  tenant_id uuid [not null]
  name varchar(100) [not null]
  is_active boolean [not null, default: true]

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, name) [unique]
  }
}

Table products {
  id uuid [pk]
  tenant_id uuid [not null]
  category_id uuid [not null]
  sku varchar(60) [not null]
  barcode varchar(80)
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
    (tenant_id, id) [unique]
    (tenant_id, category_id, is_active)
    (tenant_id, sku) [unique]
    (tenant_id, barcode) [unique]
  }

  Note: 'No editable stock column. Sum signed movements for stock; lock this row before validating/posting stock changes. Disabled categories prevent new product assignment; existing product eligibility follows product status.'
}

Table payment_methods {
  id uuid [pk]
  tenant_id uuid [not null]
  code varchar(40) [not null]
  name varchar(80) [not null]
  kind payment_kind [not null]
  is_enabled boolean [not null, default: true]

  Note: 'Seed exactly one built-in cash method per tenant; kind/code immutable. Other methods are manual noncash labels. Disabled methods still support recording a refund against an original payment.'

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, code) [unique]
  }
}

Table registers {
  id uuid [pk]
  tenant_id uuid [not null]
  code varchar(40) [not null]
  name varchar(80) [not null]
  is_enabled boolean [not null, default: true]

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, code) [unique]
  }
}

Table operation_requests {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, requested_by_user_id, idempotency_key) [unique]
    (tenant_id, target_shift_id, status, created_at)
    (tenant_id, target_sale_id)
    (tenant_id, status, created_at, id)
  }

  Note: 'Checkout/refund requests identify their target shift before posting, allowing close-shift to detect unresolved processing work. FK targets may be null for early business rejection or shift opening. Terminal outcomes are immutable.'
}

Table shifts {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, operator_user_id, opened_at, id)
    (tenant_id, register_id, opened_at, id)
  }

  Note: 'Partial unique indexes must permit at most one open shift per register and operator. Closing snapshots become immutable. Current expected cash is derived from recorded cash payments/refunds.'
}

Table sales {
  id uuid [pk]
  tenant_id uuid [not null]
  shift_id uuid [not null]
  status sale_status [not null, default: 'draft']
  version integer [not null, default: 1]
  total_amount numeric(20,2) [not null, default: 0]
  receipt_number varchar(50)
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
    (tenant_id, id) [unique]
    (tenant_id, shift_id, created_at, id)
    (tenant_id, status, completed_at, id)
    (tenant_id, receipt_number) [unique]
  }

  Note: 'One draft owner via its shift operator. Total must match line totals. Completed sale remains completed after refund; derive refunded indicator from the linked refund. No customer/credit-sale model initially.'
}

Table sale_items {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, sale_id, product_id) [unique]
    (tenant_id, product_id)
  }

  Note: 'Consolidate each SKU once per sale. Draft lines may change under sale lock/version checks; completed/voided document lines may not.'
}

Table payments {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, payment_method_id)
  }

  Note: 'Only a successfully completed sale has a payment, equal to sale total. Sale completion time supplies posting time. Reference is operator-entered evidence, not automatic settlement verification or a globally unique provider key.'
}

Table refunds {
  id uuid [pk]
  tenant_id uuid [not null]
  sale_id uuid [not null, unique]
  shift_id uuid [not null]
  operation_request_id uuid [not null, unique]
  reference varchar(50) [not null]
  amount numeric(20,2) [not null]
  reason text [not null]
  external_reference varchar(160)
  completed_at timestamptz [not null]

  checks {
    `amount > 0` [name: 'refund_positive_amount']
  }

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, shift_id, completed_at, id)
    (tenant_id, completed_at, id)
    (tenant_id, reference) [unique]
  }

  Note: 'Successful full refunds only. Amount equals original sale total; payment method comes from its original payment. Executor is the current shift operator/requester. Separate approval evidence is a pending schema extension. Refund-time product/method disablement does not block a valid return.'
}

Table refund_items {
  id uuid [pk]
  tenant_id uuid [not null]
  refund_id uuid [not null]
  sale_item_id uuid [not null, unique]
  restock boolean [not null]

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, refund_id, sale_item_id) [unique]
  }

  Note: 'Exactly one row for every original sale line. Quantity/value derive from immutable original line. Restock=true produces one movement restoring the entire sold quantity; false produces none.'
}

Table inventory_movements {
  id uuid [pk]
  tenant_id uuid [not null]
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
    (tenant_id, id) [unique]
    (tenant_id, product_id, posted_at, id)
    (tenant_id, operation_request_id, product_id) [unique]
  }

  Note: 'Append-only quantity ledger. On-hand is SUM(quantity_delta), accumulated as bigint. Cross-row validation must check linked line product/quantity, operation type/outcome, posting timestamps, and no negative product balance.'
}

Table audit_events {
  id uuid [pk]
  tenant_id uuid [not null]
  actor_user_id uuid [not null]
  action varchar(80) [not null]
  entity_type varchar(80) [not null]
  entity_id varchar(128) [not null]
  reason text
  before_values jsonb
  after_values jsonb
  created_at timestamptz [not null, default: `now()`]

  indexes {
    (tenant_id, id) [unique]
    (tenant_id, entity_type, entity_id, created_at, id)
    (tenant_id, actor_user_id, created_at, id)
  }

  Note: 'Append-only safe privileged/void/shift history, committed with the related action. Polymorphic target validated by the authorized write path; secrets are excluded.'
}

Ref: store_settings.tenant_id > tenants.id [delete: restrict]
Ref: staff_users.tenant_id > tenants.id [delete: restrict]
Ref: staff_sessions.tenant_id > tenants.id [delete: restrict]
Ref: categories.tenant_id > tenants.id [delete: restrict]
Ref: products.tenant_id > tenants.id [delete: restrict]
Ref: payment_methods.tenant_id > tenants.id [delete: restrict]
Ref: registers.tenant_id > tenants.id [delete: restrict]
Ref: operation_requests.tenant_id > tenants.id [delete: restrict]
Ref: shifts.tenant_id > tenants.id [delete: restrict]
Ref: sales.tenant_id > tenants.id [delete: restrict]
Ref: sale_items.tenant_id > tenants.id [delete: restrict]
Ref: payments.tenant_id > tenants.id [delete: restrict]
Ref: refunds.tenant_id > tenants.id [delete: restrict]
Ref: refund_items.tenant_id > tenants.id [delete: restrict]
Ref: inventory_movements.tenant_id > tenants.id [delete: restrict]
Ref: audit_events.tenant_id > tenants.id [delete: restrict]

Ref: staff_sessions.(tenant_id, staff_user_id) > staff_users.(tenant_id, id) [delete: restrict]
Ref: products.(tenant_id, category_id) > categories.(tenant_id, id) [delete: restrict]
Ref: operation_requests.(tenant_id, requested_by_user_id) > staff_users.(tenant_id, id) [delete: restrict]
Ref: operation_requests.(tenant_id, target_sale_id) > sales.(tenant_id, id) [delete: restrict]
Ref: operation_requests.(tenant_id, target_shift_id) > shifts.(tenant_id, id) [delete: restrict]
Ref: shifts.(tenant_id, register_id) > registers.(tenant_id, id) [delete: restrict]
Ref: shifts.(tenant_id, operator_user_id) > staff_users.(tenant_id, id) [delete: restrict]
Ref: shifts.(tenant_id, open_operation_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: shifts.(tenant_id, close_operation_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: sales.(tenant_id, shift_id) > shifts.(tenant_id, id) [delete: restrict]
Ref: sales.(tenant_id, completion_operation_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: sales.(tenant_id, void_operation_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: sale_items.(tenant_id, sale_id) > sales.(tenant_id, id) [delete: restrict]
Ref: sale_items.(tenant_id, product_id) > products.(tenant_id, id) [delete: restrict]
Ref: payments.(tenant_id, sale_id) > sales.(tenant_id, id) [delete: restrict]
Ref: payments.(tenant_id, payment_method_id) > payment_methods.(tenant_id, id) [delete: restrict]
Ref: refunds.(tenant_id, sale_id) > sales.(tenant_id, id) [delete: restrict]
Ref: refunds.(tenant_id, shift_id) > shifts.(tenant_id, id) [delete: restrict]
Ref: refunds.(tenant_id, operation_request_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: refund_items.(tenant_id, refund_id) > refunds.(tenant_id, id) [delete: restrict]
Ref: refund_items.(tenant_id, sale_item_id) > sale_items.(tenant_id, id) [delete: restrict]
Ref: inventory_movements.(tenant_id, product_id) > products.(tenant_id, id) [delete: restrict]
Ref: inventory_movements.(tenant_id, operation_request_id) > operation_requests.(tenant_id, id) [delete: restrict]
Ref: inventory_movements.(tenant_id, sale_item_id) > sale_items.(tenant_id, id) [delete: restrict]
Ref: inventory_movements.(tenant_id, refund_item_id) > refund_items.(tenant_id, id) [delete: restrict]
Ref: audit_events.(tenant_id, actor_user_id) > staff_users.(tenant_id, id) [delete: restrict]

TableGroup staff_and_setup {
  tenants
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

This baseline example has no discount, promotion, override, or tax applied; additional priced/taxed fixtures require M0-26 review.

All figures below belong to Business A. Business B may use the same SKU, register code, and receipt number while maintaining independent products, stock, shifts, and reports. No operation for A may link records from B.

Receive 10 hammers, each priced at PHP 150. Open a shift with PHP 1,000 float. Sell two hammers for PHP 300, accept PHP 500, and return PHP 200 change. Stock becomes 8; expected drawer cash becomes PHP 1,300.

The cashier requests a full PHP 300 cash refund in their still-open original shift, and a Manager approves from their own desktop session. The cashier executes; the refund records both identities and decreases that cashier’s expected cash from PHP 1,300 to PHP 1,000. Restocking both hammers restores stock to 10. If execution instead occurs in a later shift, its drawer bears the refund while the original closed shift remains unchanged. On that date the reports show gross sales PHP 300, refunds PHP 300, and net sales zero. If the returned line is not restocked, stock stays at 8. None of these figures states profit.

## Enforcement Beyond DBML

1. **Access and configuration:** Enforce active staff, role hierarchy, draft/shift ownership, the last active owner rule, and session revocation. Serialize concurrent owner changes using a stable tenant/settings/user lock protocol. Block access/role changes for users with open shifts; preserve attribution. Seed exactly one immutable enabled cash method per tenant using partial uniqueness and controlled setup. Category disablement affects new product assignment, not existing product eligibility. Validate nonblank names, reasons, and external references.
2. **Shift exclusivity:** Add PostgreSQL partial unique indexes on register and operator where shift status is open. Open/close requests have correct types, actors, and timestamps. Opening creates the shift and finalizes its request atomically. Never trust caller-supplied operator identities or cash totals.
3. **Drafts and snapshots:** Lock the sale on every edit/checkout/void and check its version. Add or update a SKU as a single line; increment version for each edit. Checkout compares the confirmed cart/version/prices with current locked product prices, revalidates eligibility, and freezes SKU/name/unit/price plus receipt business/operator/register labels. Compute totals with bounded decimal arithmetic. Prohibit edits/deletions of completed/voided headers/lines except no further business mutation; refunds are separate records.
4. **Posting and stock locks:** Specify a global lock order for authorization/settings, request, shift, sale, method/register, and ordered product rows, with bounded retries where necessary. Compute stock in a fresh statement after product locks at READ COMMITTED, or use a correctly retried alternative isolation strategy. The product row is the lock target even at zero stock. Price/activation edits, receipts, adjustments, checkout, and returns must coordinate with those locks. Posting must not overdraw stock.
5. **Atomic sale patterns:** At completion require at least one line, matching header/line totals, exactly one matching payment, and exactly one deduction per sold line with the same product and negative quantity. Amount/tender/change and method snapshots must match confirmed data. Sale, payment, movements, and successful operation commit together; no payment/movement survives a rejected/failed checkout. Deduction posted_at equals sale completed_at. A single completed draft is protected even if two request keys compete.
6. **Durable requests and recovery:** Uniquely claim tenant/requester/key and persist canonical payload/fingerprint before execution. Replays compare the hash and return the saved outcome; changed data conflicts. Business effects and a terminal successful outcome commit atomically under a locked processing request. After rollback, retain rejected/failed outcomes separately; allow a newly keyed deliberate attempt. Recovery competes for that same request lock and verifies committed state before resolving interrupted requests. Disable uncontrolled application writes that bypass this path.
7. **Request acceptance versus shift close:** Validate and attach checkout/refund requests to their current shift while holding that shift's lock during request acceptance. Closing takes the same lock and checks for other processing checkout/refund requests and remaining drafts. New acceptance/draft creation requires an open shift. Closure itself is excluded from the unresolved-work check. This prevents closing between request acceptance and execution. A processing request that has lost its client still needs recovery before closure.
8. **Full refunds:** Lock the original sale and current shift, verify executing-shift ownership, specific valid Manager/Owner approval (or authorized direct own-shift execution), and one full successful refund, and include exactly its full set of lines. Method derives from the original payment, amount from the sale; current pricing/enablement does not change them. Require externally verified reference for noncash and no external reference for cash. Cash refunds require enough expected current-shift cash. Post one positive stock movement per restocked line, none for non-restocked lines, with exact original quantity/product and refund completion time. Refund/header/lines/movements/audit/request outcome commit together. Unique refunds.sale_id excludes second successes while failed attempts remain in operation_requests.
9. **Cash reconciliation:** Close under shift lock, derive expected cash from opening float and completed cash sales/refunds, compare against nonnegative counted cash, compute variance, and require an explanation if nonzero. Close timestamp/cash snapshots/audit and request completion commit together. Refunds use their current shift, never rewrite the original shift, and never masquerade as a void or negative sale payment.
10. **Append-only posting:** Restrict runtime privileges and use controlled routines/deferred triggers as appropriate for cross-row completion checks. Outside the separately reviewed tenant-deletion path, no direct writes/deletes to payments, refunds, movements, closed shifts, completed documents, or audit events. Quantity receipts/adjustments also commit their movements/audit/successful request atomically. JSON request history is not an alternate editable inventory/sales store. Numeric scale checks at the API/routine boundary must reject excess decimals rather than relying on column rounding.
11. **Reporting and integrity:** Report sales by completed_at and refunds by their own completed_at; use payment kind snapshots and the original payment for refund method. Cashier scope is their own shifts; manager/owner store-wide reads differ from write permissions. Integrity SQL must check sale/line/payment equality, exact stock patterns, full refund coverage/value, no negative stock, operation/document links, and closed-shift cash snapshots against the derived formula. Use one consistent snapshot; as-of stock uses only movements posted at/before cutoff. Sales/refund filters should apply their respective posting times. Audit targets are intentionally polymorphic and need authorized target checks, not fictitious universal foreign keys.

12. **Tenant isolation:** Enable and force reviewed RLS policies on every business-owned table and restrict `tenants` visibility. Plan USING/WITH CHECK policies with server-derived transaction-local context; missing context denies access. The runtime role cannot own tables, bypass RLS, or access unrestricted provisioning/backup credentials. Define the narrow pre-authentication credential/token lookup separately. All request and background/recovery paths establish/clear validated context, including after exceptions and pooled-connection reuse. Composite foreign keys enforce same-tenant links; immutable tenant ownership, polymorphic audit targets, JSON references, and per-tenant last-Owner/cash-method rules need controlled checks too. Verify with two tenants and the real runtime role, not a migration owner. Sources: [PostgreSQL RLS](https://www.postgresql.org/docs/17/ddl-rowsecurity.html), [composite foreign keys](https://www.postgresql.org/docs/17/ddl-constraints.html#DDL-CONSTRAINTS-FK).

## Requirement Coverage

- **POS-FR-30–POS-FR-31:** tenants, tenant-keyed settings/staff and all business tables, same-tenant composite foreign keys, scoped uniqueness, and planned RLS/authentication enforcement.
- **POS-FR-01–POS-FR-04:** staff/settings/sessions/registers/payment methods, plus audit and server permissions.
- **POS-FR-05–POS-FR-09:** categories/products/inventory movements and operation requests.
- **POS-FR-10–POS-FR-17:** sales/items/payments/shifts and durable requests, including receipt snapshots and immutable completion.
- **POS-FR-18–POS-FR-20:** refunds/items, original payments, stock restoration, current shifts, and audit.
- **POS-FR-21–POS-FR-24:** shifts plus cash-payment/refund formulas and posting/closure coordination.
- **POS-FR-25–POS-FR-29:** derived SQL reports/histories/integrity queries, scoped exports, audit events, and operation outcomes.

Candidate indexes are starting points, not measured optimization claims. Full accounting, inventory valuation, suppliers/payables, restaurant extensions, and multiple locations do not have placeholder tables in this release.
