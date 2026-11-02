# Vault Database Design Goals

Review position **03 of 07**. Agree on measurable database quality and recovery goals before finalizing the design. Track this review in [GitHub issue #3](https://github.com/syvonni/Vault/issues/3). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Status: planning draft for Vault as a learning project. These are design goals with review/verification evidence, not claims that the system already meets them. Review them in checklist M0-09–M0-11 before implementation.

## 1. Correctness, Consistency, and Integrity

Sales, payments, refunds, stock, and shift totals must agree. Define valid states and enforce them with primary/foreign keys, uniqueness, local checks, controlled posting routines, and transactions. A foreign key alone cannot enforce a balanced business operation or prevent concurrent overselling.

Evidence: a sale's lines equal its total/payment; exact per-line stock changes; no negative stock; one full successful refund per sale; immutable completed records; concurrent last-item tests; integrity queries that detect isolated corruption. Links: POS-NFR-01, POS-NFR-02, POS-FR-14–POS-FR-15, POS-FR-20, POS-FR-27.

## 2. Atomicity, Isolation, and Durable Outcomes

A checkout/refund either commits all its related effects or none. Deterministic product/shift/sale locks serialize competing actions. Durable operation keys let clients resolve retries and missing responses without posting twice. Protect committed work through PostgreSQL durability and verified recovery, rather than assuming a backup makes every recent commit recoverable.

Evidence: rollback injection; duplicate/reordered request tests; checkout-versus-close races; crash/response-loss scenarios; documented lock order/isolation/retry policy. Durability settings and any hosting limitations must be recorded. Links: POS-NFR-02, POS-FR-15, POS-FR-29.

## 3. Scalability and Controlled Flexibility

Use normalized retail entities and catalog rows so more SKUs, categories, registers, sales, and movements do not require a table per product/business type. Keep the supported single-location, whole-unit model explicit. Support a shared multi-tenant hosted installation and a single-tenant business-local installation with the same PostgreSQL schema; each business has one location. Use tenant-leading indexes and tenant-scoped business uniqueness; sharding, arbitrary custom fields, and microservices remain outside scope.

Evidence: at least 100,000 valid inventory movements, multiple concurrent registers, distribution-aware datasets, and query plans with skewed data across at least two tenants. Document which changes would require a deliberate schema extension. Links: POS-NFR-04, POS-FR-05–POS-FR-09.

## 4. Performance and Efficient Access

Use pagination, selective indexes, appropriate joins/aggregates, and measured plans. A faster query must still return correct totals and enforce permissions. Indexes have storage/write costs. Derived stock is authoritative; add caches only after measuring a bottleneck.

Evidence: proposed history p95 at most 500 ms for five minutes with 20 concurrent readers, pages up to 100 rows, and at least 100,000 inventory movements. Record hardware/cache conditions/errors and before/after SQL plans; benchmark reporting too without inventing unmeasured response-time claims. Links: POS-NFR-04, POS-NFR-08.

## 5. Maintainability and Reproducibility

Keep a data dictionary, meaningful names, documented ownership/cardinality, normalized source records, justified immutable snapshots, one migration authority, and reproducible synthetic fixtures. Keep secrets and generated backups out of source control. Track schema changes within this one project; database migrations are not additional product releases.

Both modes use the same migration authority and business rules. Maintain deployment-specific configuration/runbooks instead of forking the data model. Each installation is authoritative; no cloud/local synchronization or merge is implied.

Evidence: a reviewer can trace every field/invariant to a feature; fresh setup later works through Django migrations, including SQL routines/privileges; dependencies are locked once implementation is authorized. Avoid unnecessary stored duplicates; document why sale/payment snapshots preserve history. Links: checklist M0-10–M0-15, M1-05–M1-14, M4-22.

## 6. Security and Least Privilege

Owner, Manager, and Cashier are application roles; PostgreSQL runtime/migration/backup roles are separate database identities. Never expose PostgreSQL directly to the desktop/mobile client. Hash passwords/tokens using supported libraries, revoke sessions, parameterize SQL, restrict privileged posting and history edits, and use protected transport outside the local development exception.

Apply server-derived tenant scope and PostgreSQL RLS to business tables. Use same-tenant composite foreign keys; a tenant ID column alone does not prevent cross-business links. Missing context denies access. Runtime roles must not own tables or bypass RLS; plan transaction-local context and restricted authentication/onboarding paths.

Evidence: two-tenant API and actual-runtime-role RLS tests, pooled-connection context reset, cross-tenant FK failures, scoped reports/exports/idempotency, permission/access tests, actual runtime-role mutation failures, secret-free audit/error records, session revocation, and restoration without exposing backups. No standalone Auditor application role exists. Links: POS-NFR-03, POS-FR-01–POS-FR-04, POS-FR-28.

## 7. Backup, Recovery, and Contingency Planning

Backups protect data only if a restore can be performed and verified. The hosted operator owns hosted-service recovery; the business’s designated local administrator owns local-server recovery. Business Owner is an application role and does not automatically grant database/backup administration. A persistent Docker volume is storage, not a backup. Plan for accidental deletion, a corrupt schema/data change, lost local storage, interrupted operations, and unavailable backend/network.

Proposed learning targets, subject to M0 review:

- **Recovery point objective (RPO):** at most 24 hours of data loss for backup-based recovery, requiring a successful daily backup while the project is in use. Also back up before destructive/schema maintenance. If no data is being collected, document that operating condition rather than claiming an active daily service.
- **Recovery time objective (RTO):** restore and validate the synthetic acceptance dataset within 60 minutes after a usable backup and fresh target environment are available. Record provisioning/detection delays separately; this is an unproven target until tested.
- **Retention:** keep seven daily backups and four weekly copies while actively exercising the project; document pruning and failures. Retain at least one protected copy off the database host, without committing it to Git.
- **Protection:** restrict backup access; encrypt off-host copies and document how the authorized owner recovers encryption credentials. No secret keys in the repository.
- **Restore drill:** a hosted shared-database backup contains all tenants and is accessible only to authorized hosted operators, not ordinary business accounts. A local backup contains that installation’s one business and is restricted to its authorized local administrator. Restore into a separate protected database; compare row counts, sale/refund totals, stock, shift cash, and integrity reports per tenant, then recheck isolation policies/privileges before exposing the restored service. Whole-installation recovery in both deployment modes is in scope; selective per-tenant restoration/export tooling is deferred. Record the backup timestamp, measured time, and result.
- **Outage response:** distinguish internet loss from local server/LAN loss. A reachable business-local server can continue local cash posting without internet; a hosted client needs its hosted API. Pause new checkout/refund posting whenever the selected backend is unreachable; preserve request keys and resolve uncertain outcomes after recovery. Do not silently invent offline sales synchronization or repeat an external collection/refund.
- **Failed restore/backup:** keep the source database unchanged, retain error evidence, try the last known verified copy, and report the actual available recovery point. A managed provider's backup feature does not automatically prove these targets.

Tools to consider later: PostgreSQL `pg_dump`/`pg_restore` for this logical-backup exercise; stronger continuous recovery is outside the initial learning requirement unless explicitly added. No backup job or script is being created now. Link: POS-NFR-06.

## 8. Traceability, Observability, and Privacy

Keep immutable receipt/payment snapshots, operation outcomes, stock history, and privileged audit events. Distinguish a failed request from an inconsistent committed document. Record enough to investigate without copying passwords, tokens, or unnecessary personal data into logs/JSON payloads.

Evidence: request/receipt references connect history to business events; all privileged actions have atomic audit events; reconciliation flags deliberately broken fixtures; structured error logs and measured performance results are readable. Links: POS-FR-16, POS-FR-26–POS-FR-29.

## Deployment and Storage Evidence

Demonstrate the same schema and known retail totals on a hosted multi-tenant database and a separate business-local single-tenant database. Local PostgreSQL storage must persist beyond process/container restart; persistent storage alone is not a backup. Verify server restart, unavailable disk/server/LAN handling, backup to a protected copy outside the database host, and a measured restore in each mode. Document host OS/resources, startup ownership, update/rollback steps, local API addressing, certificate trust, and firewall boundaries before claiming installation support.

The local server runs both Django and PostgreSQL; devices use its API and hold no independently authoritative stock/payment database. Keep RLS and runtime-role verification in local mode even with only one tenant. Separate installations have separate account/token namespaces. Moving or merging live businesses between installations, cloud/local replication, and per-device disconnected posting are outside scope.

## Tenant-Isolation Design Evidence

Review the tenant table, one-settings-row-per-tenant rule, immutable tenant ownership, same-tenant relationship keys, per-tenant Owner/cash-method rules, and tenant-scoped identifiers. Exercise a large tenant alongside a small one for performance evidence. Context setup must cover HTTP requests, authentication lookup, reporting, background recovery, and controlled onboarding; verify missing context and failed transactions do not leave a pooled connection scoped to another tenant.

RLS is defense in depth alongside role/ownership checks, not a replacement for them. PostgreSQL superusers/BYPASSRLS bypass policies, and table owners normally bypass them; the runtime role must have neither property. Review USING/WITH CHECK policies and FORCE ROW LEVEL SECURITY, least-privilege routines, and tenant-table policies before coding. Sources: [PostgreSQL RLS](https://www.postgresql.org/docs/17/ddl-rowsecurity.html), [composite foreign keys](https://www.postgresql.org/docs/17/ddl-constraints.html#DDL-CONSTRAINTS-FK).

## Pricing Snapshots and Deletion Lifecycle — Review Required

POS-FR-32–POS-FR-36 expand the current design. Specify pricing calculation order, precision/rounding, allocations, override authorization, discount/promotion/tax snapshots, original-paid-value refunds, and tax-aware report reconciliation before approving the schema. The existing simple line-total check is not sufficient for these features.

Ordinary append-only history remains protected. Owner-initiated whole-business deletion needs a separately privileged, tenant-scoped lifecycle with verification, safe retries, retained-evidence policy, and backup/restore treatment. A hosted purge must leave every other tenant unchanged. Local deletion must explain the administrator’s responsibilities for database files and backup copies. Review any retention exceptions explicitly; do not claim that deletion removes independent copies immediately. The old deletion restrictions must be reconciled with this lifecycle before migrations are approved.

## Refund Approval, Device Control, and Reconciliation — Review Required

The confirmed workflow separates refund approver from executing cashier/shift, permits direct Manager/Owner own-shift execution, binds approval to exact request content, and invalidates stale approval. Review same-tenant attribution, approval/claim validity and consumption, restricted refund lookup, duplicate physical/provider payouts, and races with shift close or role changes.

One active mobile control session operates each shift; replacement-device resume invalidates prior authority under server checks. Model this separately from staff session history. External payment/refund followed by rejected posting requires a protected reconciliation lifecycle with original references, actors, amounts, and resolution evidence. Review cash effects and closure rules; do not assert a rollback reverses external money movement.

## Review Gate

For each goal, M0 review must name the enforcing component, the schema/routine decisions needed, the future verification case, and any accepted limitation. Stronger scalability/security/availability claims require evidence; they are not implied by choosing PostgreSQL or Docker.
