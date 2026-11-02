# Vault POS — Tech Stack and Operating Plan

## Your Review Checklist

Check each item after reviewing it and resolving its questions. These track your review, not completed implementation.

- [ ] Review the confirmed Python/Django/PostgreSQL/DRF/Vue/React Native choices and the recommended compatible versions.
- [ ] Decide whether to accept Tauri for desktop packaging and Expo for mobile tooling.
- [ ] Review the selected developer, formatting, dependency, frontend/backend test, and performance tools.
- [ ] Review and resolve the token-auth proposal, session expiry/revocation, and client credential storage.
- [ ] Confirm the Docker/local-running plan and how desktop/mobile clients reach the backend.
- [ ] Choose an acceptable deployment scope (local demo is valid) and review build/signing/distribution expectations without provisioning anything.
- [ ] Record resolved choices and any accepted limitations; confirm this stack review is complete before moving to 05.

Review position **04 of 07**. Resolve technology and operating choices that support the agreed requirements and workflows. See the [review order](07-progress-checklist.md#document-review-order). This document remains a draft; numbering does not mark it approved.

Status: documentation only, reviewed against official documentation on 2026-11-01 (Asia/Manila). No application, dependencies, lockfiles, Docker configuration, migrations, CI workflow, credentials, or deployment have been created. Complete [Milestone 0](07-progress-checklist.md) before implementation.

This is one learning project, not a multi-release roadmap. Build order, package versions, schema migration history, and cart edit counters remain useful inside that one deliverable.

## Confirmed Choices

- Python with Django for the backend and PostgreSQL for persistent data.
- Django REST framework (DRF) for the API, explicitly confirmed by the user.
- Vue for the desktop interface, explicitly confirmed by the user.
- React Native for the mobile POS direction; Vue for the desktop back office. The user clarified this split; detailed workflows, refund handoff, and device/platform acceptance remain under review in [02-feature-specifications.md](02-feature-specifications.md#workflow-drafts-by-client).
- Application roles: Owner, Manager, Cashier. Owner/Manager can review audit history; there is no separate read-only reviewer role.
- The user delegated developer-tool selection; choices below are selected for the plan, not installed.

## Recommended Package Baseline

Select these compatible release families for planning. Where an exact patch is listed, it is a research snapshot; recheck security patches before authorized setup. Do not resolve/install or fabricate a lockfile during planning. Final dependency resolution and OS builds remain untested.

- **Python 3.13.x + Django 5.2 LTS**, using **Django 5.2.18** as the checked patch baseline. Django lists Python 3.13 support for 5.2, and 5.2 remains supported through April 2028. The LTS choice is deliberate rather than a claim that it is the newest feature release. Sources: [5.2 compatibility](https://docs.djangoproject.com/en/5.2/releases/5.2/), [supported releases](https://www.djangoproject.com/download/).
- **DRF 3.18.3**, whose supported Python/Django families include this baseline. Sources: [official requirements](https://www.django-rest-framework.org/), [published release](https://pypi.org/project/djangorestframework/).
- **PostgreSQL 17.x + Psycopg 3.3.6**. Django 5.2 supports PostgreSQL 14+ and Psycopg 3.1.8+; these choices satisfy those published requirements. Use the supported PostgreSQL 17 minor available at implementation time. Sources: [Django PostgreSQL requirements](https://docs.djangoproject.com/en/5.2/ref/databases/), [Psycopg release](https://pypi.org/project/psycopg/), [PostgreSQL maintenance policy](https://www.postgresql.org/support/versioning/).
- **Vue 3 + TypeScript 5.9.x + Vite 8.x**, using **Node.js 24 LTS** and npm for frontend tooling. Vue's official scaffold supports TypeScript/Vite; Node 24 is an LTS line above Vite's documented minimum. Lock maintained compatible package patches after authorization. Sources: [Vue setup](https://vuejs.org/guide/quick-start.html), [Vite requirements](https://vite.dev/guide/), [Node releases](https://nodejs.org/en/about/previous-releases).
- **Tauri 2.x**, if its proposed desktop-shell choice is accepted; use a supported stable Rust toolchain and native prerequisites for each target OS. Do not confuse Node frontend tooling or the Rust shell with a second business backend. Source: [Tauri prerequisites](https://v2.tauri.app/start/prerequisites/).
- **Proposed mobile pairing: Expo SDK 58 + React Native 0.88 + React 19.3.0**, with Node 24 meeting Expo's documented Node 22.13+ minimum. Keep the Expo/RN/React combination paired; do not independently upgrade each to an unrelated latest version. This is a proposal until Expo is accepted. Source: [Expo compatibility table](https://docs.expo.dev/versions/latest/#each-expo-sdk-version-depends-on-a-react-native-version).

Developer tools should use maintained releases compatible with these families, frozen in the eventual dependency lockfiles. Pin container images/toolchains sufficiently for reproducibility when setup is authorized; documentation ranges are not a completed installation compatibility test.

## What Each Layer Does

- **Vue:** builds the desktop Owner/Manager workspace for inventory, staff/configuration, metrics/reports, and audit review. TypeScript adds checked data/component types; Vite runs/builds the interface assets.
- **Tauri (recommended, not yet confirmed):** puts those Vue assets in a Windows/macOS application window and provides controlled native integration/packaging. Vue builds the screens; Tauri supplies the desktop application container. Electron remains an alternative if the user declines Tauri.
- **Django + DRF:** provide staff identity/permissions, API contracts, and controlled business actions. Use Django ORM/migrations, with parameterized SQL/routines for the database learning goals. There is no separate FastAPI/Express backend.
- **Psycopg:** the Python PostgreSQL driver underneath Django's database adapter. It sends database commands/parameters, receives rows, and participates in transaction handling. It is not a second database, an ORM, or a desktop UI framework. Django commonly calls it on the application's behalf. Source: [Psycopg introduction](https://www.psycopg.org/psycopg3/docs/).
- **PostgreSQL:** authoritative persisted records, constraints, locks, inventory movements, request outcomes, and reporting SQL. Desktop/mobile clients never connect directly to it.
- **React Native + proposed Expo:** mobile POS screens for selling, recorded payments/receipts, and cashier shifts, calling the same DRF API. This supersedes the earlier read-oriented manager-companion proposal. Expo adds mobile development/build tooling and device-access libraries to React Native; it does not replace Django or PostgreSQL. A development build is a build of your own app with its native dependencies, whereas Expo Go is a general-purpose development client. Sources: [Expo SDK](https://docs.expo.dev/versions/latest/), [development builds](https://docs.expo.dev/develop/development-builds/introduction/).

The earlier “UI approach” question meant how screens are rendered and connected: Django-rendered HTML versus a separate client calling an API. Vue desktop back office plus React Native mobile POS, both using DRF, establishes the separate-client direction. Exact navigation, cart interactions, receipt layout, component styling, and mobile workflow details still belong in feature/screen planning. It did not mean planning a second product release.

## Selected Developer and Verification Tools

- **uv:** Python environment/dependency management and eventual lockfile. [uv documentation](https://docs.astral.sh/uv/).
- **Ruff:** Python linting/formatting. [Ruff documentation](https://docs.astral.sh/ruff/).
- **npm + committed package lockfiles:** JavaScript dependency management for separate desktop/mobile projects. Use Vue's TypeScript checks, ESLint, and Prettier for client checks/formatting; select compatible configs during authorized setup.
- **pytest + pytest-django:** backend, permission, rollback, and PostgreSQL integration tests. Concurrency tests use real independent connections/transactions, not SQLite or a test wrapper that hides commits. [pytest-django](https://pytest-django.readthedocs.io/en/latest/).
- **Vitest + Vue Test Utils:** Vue logic/components. **Playwright:** browser-rendered Vue workflows against the API. Browser tests do not prove native shell/signing behavior; add Windows/macOS packaged-app smoke checks. Sources: [Vitest](https://vitest.dev/guide/), [Vue testing](https://vuejs.org/guide/scaling-up/testing.html), [Playwright](https://playwright.dev/docs/intro).
- **jest-expo + React Native Testing Library:** proposed mobile component/workflow tests if Expo is accepted; include real-device/emulator smoke tests. [Expo testing](https://docs.expo.dev/develop/unit-testing/).
- **Locust:** Python-written API load scenarios for the specified 20-reader/five-minute benchmark; report p95 and errors. **PostgreSQL EXPLAIN (ANALYZE, BUFFERS):** inspect query work; **pg_stat_statements:** optional aggregate query evidence where available. Perform measurements against synthetic test/staging data. [Locust documentation](https://docs.locust.io/en/stable/).
- **GitHub Actions:** proposed checks/build workflow during implementation. Plan lint/type/unit checks, PostgreSQL integration tests, browser workflows, and Windows/macOS build checks. Keep signing credentials in protected CI secrets; deployment/build execution is not authorized now. [GitHub Actions concepts](https://docs.github.com/en/actions/get-started/understand-github-actions).

No Redis, task queue, microservices, or additional analytics backend is needed initially.

## Authentication Proposal — Explain and Review Before Selection

Authentication answers “Which staff member is this?” Authorization answers “May this person refund this sale or access this shift?” Login is not permission to perform every action.

Recommended native-client flow:

1. Staff logs in over a protected connection; Django verifies its stored password hash and active-user state.
2. The API issues a random per-device access token with expiry. The server retains only its verifier/hash, not a copy of the usable raw token.
3. The client includes the token with API requests. The backend checks token validity, active status, current role, and ownership/record permissions on every action.
4. Logout, expiry, password/role/access changes, or explicit session revocation invalidate the relevant tokens. Staff cannot keep old permissions by retaining a previously issued token.
5. Preserve separate nonsecret session-history metadata even if an authentication library deletes expired/revoked token records. Losing login does not close a cashier's business shift; reauthentication recovers the same operator context.

**Proposed library: Django REST Knox 5.1.0**, which supports multiple tokens, hashed storage, and expiry. Confirm its configured expiry/logout behavior, Django compatibility, and schema mapping during M0; it is not installed/approved merely because it is recommended. Sources: [Knox project](https://github.com/django-commons/django-rest-knox), [published release](https://pypi.org/project/django-rest-knox/), [DRF authentication](https://www.django-rest-framework.org/api-guide/authentication/).

For the learning desktop build, initially keep the raw token only in memory and log in again after restarting the app. Do not persist it in webview local storage. For proposed Expo mobile, use SecureStore for the token, never retain the staff password. Expiry still applies and reauthentication is required. Source: [Expo SecureStore](https://docs.expo.dev/versions/latest/sdk/securestore/).

A shared staff password would destroy attribution. There is no public signup: controlled first-owner bootstrap, then owner-provisioned individual staff. Browser-only Django admin can use its normal session/CSRF protections; native-token APIs need their own explicit policy. No JWT, identity provider, refresh-token hierarchy, or custom cryptography is required just for this learning scope.

## Running the Project Later — Docker Is Appropriate

Selected local operating plan: Docker Desktop with Docker Compose for two services, PostgreSQL (`db`) and Django/DRF (`api`), with persistent database storage and a separate protected test database. A container volume is not a backup. Native Tauri tooling and Expo/emulators run on the host, where they can open actual OS windows/use native SDKs. Source: [Docker Compose](https://docs.docker.com/compose/).

Planned startup sequence after implementation authorization:

1. Prepare the reviewed local configuration and secret placeholders; start `db`/`api` services through Compose.
2. Run approved Django migrations and the controlled first-owner setup inside the backend service.
3. Run the Vue/Tauri development application on the host; it connects to the local API endpoint.
4. Run the mobile development client/emulator on the host/device when its workflow is ready. Use the laptop's reachable development address; a phone's localhost is not the laptop.
5. Use separate test data/credentials for automated checks. Packaged desktop users launch the installed client, not the development database or Django server themselves.

There is no runnable Compose file, Dockerfile, package script, or startup command yet. The architecture/runbook is a plan; no container or package is being built. Native Windows/macOS builds still require appropriate OS toolchains/build runners.

## Deployment Options — AWS Excluded, Provider Undecided

- **Render:** my first hosted option to evaluate for a learning demo, with a Django web service and PostgreSQL. Check current plan limits, regional availability, database privileges/backups, and cost before choosing a plan. [Django on Render](https://render.com/docs/deploy-django).
- **Railway:** another Django/PostgreSQL deployment workflow to compare; evaluate storage/backups/networking and ongoing resource usage rather than assuming identical managed-database guarantees. [Django on Railway](https://railway.com/guides/django).
- **DigitalOcean App Platform or a VM:** a container-oriented alternative. App Platform reduces server management; a VM gives more operational control and more responsibility for TLS, patching, database protection, and backups. [Container deployment](https://docs.digitalocean.com/products/app-platform/how-to/deploy-from-container-images/).
- **Local-only Docker demo:** valid for the learning project without any public deployment. A store-LAN server is another possible topology, but secure access from outside that LAN is a separate design decision.

Do not provision accounts/resources or choose a paid plan now. Hosted API availability requires a working connection; installed apps do not become offline POS systems. Keep hosted PostgreSQL private to the backend and plan credential/backup ownership whichever provider is chosen.

## Workflows, Packaging, Signing, and Distribution Options

Recommended workflow: review docs → authorized implementation → local tests → CI checks/builds → manual demo build. This is a development sequence inside one project, not product release planning.

- **Desktop:** proposed Tauri packaging for a macOS application/DMG and Windows installer. Start with local/internal learning builds, test both OS targets, and use signed/notarized distribution if sharing beyond the controlled demo. Signing identifies the publisher and helps verify the binary; it is separate from staff login and does not guarantee the app has no bugs. Follow platform-specific requirements if public distribution is chosen. Sources: [Tauri distribution](https://v2.tauri.app/distribute/), [macOS signing](https://v2.tauri.app/distribute/sign/macos/), [Windows signing](https://v2.tauri.app/distribute/sign/windows/).
- **Mobile:** if Expo is accepted, use development builds/local native toolchains, with EAS Build/internal distribution as an option. Android emulator/device and iOS simulator testing are proposed within the same learning project. Public app-store submission, paid developer accounts, automatic updates, and signing-service enrollment are not required now. [Expo build workflow](https://docs.expo.dev/develop/development-builds/introduction/).
- **Automated updates:** unnecessary for the initial learning deliverable; manual demo installation is enough unless the user chooses to learn update signing/delivery explicitly.

## What Remains Open Before Code

Confirm the desktop shell (Tauri recommended), proposed Expo tooling, mobile POS device/register binding and refund handoff, the token-auth design/library, and detailed screen/feature behavior. Hosting may remain local/undecided if that is the accepted project scope. Finish architecture/data-dictionary/invariant reviews before treating DBML as final. The earlier “adapt model, then begin migrations” instruction is replaced by the explicit [planning gate](05-architecture-planning.md): model review now, executable migrations only after authorization.
