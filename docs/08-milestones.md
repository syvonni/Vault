# Vault Milestones

Review position **08 of 08**. Review milestone outcomes and completion gates after the detailed task roadmap. Track this review alongside document 07 in [GitHub issue #7](https://github.com/syvonni/Vault/issues/7). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft.

## How to Use This File

This file owns the milestone outcomes and completion gates. [07-progress-checklist.md](07-progress-checklist.md) keeps the complete task roadmap, stable task IDs, review navigation, and sparse commit guidance. GitHub issues hold active progress; do not maintain another checklist here. Milestones organize work in Vault as a learning project, not separate product releases.

Requirements and final product acceptance live in document 01. Milestone gates below determine when each stage is ready to proceed; they are not a second copy of the product requirements.

## Milestone 0 — Feature Specifications, Architecture, and Design Review

**Outcome:** A coherent, reviewed plan for Vault as a learning project. This milestone produces documentation only. Do not create app scaffolding, environments, dependencies, migration/configuration files, CI workflows, containers, or deployments during planning.

**Completion gate:** Pricing/deletion and confirmed client workflow additions (M0-26–M0-30), feature specifications, screen flows, architecture, database goals, and conceptual schema are reviewed together; outstanding scope/design decisions are resolved; the user explicitly authorizes implementation. A generated document or valid DBML syntax alone cannot satisfy this gate. The current documents are drafts, so this gate is open.

Detailed work: **M0-01–M0-30** in document 07.

## Milestone 1 — Design and Database Foundation

**Coverage:** POS-FR-01–POS-FR-36 foundations; POS-NFR-01, POS-NFR-03.

**Outcome:** A clean database can be created with the POS schema, initial configuration, staff roles, and synthetic reference data.

**Completion gate:** Repeatable setup creates the POS database, constraints demonstrably reject invalid rows, and schema relationships match the documentation, including tenant-scoped keys and runtime-role isolation. No selling workflow is marked complete yet.

Detailed work: **M1-01–M1-23** in document 07.

## Milestone 2 — Staff Access, Inventory, Shifts, and Checkout

**Coverage:** POS-FR-01–POS-FR-17, POS-FR-21, POS-FR-22, POS-FR-28–POS-FR-31; POS-NFR-01–POS-NFR-03, POS-NFR-05. Also covers the relevant POS-FR-32–POS-FR-35 pricing foundations/refund/report behavior.

**Outcome:** A cashier can complete cash/manual noncash sales without duplicate posting or overselling; managers can maintain the catalog and receive/correct stock.

**Completion gate:** Backend tests demonstrate valid cash/noncash selling, accurate stock, immutable receipts, authorized catalog/stock actions, idempotency, rollback, tenant isolation/context lifecycle, and last-item concurrency protection. Refund and close-shift completion are still pending Milestone 3.

Detailed work: **M2-01–M2-30** in document 07.

## Milestone 3 — Refunds, Cash Reconciliation, Reports, and Audit

**Coverage:** POS-FR-16, POS-FR-18–POS-FR-20, POS-FR-22–POS-FR-31; POS-NFR-01–POS-NFR-03, POS-NFR-05. Also covers the relevant POS-FR-32–POS-FR-35 pricing foundations/refund/report behavior.

**Outcome:** Staff can handle returns and finish shifts; owners can inspect operational reports and detect inconsistent records.

**Completion gate:** Full refunds, restock choices, cash closure, receipt/history visibility, reports, and integrity queries match known fixtures and pass concurrency/rollback and cross-tenant isolation tests. Original sales/shifts remain unchanged by later returns.

Detailed work: **M3-01–M3-23** in document 07.

## Milestone 4 — Frontend, Performance, Recovery, and Delivery

**Coverage:** All POS functional requirements in the UI; POS-NFR-04, POS-NFR-06–POS-NFR-08 and final regression checks.

**Outcome:** A usable POS demonstrates the verified workflows, handles retries clearly, meets a measured performance target, and can be restored/reproduced.

**Completion gate:** Required role flows work through the frontend, correctness/concurrency checks pass, benchmark results meet the agreed target or document an explicitly revised target, restore verification succeeds, a fresh checkout reproduces the demo, two businesses demonstrate isolated mobile/desktop operation on the hosted service, and one business demonstrates LAN operation, internet-outage cash selling, safe server/LAN failure handling, restart persistence, and recovery on the local-server installation. Approved pricing flows and the business-deletion demonstration also pass their reviewed acceptance cases.

Detailed work: **M4-01–M4-28** in document 07.
