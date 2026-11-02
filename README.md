# Vault POS

## Your Review Checklist

Use these as your document-level review tracker. Start with 01 and keep 07 open alongside it. All reviews remain unchecked.

- [ ] Review and finalize [01 — Requirements](docs/01-requirements.md).
- [ ] Review and finalize [02 — Features and workflows](docs/02-feature-specifications.md).
- [ ] Review and finalize [03 — Database goals](docs/03-database-design-goals.md).
- [ ] Review and finalize [04 — Tech stack](docs/04-tech-stack.md).
- [ ] Review and finalize [05 — Architecture](docs/05-architecture-planning.md).
- [ ] Review and finalize [06 — Conceptual schema](docs/06-conceptual-schema.md).
- [ ] Reconcile [07 — Progress checklist](docs/07-progress-checklist.md) and resolve the remaining planning items.

Vault is a point-of-sale and inventory project, starting with a single-location hardware store that sells individually counted products. Its reusable retail core covers catalog, stock movements, sales, recorded payments, internal receipts, full refunds, cashier shifts, and operational reports.

A cashier can sell a product, accept cash and calculate change, or record a manually verified noncash payment. Stock deductions and the completed sale commit together. A manager can record a full refund, decide which returned items go back into stock, and review cash-drawer variances.

The project emphasizes PostgreSQL normalization, atomic operations, concurrency, idempotency, advanced SQL, performance measurement, and backup recovery.

Python, Django, PostgreSQL, Django REST framework, and Vue are confirmed. The working split is React Native mobile POS and Vue desktop back office for inventory, administration, reports, and audit review. Desktop shell, detailed workflows/refund handoff, authentication, and hosting proposals remain under planning review. Developer tools and a compatible package baseline are documented in the tech stack plan.

## Scope

One business/location, multiple registers, PHP, whole-number product quantities, one payment per sale, and full-sale refunds. Full accounting, payroll, expenses/payables, restaurant workflows, integrated payment processing, and statutory invoicing are deferred. Payments are recorded by Vault; money is handled outside it.

This replaces the earlier simulated-wallet concept. Existing documents describe the POS design; no application, migrations, or runtime setup has been implemented yet.

## Planning Only

This is one learning-project deliverable built in stages, not a roadmap of product releases. The user has not authorized application code/setup. Complete the Milestone 0 feature-specification, architecture, database-goal, and schema reviews before requesting implementation authorization. Existing documents and DBML are drafts, not completed approvals.

## Review Order — Start Here

**Current step: 01 — Requirements.** Review one document at a time; you do not need to finalize all of them at once. Every document is still a draft, and implementation remains unauthorized.

1. [01 — Requirements](docs/01-requirements.md): decide what Vault does, who uses it, the mobile-POS/desktop-back-office split, and what is out of scope.
2. [02 — Feature specifications and workflows](docs/02-feature-specifications.md): walk through what each person does, screen steps, validation, failures, retries, and acceptance examples.
3. [03 — Database design goals](docs/03-database-design-goals.md): agree on integrity, performance, security, maintainability, and backup/recovery expectations.
4. [04 — Tech stack](docs/04-tech-stack.md): settle the remaining technology, authentication, local-running, and delivery choices that support those workflows.
5. [05 — Architecture planning](docs/05-architecture-planning.md): decide how mobile, desktop, API, application services, and PostgreSQL fit together and enforce the rules.
6. [06 — Conceptual schema](docs/06-conceptual-schema.md): review tables, fields, relationships, and constraints against the preceding decisions.
7. [07 — Progress checklist](docs/07-progress-checklist.md): reconcile the plan with its tasks and confirm the planning gate before any implementation.

Keep **07 open as a tracker throughout**, rather than waiting until the end to use it. Its existing task IDs are stable; the filename prefixes give the document review order. If a later review exposes a gap, revisit only the affected earlier decisions.

At each step, resolve the document's open questions, record agreed changes, and explicitly confirm that review before moving on. Draft checkmarks mean a document exists, not that you have approved it. After all planning reviews, coding/setup still requires your explicit authorization.

For now, open **01** and review only the purpose, scope, client responsibilities, and roles. We can handle its functional requirements next, before moving to **02**. Runtime setup, test, benchmark, and recovery commands will be added only when implementation is authorized and those commands exist.
