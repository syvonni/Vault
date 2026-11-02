# Vault Architecture Planning

Review position **05 of 07**. Review component boundaries, data flows, authentication, and where business rules are enforced. Track this review in [GitHub issue #5](https://github.com/syvonni/Vault/issues/5). See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Status: draft planning outline. This document is the explicit architecture deliverable for M0-06–M0-08 and M0-14. It authorizes no app scaffolding, dependency installation, configuration, migration, CI, container, or deployment work.

## Scope and Decisions

Vault is a learning project with two supported deployment modes (shared hosted multi-tenant and business-local single-tenant), one codebase/schema, one location per business, multiple registers, whole-unit products, PHP, and the POS workflows in [01-requirements.md](01-requirements.md). Owner, Manager, and Cashier are the only staff roles. Audit history is a capability, not a fourth user role.

Confirmed technology: Python/Django, PostgreSQL, Django REST framework, Vue desktop UI, and React Native mobile direction. Tauri is confirmed as the Windows/macOS desktop container. Expo and the token-auth mechanism remain proposals explained in [04-tech-stack.md](04-tech-stack.md). Developer/test tooling is selected there under the user's delegated choice. Hosting remains undecided and AWS is excluded from the shortlist.

## Working Client Split and Component Responsibilities

The user has proposed mobile POS and desktop back office. The draft step-by-step flows are in [02-feature-specifications.md](02-feature-specifications.md#workflow-drafts-by-client); they replace the earlier desktop-selling/mobile-summary proposal. Both clients use the same Django API and authoritative database. This split remains subject to the workflow review gate.

- **Desktop client:** Vue supplies the Owner/Manager back office: inventory/catalog/price management, staff/settings administration, metrics/reports, shift review, integrity checks, and audit history. The confirmed Tauri desktop container provides a desktop window/native integration. Inventory posting still uses server-authorized actions and request keys.
- **Mobile client:** React Native provides POS: operator login, register/shift opening, product lookup/cart, payment recording, receipts, own history, draft void, and shift closure. Cashier refund execution on mobile requires specific separately authenticated Manager/Owner approval in the desktop back office. Record approver separately from executing shift/operator; a Manager/Owner operating their own mobile shift can execute directly. One active mobile control session operates each shift, including explicit replacement-device handoff. Expo and device/platform details remain open. Share API contracts/permissions, not assumed UI components.
- **API layer:** Django REST framework authenticates/validates requests, derives tenant context from the staff account, applies record/role permissions, and translates results/errors. Generic model CRUD must not bypass posting actions.
- **Application services:** Django modules for controlled tenant onboarding, identity/setup, catalog/inventory, sales/payments/refunds, shifts, reporting, and audit. Coordinate domain actions, transaction boundaries, and recovery. These are modules inside one backend, not separate microservices.
- **Database:** PostgreSQL uses one shared schema with explicit tenant keys, same-tenant composite foreign keys, scoped business uniqueness, and row-level security. It stores authoritative normalized source records, stock movements, document snapshots, operation outcomes, and audit data. It enforces structural constraints and controlled posting/locking rules.
- **Operational boundary:** Local Docker services later run the backend/database; native desktop/mobile tools run on the host. Both a shared non-AWS hosted installation and a business-local server installation are required. Each runs its backend and database; clients connect to its API without a separate authoritative database in each client.

## Architecture Questions to Resolve Before Code

1. Apply the confirmed Vue + Tauri desktop container, Manager/Owner-authorized cashier mobile refunds, logical register selection/one active mobile shift-control session, and PDF receipt delivery. Specify remaining approval/version/expiry, device-handoff, reconciliation and platform-acceptance details.
2. Map Django's custom staff user and session/token approach to the conceptual `staff_users`/`staff_sessions` needs. Retain role, expiry, history/revocation, and hashed-secret behavior without blindly duplicating framework tables.
3. Decide which critical invariants live in PostgreSQL routines/deferred constraints and which application services invoke them. Separate migration/runtime identities and document how callers cannot bypass controlled writes.
4. Specify checkout/refund/shift-close sequences, durable operation-claim lifecycle, stale-cart detection, global lock ordering, bounded retries, crash recovery, and safe user-visible failures.
5. Define request/response contracts: decimal money represented safely (for example decimal strings), whole integer quantities, timestamps/timezones, pagination, errors, references, and idempotency keys.
6. Define authentication/authorization flows and client credential handling, including permission changes and expired sessions during an open shift. An expired session does not silently close its business shift; reauthentication must preserve valid operator attribution.
7. Plan the local topology, device-to-backend addressing, config/secrets, and separate test database. A mobile device's localhost is not the development laptop; the later runbook must name the reachable development host.
8. Plan both a non-AWS hosted deployment and a business-local backend/database server, plus local Docker for development/testing, accepting that unreachable backend means no new checkout/refund posting. Do not add offline synchronization implicitly.
9. Define logging/error visibility, integrity reports, backups/restore targets, CI/test stages, and desktop/mobile packaging/signing expectations. Use [database design goals](03-database-design-goals.md) for review evidence.

## Confirmed Client Workflow Boundaries

Tauri packages Vue for Windows/macOS. Mobile submits sales/refunds to the installation’s API; desktop fetches those same records and provides authenticated Manager/Owner refund approval. Both use one authoritative server/database, including in local mode. This included client/server exchange is distinct from excluded local/cloud database replication.

Refund approval binds a specific tenant/sale/amount/restock decisions/reason/executor/shift/request version and records approver separately from execution. Changes invalidate approval; role/revocation/expiry and duplicate-execution claims must be designed before posting. Cash impact belongs to the executing operator's current shift. Plan restricted receipt lookup for cashier-initiated returns without expanding ordinary history permissions.

A logical register represents a checkout station/drawer. One active mobile control session may operate its shift. Explicit same-operator resume on a replacement device invalidates the previous control session; posting rechecks authority. State/race details still require modeling.

Receipts are on-screen and PDF download/share with tenant/role checks. Missing response uses existing operation lookup; definitive posting rejection after actual external money movement requires a Manager/Owner reconciliation case with evidence. Specify safe recovery/compensation, duplicate-external-action prevention, and shift-closure handling; do not treat a precheck or approval as a guarantee of database/provider atomicity.

## Future Communication and Concurrency Review

Roadmap tasks **M0-23–M0-25** explicitly reserve planning for device/server update timing, cross-device freshness, acknowledgment/reconnect behavior, the cloud/local synchronization boundary, and a consolidated simultaneous-request test matrix. Record future decisions here and the screen/error behavior in document 02. No refresh mechanism, synchronization algorithm, or new offline capability is selected by this reminder. Existing transaction/idempotency/locking requirements remain in force.

## Deployment Boundary

The same Django services and tenant-aware PostgreSQL schema support both topologies:

- **Hosted:** desktop/mobile → internet/protected API → shared Django service → private PostgreSQL database with multiple tenants.
- **Business-local:** desktop/mobile → store LAN/protected API → designated store server running Django → private local PostgreSQL database with one tenant.

Each installation owns one authoritative database. No synchronization between independent local/cloud databases, replication, automatic failover, cross-installation identity service, or disconnected client posting is included. The tenant key and RLS stay enabled in local mode. Internet-independent local operation still depends on the local server/LAN and does not guarantee external payment-provider availability.

Define deployment-specific secrets, API endpoints, protected transport/certificate trust, firewall rules, persistent storage, production application-server process, startup/restart, logs/health checks, upgrades/rollback, onboarding, and backup ownership. Choose a supported local-server host OS/runtime/resource baseline separately from desktop client OS targets. The local administrator is an installation responsibility, not an additional application role.

Clients may be configured for one installation at a time. Endpoint changes invalidate old client credentials/cache and require authentication on the destination; never replay pending operations to a different installation. Resolve active drafts/shifts and uncertain outcomes on their originating server. Automated data migration between installations is deferred.

## Tenant and Identity Boundary

A `tenants` row represents one business. Every business-owned row carries `tenant_id`; `store_settings` has exactly one row per tenant. Each staff account belongs to one immutable tenant with one role; canonical email unique within the installation keeps initial login unambiguous; separate installations have independent accounts/tokens. No cross-business membership model or business-switching UI is planned.

The authorized installation operator (hosted operator or designated local administrator) atomically onboards a tenant, settings, built-in cash method, and first Owner through a separate restricted operational path with protected audit evidence. This is not a public endpoint or an application staff role. Normal business Owners cannot access other tenants or operator credentials.

After narrowly scoped credential/token verification, the API derives the tenant from the authenticated account and installs transaction-local database context. The authentication lookup must not require an unrestricted general-purpose database bypass: specify a minimal credential-resolution interface, safe failures/rate limits, and its privileges before implementation. Review Django user/auth-library mapping explicitly; framework authentication does not automatically supply tenant isolation. Source: [Django custom authentication](https://docs.djangoproject.com/en/5.2/topics/auth/customizing/).

All business reads/writes, joins, aggregates, exports, audit targets, background/recovery jobs, and optional cache keys include validated tenant scope. Client fields cannot set or reassign tenant ownership. Same-tenant composite foreign keys prevent linking another tenant's staff/product/shift/sale/request. Staff roles and own-shift permissions still apply within the tenant.

Plan PostgreSQL RLS with both read visibility and write checks, FORCE ROW LEVEL SECURITY where applicable, and a non-owner runtime role without superuser/BYPASSRLS. Context is transaction-local and missing context denies business access; connection reuse cannot retain another tenant's context. Tenant registry policies must expose only the authenticated tenant. Review privileged routines for bypass/search-path risks and keep migration/onboarding/backup credentials outside request-serving code. Sources: [PostgreSQL RLS](https://www.postgresql.org/docs/17/ddl-rowsecurity.html), [foreign keys](https://www.postgresql.org/docs/17/ddl-constraints.html#DDL-CONSTRAINTS-FK).

Hosted backups contain all hosted tenants and belong to the hosted operator’s operational boundary. Local backups contain that local installation’s one business and belong to its designated administrator’s operational boundary. Restore drills verify each tenant's totals and isolation before clients reconnect. Per-tenant self-service restore, billing, public signup, and a platform administration UI remain deferred.

## Added Pricing and Deletion Boundaries — Pending Design

POS-FR-32–POS-FR-35 require a reviewed server-side pricing flow across configuration, cart preview/confirmation, override authorization, transactional completion, immutable receipts, full refunds, and reporting. Define calculation/rounding/allocation and concurrent rule-change behavior before selecting the schema or final formulas.

POS-FR-36 requires a tenant-deletion state/permission model distinct from normal staff administration. Plan Owner verification/confirmation, export, pending operations/open shifts, access revocation, any cancellation window, controlled purge/retry/status handling, and backup/restore exclusions or suppression evidence. Define what minimal operational evidence remains, who may access it, and why. Hosted execution cannot affect other tenants; local execution needs documented installation-administrator duties. Exact lifecycle and execution mechanism remain undecided.

## Needed Planning Artifacts

- Hosted and business-local system context/component diagrams showing desktop/mobile → authenticated API → domain services → PostgreSQL, and external manually verified payments outside Vault.
- Data-flow and state diagrams for checkout, refund, draft void, and shift closure, including failed/unknown-response recovery.
- Screen/navigation sketches and reviewed feature specifications from [02-feature-specifications.md](02-feature-specifications.md).
- Tenant/role/action/record-scope mapping, controlled onboarding/first-owner flow, restricted credential lookup, RLS context lifecycle, and two-tenant isolation verification plan.
- Schema data dictionary and invariant/enforcement/verification mapping.
- Dependency/version baseline and separate hosted/business-local installation, startup, update/rollback, backup/recovery, and client endpoint runbooks; commands become executable only after implementation is authorized.

These artifacts are still planning tasks. This outline is not proof that they have been completed or that the conceptual schema is final.

## Meaning of the Earlier “Adapt the Model, Then Begin Migrations” Step

Adapting the model means deciding on paper how staff identities, password hashes, active flags, roles, per-device sessions/tokens, expiry, and revocation map to Django and the selected authentication package. It may change the DBML before implementation. A migration is executable code that later creates/changes database structure; it comes after the planning gate and explicit authorization, not during this review.

## Acceptance Gate

M0 is complete only when the required artifacts are reviewed alongside requirements, feature specifications, goals, and DBML; unresolved behavior is settled; and the user explicitly authorizes implementation. Keep the schema marked draft until that point. One project can still have package versions, Git history, database migrations, and a cart edit counter without planning additional product releases.
