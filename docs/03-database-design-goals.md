# Vault Database Design Goals

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Confirm the integrity, consistency, atomicity, concurrency, and durability goals.
- [ ] Confirm the intended data growth and flexibility boundaries without expanding project scope.
- [ ] Confirm measurable performance targets and what benchmark evidence will be needed.
- [ ] Confirm maintainability, normalization, schema-history, and reproducibility expectations.
- [ ] Confirm security, least-privilege, auditability, and privacy expectations.
- [ ] Review backup schedule/retention, off-host protection, recovery-point/recovery-time targets, restore checks, and outage procedures.
- [ ] Record accepted targets/limitations and confirm this goals review is complete before moving to 04.

Review position **03 of 07**. Agree on measurable database quality and recovery goals before finalizing the design. See the [review order](../README.md#review-order--start-here). This document remains a draft; numbering does not mark it approved.

Status: planning draft for the single learning-project deliverable. These are design goals with review/verification evidence, not claims that the system already meets them. Review them in checklist M0-09–M0-11 before implementation.

## 1. Correctness, Consistency, and Integrity

Sales, payments, refunds, stock, and shift totals must agree. Define valid states and enforce them with primary/foreign keys, uniqueness, local checks, controlled posting routines, and transactions. A foreign key alone cannot enforce a balanced business operation or prevent concurrent overselling.

Evidence: a sale's lines equal its total/payment; exact per-line stock changes; no negative stock; one full successful refund per sale; immutable completed records; concurrent last-item tests; integrity queries that detect isolated corruption. Links: POS-NFR-01, POS-NFR-02, POS-FR-14–POS-FR-15, POS-FR-20, POS-FR-27.

## 2. Atomicity, Isolation, and Durable Outcomes

A checkout/refund either commits all its related effects or none. Deterministic product/shift/sale locks serialize competing actions. Durable operation keys let clients resolve retries and missing responses without posting twice. Protect committed work through PostgreSQL durability and verified recovery, rather than assuming a backup makes every recent commit recoverable.

Evidence: rollback injection; duplicate/reordered request tests; checkout-versus-close races; crash/response-loss scenarios; documented lock order/isolation/retry policy. Durability settings and any hosting limitations must be recorded. Links: POS-NFR-02, POS-FR-15, POS-FR-29.

## 3. Scalability and Controlled Flexibility

Use normalized retail entities and catalog rows so more SKUs, categories, registers, sales, and movements do not require a table per product/business type. Keep the supported single-location, whole-unit model explicit. Do not add arbitrary custom fields, multi-tenant scaffolding, sharding, or microservices as a substitute for a demonstrated need.

Evidence: at least 100,000 valid inventory movements, multiple concurrent registers, distribution-aware datasets, and query plans. Document which changes would require a deliberate schema extension. Links: POS-NFR-04, POS-FR-05–POS-FR-09.

## 4. Performance and Efficient Access

Use pagination, selective indexes, appropriate joins/aggregates, and measured plans. A faster query must still return correct totals and enforce permissions. Indexes have storage/write costs. Derived stock is authoritative; add caches only after measuring a bottleneck.

Evidence: proposed history p95 at most 500 ms for five minutes with 20 concurrent readers, pages up to 100 rows, and at least 100,000 inventory movements. Record hardware/cache conditions/errors and before/after SQL plans; benchmark reporting too without inventing unmeasured response-time claims. Links: POS-NFR-04, POS-NFR-08.

## 5. Maintainability and Reproducibility

Keep a data dictionary, meaningful names, documented ownership/cardinality, normalized source records, justified immutable snapshots, one migration authority, and reproducible synthetic fixtures. Keep secrets and generated backups out of source control. Track schema changes within this one project; database migrations are not additional product releases.

Evidence: a reviewer can trace every field/invariant to a feature; fresh setup later works through Django migrations, including SQL routines/privileges; dependencies are locked once implementation is authorized. Avoid unnecessary stored duplicates; document why sale/payment snapshots preserve history. Links: checklist M0-10–M0-15, M1-05–M1-14, M4-22.

## 6. Security and Least Privilege

Owner, Manager, and Cashier are application roles; PostgreSQL runtime/migration/backup roles are separate database identities. Never expose PostgreSQL directly to the desktop/mobile client. Hash passwords/tokens using supported libraries, revoke sessions, parameterize SQL, restrict privileged posting and history edits, and use protected transport outside the local development exception.

Evidence: permission/access tests, actual runtime-role mutation failures, secret-free audit/error records, session revocation, and restoration without exposing backups. No standalone Auditor application role exists. Links: POS-NFR-03, POS-FR-01–POS-FR-04, POS-FR-28.

## 7. Backup, Recovery, and Contingency Planning

Backups protect data only if a restore can be performed and verified. A persistent Docker volume is storage, not a backup. Plan for accidental deletion, a corrupt schema/data change, lost local storage, interrupted operations, and unavailable backend/network.

Proposed learning targets, subject to M0 review:

- **Recovery point objective (RPO):** at most 24 hours of data loss for backup-based recovery, requiring a successful daily backup while the project is in use. Also back up before destructive/schema maintenance. If no data is being collected, document that operating condition rather than claiming an active daily service.
- **Recovery time objective (RTO):** restore and validate the synthetic acceptance dataset within 60 minutes after a usable backup and fresh target environment are available. Record provisioning/detection delays separately; this is an unproven target until tested.
- **Retention:** keep seven daily backups and four weekly copies while actively exercising the project; document pruning and failures. Retain at least one protected copy off the database host, without committing it to Git.
- **Protection:** restrict backup access; encrypt off-host copies and document how the authorized owner recovers encryption credentials. No secret keys in the repository.
- **Restore drill:** restore into a separate database; compare row counts, sale/refund totals, stock, shift cash, and integrity reports. Record the backup timestamp, measured time, and result.
- **Outage response:** pause new checkout/refund posting when the backend is unreachable; preserve request keys and resolve uncertain outcomes after recovery. Do not silently invent offline sales synchronization or repeat an external collection/refund.
- **Failed restore/backup:** keep the source database unchanged, retain error evidence, try the last known verified copy, and report the actual available recovery point. A managed provider's backup feature does not automatically prove these targets.

Tools to consider later: PostgreSQL `pg_dump`/`pg_restore` for this logical-backup exercise; stronger continuous recovery is outside the initial learning requirement unless explicitly added. No backup job or script is being created now. Link: POS-NFR-06.

## 8. Traceability, Observability, and Privacy

Keep immutable receipt/payment snapshots, operation outcomes, stock history, and privileged audit events. Distinguish a failed request from an inconsistent committed document. Record enough to investigate without copying passwords, tokens, or unnecessary personal data into logs/JSON payloads.

Evidence: request/receipt references connect history to business events; all privileged actions have atomic audit events; reconciliation flags deliberately broken fixtures; structured error logs and measured performance results are readable. Links: POS-FR-16, POS-FR-26–POS-FR-29.

## Review Gate

For each goal, M0 review must name the enforcing component, the schema/routine decisions needed, the future verification case, and any accepted limitation. Stronger scalability/security/availability claims require evidence; they are not implied by choosing PostgreSQL or Docker.
