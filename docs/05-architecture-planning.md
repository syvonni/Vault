# Vault Architecture Planning

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Confirm mobile POS, desktop back office, API, application services, and database responsibilities.
- [ ] Complete and review the system-context/component diagram and protected communication boundaries.
- [ ] Define modules and the API request/response, error, money, timestamp, pagination, and idempotency conventions.
- [ ] Review the Django-user/session mapping, role/ownership checks, and runtime versus migration database permissions.
- [ ] Complete checkout/refund/void/shift-close flow/state diagrams, transaction boundaries, lock order, and failure recovery.
- [ ] Review local/device addressing, hosting options, logging, backup/recovery, CI, and packaging plans.
- [ ] Resolve the architecture questions and confirm the plan matches documents 01–04 before moving to 06.

Review position **05 of 07**. Review component boundaries, data flows, authentication, and where business rules are enforced. See the [review order](../README.md#review-order--start-here). This document remains a draft; numbering does not mark it approved.

Status: draft planning outline. This document is the explicit architecture deliverable for M0-06–M0-08 and M0-14. It authorizes no app scaffolding, dependency installation, configuration, migration, CI, container, or deployment work.

## Scope and Decisions

One learning-project deliverable, one business/location, multiple registers, whole-unit products, PHP, and the POS workflows in [01-requirements.md](01-requirements.md). Owner, Manager, and Cashier are the only staff roles. Audit history is a capability, not a fourth user role.

Confirmed technology: Python/Django, PostgreSQL, Django REST framework, Vue desktop UI, and React Native mobile direction. Tauri, Expo, and the token-auth mechanism remain proposals explained in [04-tech-stack.md](04-tech-stack.md). Developer/test tooling is selected there under the user's delegated choice. Hosting remains undecided and AWS is excluded from the shortlist.

## Working Client Split and Component Responsibilities

The user has proposed mobile POS and desktop back office. The draft step-by-step flows are in [02-feature-specifications.md](02-feature-specifications.md#workflow-drafts-by-client); they replace the earlier desktop-selling/mobile-summary proposal. Both clients use the same Django API and authoritative database. This split remains subject to the workflow review gate.

- **Desktop client:** Vue supplies the Owner/Manager back office: inventory/catalog/price management, staff/settings administration, metrics/reports, shift review, integrity checks, and audit history. The proposed Tauri shell provides a desktop window/native integration. Inventory posting still uses server-authorized actions and request keys.
- **Mobile client:** React Native provides POS: operator login, register/shift opening, product lookup/cart, payment recording, receipts, own history, draft void, and shift closure. Manager/Owner mobile refund execution is a draft under existing own-shift rules; review the handoff before approval. Expo and device/platform details remain open. Share API contracts/permissions, not assumed UI components.
- **API layer:** Django REST framework authenticates/validates requests, applies record/role permissions, and translates results/errors. Generic model CRUD must not bypass posting actions.
- **Application services:** Django modules for identity/setup, catalog/inventory, sales/payments/refunds, shifts, reporting, and audit. Coordinate domain actions, transaction boundaries, and recovery. These are modules inside one backend, not separate microservices.
- **Database:** PostgreSQL stores authoritative normalized source records, stock movements, document snapshots, operation outcomes, and audit data. It enforces structural constraints and controlled posting/locking rules.
- **Operational boundary:** Local Docker services later run the backend/database; native desktop/mobile tools run on the host. A remote deployment may replace the local backend address without duplicating the database into each client.

## Architecture Questions to Resolve Before Code

1. Confirm Tauri versus another shell, client UI conventions, mobile POS device/register binding, refund handoff, mobile platforms, and protected network boundaries. Vue is already chosen; the desktop-shell choice is independent.
2. Map Django's custom staff user and session/token approach to the conceptual `staff_users`/`staff_sessions` needs. Retain role, expiry, history/revocation, and hashed-secret behavior without blindly duplicating framework tables.
3. Decide which critical invariants live in PostgreSQL routines/deferred constraints and which application services invoke them. Separate migration/runtime identities and document how callers cannot bypass controlled writes.
4. Specify checkout/refund/shift-close sequences, durable operation-claim lifecycle, stale-cart detection, global lock ordering, bounded retries, crash recovery, and safe user-visible failures.
5. Define request/response contracts: decimal money represented safely (for example decimal strings), whole integer quantities, timestamps/timezones, pagination, errors, references, and idempotency keys.
6. Define authentication/authorization flows and client credential handling, including permission changes and expired sessions during an open shift. An expired session does not silently close its business shift; reauthentication must preserve valid operator attribution.
7. Plan the local topology, device-to-backend addressing, config/secrets, and separate test database. A mobile device's localhost is not the development laptop; the later runbook must name the reachable development host.
8. Compare non-AWS hosted and store-local deployment choices, accepting that unreachable backend means no new checkout/refund posting. Do not add offline synchronization implicitly.
9. Define logging/error visibility, integrity reports, backups/restore targets, CI/test stages, and desktop/mobile packaging/signing expectations. Use [database design goals](03-database-design-goals.md) for review evidence.

## Needed Planning Artifacts

- System context/component diagram showing desktop/mobile → authenticated API → domain services → PostgreSQL, and external manually verified payments outside Vault.
- Data-flow and state diagrams for checkout, refund, draft void, and shift closure, including failed/unknown-response recovery.
- Screen/navigation sketches and reviewed feature specifications from [02-feature-specifications.md](02-feature-specifications.md).
- Role/action/record-scope mapping and initial-owner/staff-credential flow.
- Schema data dictionary and invariant/enforcement/verification mapping.
- Dependency/version baseline and proposed local/run/deployment/distribution runbooks; commands become executable only after implementation is authorized.

These artifacts are still planning tasks. This outline is not proof that they have been completed or that the conceptual schema is final.

## Meaning of the Earlier “Adapt the Model, Then Begin Migrations” Step

Adapting the model means deciding on paper how staff identities, password hashes, active flags, roles, per-device sessions/tokens, expiry, and revocation map to Django and the selected authentication package. It may change the DBML before implementation. A migration is executable code that later creates/changes database structure; it comes after the planning gate and explicit authorization, not during this review.

## Acceptance Gate

M0 is complete only when the required artifacts are reviewed alongside requirements, feature specifications, goals, and DBML; unresolved behavior is settled; and the user explicitly authorizes implementation. Keep the schema marked draft until that point. One project can still have package versions, Git history, database migrations, and a cart edit counter without planning additional product releases.
