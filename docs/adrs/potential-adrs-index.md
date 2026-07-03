# Potential ADRs Index

## Analysis Progress

### Analyzed Modules
- **CONFIG**: Configuration & Environment - 2026-07-01 - 0 high, 0 medium ADRs (no candidates met the ≥75 threshold; see notes below)
- **AUTH**: Authentication & Authorization - 2026-07-01 - 1 high, 0 medium ADRs
- **USERS**: User Management - 2026-07-01 - 0 high, 0 medium ADRs (no candidates met the ≥75 threshold; see notes below)
- **DATA**: Data Layer - 2026-07-01 - 2 high, 0 medium ADRs
- **ORDERS**: Order Management - 2026-07-01 - 1 high, 0 medium ADRs
- **PRODUCTS**: Product Catalog - 2026-07-01 - 0 high, 0 medium ADRs (no candidates met the ≥75 threshold; see notes below)
- **CUSTOMERS**: Customer Management - 2026-07-01 - 0 high, 0 medium ADRs (no candidates met the ≥75 threshold; see notes below)

- **API-CORE**: API Core / Bootstrap - 2026-07-01 - 1 high, 0 medium ADRs

### Pending Analysis
_None — all 8 modules analyzed._

## High Priority ADRs (must-document/)

### Module: AUTH
| Title | Category | File |
|-------|----------|------|
| Stateless JWT Authentication with Custom RBAC (Build vs. Buy Identity) | Security / Architecture | [Link](./potential-adrs/must-document/AUTH/stateless-jwt-authentication-with-custom-rbac.md) |

### Module: DATA
| Title | Category | File |
|-------|----------|------|
| MySQL as the Primary Relational Database | Technology (Infrastructure Service) | [Link](./potential-adrs/must-document/DATA/mysql-as-primary-relational-database.md) |
| Prisma ORM as the Data Access Layer | Technology (ORM/Data Access Layer) | [Link](./potential-adrs/must-document/DATA/prisma-orm-for-data-access.md) |

### Module: ORDERS
| Title | Category | File |
|-------|----------|------|
| Transactional, State-Machine-Driven Order Lifecycle with Embedded Audit Trail | Architecture / Data Integrity | [Link](./potential-adrs/must-document/ORDERS/transactional-state-machine-driven-order-lifecycle-with-audit-trail.md) |

### Module: API-CORE
| Title | Category | File |
|-------|----------|------|
| Express.js as Primary Web Framework with Manual Dependency Injection (Composition Root) | Architecture / Technology | [Link](./potential-adrs/must-document/API-CORE/express-framework-with-manual-dependency-injection.md) |
| REST API Architecture with URL-Based Versioning (`/api/v1`) | Architecture (API Protocol/Architecture) | [Link](./potential-adrs/must-document/API-CORE/rest-api-architecture-with-url-versioning.md) |

## Medium Priority ADRs (consider/)

_None identified yet._

## Summary
- High Priority: 6 ADRs
- Medium Priority: 0 ADRs
- Total: 6 ADRs
- Modules Analyzed: 8 of 8

## Notes

### CONFIG module (analyzed 2026-07-01)
Evaluated `src/config/env.ts` (Zod-validated, fail-fast environment loading; Node's native `--env-file` instead of `dotenv`).

- Did not match any Step 0 "Positive Identification" category (not an infrastructure service, primary framework, ORM, or API protocol).
- Passed Red Flags 1/2 (not domain modeling or business workflow) but is borderline on Red Flag 3/4 (config detail / trivial implementation): single file (~28 LOC), no cross-module structural integration beyond exposing typed values, low cost to change (<1 week), and only occasional team-knowledge relevance.
- Scored using the 3-dimension rubric: Scope+Impact ~15, Cost to Change ~10, Team Knowledge ~10 → total ~35/150, well below the 75-point "consider" threshold.
- Git history is minimal (single "init repository" commit for the whole repo), so no temporal evolution signal to elevate this.
- Conclusion: **No potential ADR created for CONFIG.** This is a solid, common best-practice implementation but does not meet the bar for architectural significance (not structural enough, not costly enough to change, not broadly consequential). If the team later considers this a deliberate, debated choice (e.g., explicitly rejecting `dotenv` for operational reasons), it could be reconsidered as a small "consider/" candidate.

### AUTH module (analyzed 2026-07-01)
Evaluated `src/modules/auth/*` and `src/middlewares/auth.middleware.ts` (JWT issuance/verification, bcrypt password hashing, custom RBAC via `requireRole`).

- Matched Step 0 "Positive Identification": domain-specific critical infrastructure (Authentication, user-facing API) — guaranteed base score of 75.
- Scored using the 3-dimension rubric: Scope+Impact 25 (affects all feature modules via `authenticate`/`requireRole` middleware applied across every protected router), Cost to Change 20 (migrating to an external IdP or session-based model would take 2-6 months and touch every controller), Team Knowledge 25 (every engineer working on protected routes must understand this model) → total 145/150.
- RBAC (`requireRole`) and password hashing (`bcrypt`) were folded into the same decision per Red Flag 5 (overly granular components of one cohesive "authentication + authorization strategy"), rather than split into separate ADR files.
- Git history: single "init repository" commit (2026-06-24) introduced the pattern fully formed; no subsequent iteration to date.
- No existing ADRs found in `docs/adrs/generated/` — no duplicate/related-ADR context applies.
- Conclusion: **Potential ADR created** at `potential-adrs/must-document/AUTH/stateless-jwt-authentication-with-custom-rbac.md` (must-document priority).

### USERS module (analyzed 2026-07-01)
Evaluated `src/modules/users/*` (`user.controller.ts`, `user.service.ts`, `user.repository.ts`, `user.routes.ts`, `user.schemas.ts` — ~119 LOC total): standard CRUD for internal system users (ADMIN/OPERATOR), password hashing on creation (`bcrypt`), and `toPublic` DTO stripping of `passwordHash`.

- Did not independently match a Step 0 "Positive Identification" category. The ORM (Prisma) and authentication/password-hashing strategy are structural decisions, but they are already captured as the DATA module's `prisma-orm-for-data-access` ADR and the AUTH module's `stateless-jwt-authentication-with-custom-rbac` ADR (bcrypt hashing lives there). USERS itself is a thin, conventional CRUD consumer of those decisions, not the origin of a new one.
- Red Flag 1 (Domain Modeling): the User entity/role shape (ADMIN/OPERATOR) is a business-entity concern, not a modeling *style* decision — disqualifies a domain-modeling ADR.
- Red Flag 4 (Trivial Implementation): the module is ~119 LOC across 5 files, follows the same repository/service/controller pattern as CUSTOMERS/PRODUCTS with no unique cross-cutting integration; changing it would take days, not weeks/months.
- Scored using the 3-dimension rubric: Scope+Impact ~10 (consumed by AUTH but doesn't itself define shared infrastructure), Cost to Change ~10 (<2 weeks to alter), Team Knowledge ~10 (relevant mainly to user-management features) → total ~30/150, below the 75-point threshold.
- Git history: introduced fully formed in the single "init repository" commit (2026-06-24); no subsequent evolution to signal growing architectural weight.
- No existing ADRs in `docs/adrs/generated/` overlap with this module beyond the already-covered AUTH/DATA decisions.
- Conclusion: **No potential ADR created for USERS.** The module's structurally significant choices (ORM, password hashing/authentication strategy) are already documented under DATA and AUTH; the CRUD logic specific to USERS itself does not clear the architectural-significance bar.

### PRODUCTS module (analyzed 2026-07-01)
Evaluated `src/modules/products/product.service.ts`, `product.repository.ts`, `product.controller.ts`, `product.schemas.ts`, `product.routes.ts`, and the `Product` model in `prisma/schema.prisma`.

- No `docs/adrs/generated/` directory exists yet, so no existing formal ADRs were available for duplicate/relationship detection.
- Confirmed as a standard layered-CRUD module (`*.routes.ts` → `*.controller.ts` → `*.service.ts` → `*.repository.ts`, Zod-validated), fully consistent with the patterns already captured as standalone ADRs under DATA (Prisma ORM as the data access layer) and API-CORE (Express with manual dependency injection). Re-flagging Prisma/Express usage here would be Red-Flag-5 (overly granular / duplicative of already-scored decisions), so it was not re-scored.
- Candidate: **SKU-based product catalog with integer-cents pricing** (`priceCents: Int` in `prisma/schema.prisma:61`, validated as `z.number().int().nonnegative()` in `product.schemas.ts:9`). This is a real, deliberate technical convention (avoids floating-point currency bugs) rather than a business-entity detail, so it survives Red Flag 1. However, scored strictly from PRODUCTS-module evidence alone: Scope+Impact ~15 (only a single field here; the richer cross-module usage — subtotal/discount/total math, price snapshotting at order-item creation — lives in ORDERS), Cost to Change ~15-20 (schema/API-contract migration, but bounded to money-typed columns), Team Knowledge ~15 (relevant mainly to commerce-touching code, not universally) → total ≈45-50/150, below the 75-point threshold. Note: the ORDERS-module analysis run separately (see notes above, if present) also evaluated this convention with its own evidence and reached a similarly sub-threshold conclusion — corroborating that it does not clear the bar as a standalone ADR from either module's perspective.
- SKU uniqueness enforcement (DB `@unique` + application-level check in `product.service.ts:31-34`), search-by-name/SKU (`contains` filter in `product.repository.ts:16-21`), and the pagination usage are standard, low-blast-radius implementation details — disqualified under Red Flags 3/4 (configuration/trivial, <2 files, no cross-cutting strategic trade-off) and already governed by the shared `paginated()`/`validate.middleware.ts` helpers (API-CORE's concern, not PRODUCTS').
- Git history: single "init repository" commit (2026-06-24) for all product files and `prisma/schema.prisma`; no subsequent iteration, so no temporal-evolution signal to elevate any candidate.
- Conclusion: **No potential ADR created for PRODUCTS.** This module is a well-executed instance of already-documented architectural patterns; its one distinctive convention (integer-cents money) does not independently clear the architectural-significance bar.

### CUSTOMERS module (analyzed 2026-07-01)
Evaluated `src/modules/customers/customer.controller.ts`, `customer.repository.ts`, `customer.service.ts`, `customer.routes.ts`, `customer.schemas.ts`, and the `Customer` model in `prisma/schema.prisma`.

- No `docs/adrs/generated/` directory exists yet, so no existing formal ADRs were available for duplicate/relationship detection.
- Confirmed as a standard layered-CRUD module (`customer.routes.ts` → `customer.controller.ts` → `customer.service.ts` → `customer.repository.ts`, Zod-validated, `authenticate` middleware applied at the router level), identical in shape to USERS and PRODUCTS. The structural building blocks — Prisma ORM, Express routing/middleware wiring, Zod request validation, JWT authentication gate — are already captured as standalone ADRs under DATA and AUTH (and pending API-CORE). Re-flagging them here would be Red Flag 5 (overly granular / duplicative of already-scored decisions), so they were not re-scored.
- Candidate: **`address` stored as an embedded JSON column** on `Customer` (validated via a nested Zod object in `customer.schemas.ts:7-13`, persisted as a single JSON field rather than a normalized `Address` table/relation). Considered as a possible "schema modeling style" decision, but on inspection this is a single field on a single entity, not a repeated pattern applied elsewhere in the codebase (no other entity uses embedded JSON value objects) — it reads as a business-entity attribute choice (Red Flag 1: domain modeling, WHAT is modeled) rather than a general modeling *style*. Even if scored, it is confined to ~2 files with low cross-module blast radius (Red Flag 4: trivial implementation) — the field is only ever read/written whole and not queried by sub-fields. Scoring anyway for rigor: Scope+Impact ~10 (1-2 modules; ORDERS only embeds a customer summary, not the address), Cost to Change ~10 (schema migration to a relational table is a bounded, <2 week effort), Team Knowledge ~5 (rarely relevant outside this module) → total ≈25/150, well below threshold.
- Candidate: **Email uniqueness enforced via application-level pre-check** (`findByEmail` lookup in `customer.service.ts:30-33,45-49` before create/update, raising `ConflictError`) rather than relying solely on a DB unique constraint. This is a business validation rule (Red Flag 2: business workflow) applied identically to the USERS module's own email-uniqueness handling (already assessed as sub-threshold there) — disqualified, not scored further.
- Pagination (`list()` with `page`/`pageSize`/`$transaction` count) and search-by-name/email/document (`contains` filter in `customer.repository.ts:12-22`) are standard, low-blast-radius implementation details — disqualified under Red Flags 3/4 (trivial, <2 files) and are instances of the shared `paginated()` helper (API-CORE's concern, not CUSTOMERS').
- Git history: single "init repository" commit (2026-06-24) introduced all customer files and the `Customer` Prisma model fully formed (`git log --follow` on each file returns exactly one commit); no subsequent iteration, so no temporal-evolution signal to elevate any candidate.
- Conclusion: **No potential ADR created for CUSTOMERS.** This module is a thin, conventional CRUD consumer of already-documented architectural decisions (Prisma ORM, JWT auth, Zod validation, layered architecture); none of its module-local specifics (embedded JSON address, email-uniqueness check) clear the 75-point architectural-significance bar. This mirrors the USERS and PRODUCTS conclusions.

### DATA module (analyzed 2026-07-01)
Evaluated `prisma/schema.prisma`, `src/config/database.ts`, `docker-compose.yml`, and Prisma migration/seed tooling.

- Matched Step 0 "Positive Identification" twice: MySQL is a core infrastructure service (the sole persistent data store for every domain entity) and Prisma is the primary ORM/data-access layer — both guarantee a base score of 75 each.
- **MySQL as the Primary Relational Database** (Score 150/150): every feature module depends on it with no fallback store; schema is written with MySQL-specific type mappings (`@db.Char(36)`, `ENUM` columns); a `SHADOW_DATABASE_URL` extends the choice into migration/CI tooling.
- **Prisma ORM as the Data Access Layer** (Score 145/150): the sole data-access mechanism across all 5 feature modules, no raw SQL or alternative query path found; drives migrations, seeding, and type generation.
- Both decisions were kept as two separate ADRs rather than merged, since "which database" and "which ORM/access layer" are independently substitutable choices (e.g., Prisma could target Postgres; MySQL could be accessed via a different ORM).
- Git history: single "init repository" commit (2026-06-24); both choices are embedded deeply and consistently with no bypass.
- Conclusion: **Two potential ADRs created** at `potential-adrs/must-document/DATA/mysql-as-primary-relational-database.md` and `potential-adrs/must-document/DATA/prisma-orm-for-data-access.md` (both must-document priority).

### ORDERS module (analyzed 2026-07-01)
Evaluated `src/modules/orders/order.service.ts` (256 lines, largest file in the codebase), `order.status.ts`, `order.repository.ts`, and the `Order`/`OrderItem`/`OrderStatusHistory`/`OrderNumberSequence` models in `prisma/schema.prisma`.

- Matched Step 0 via architectural significance: a declarative finite-state-machine for order status transitions, multi-table `prisma.$transaction` atomicity (stock debit/replenish + status change + audit row, all-or-nothing), and an embedded immutable audit trail (`OrderStatusHistory`) are implemented together as one cohesive transactional-consistency strategy for the system's most business-critical write paths.
- Consolidated per Red Flag 5: the FSM, transaction boundary, audit trail, and `OrderNumberSequence` reservation were documented as **one** ADR rather than four, since they are invoked together on every order mutation to serve a single purpose (consistent order state under concurrency/failure).
- Scored 135/150: Scope+Impact high (touches `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`, plus `Product` stock and `Customer` validation across modules), Cost to Change high (redesign needed for parallel/branching workflows or async side effects), Team Knowledge high (any new status or transition-tied side effect must respect the transaction/FSM guard).
- The `OrderNumberSequence` single-row upsert concurrency bottleneck was evaluated as a standalone candidate (~35-45/150, below threshold) but folded into this ADR as a documented trade-off rather than discarded, since it lives inside the same transaction boundary.
- Integer-cents monetary representation was explicitly excluded from ORDERS scope (it originates in PRODUCTS; ORDERS only propagates/snapshots it) — noted as a candidate for a future cross-cutting or PRODUCTS-scoped ADR if ever revisited.
- Git history: single "init repository" commit (2026-06-24); the whole pattern appeared fully formed with no subsequent iteration.
- Conclusion: **Potential ADR created** at `potential-adrs/must-document/ORDERS/transactional-state-machine-driven-order-lifecycle-with-audit-trail.md` (must-document priority, score 135/150).

### API-CORE module (analyzed 2026-07-01)
Evaluated `src/app.ts`, `src/server.ts`, `src/routes/index.ts`, and the per-module Express router pattern used across all 5 feature modules.

- Matched Step 0 "Positive Identification": Express is the primary web framework/platform underlying the entire HTTP surface — guaranteed base score of 75.
- Tightly coupled to the framework choice is a deliberate avoidance of an IoC/DI container: `buildControllers()` in `app.ts` manually constructs the full repository → service → controller graph for every feature module via plain constructor calls (a "poor man's DI" / composition-root pattern). Folded into the same ADR per Red Flag 5, since both were introduced together in the same file/commit and are inseparable in explaining "how the application is built and wired."
- Scored 145/150: Scope+Impact high (100% of HTTP routes pass through this bootstrap; all 5 modules wired through `buildControllers()`/`buildApiRouter()`), Cost to Change high (migrating framework or introducing a DI container would touch every module's construction site), Team Knowledge high (every engineer adding a module or endpoint must understand this exact wiring convention).
- Two adjacent candidates (the centralized `AppError` error-handling hierarchy and the shared `pino`/`pino-http` logging setup) were evaluated but scored below the 75-point threshold on their own at the current codebase scale — not promoted to separate ADRs.
- Git history: single "init repository" commit (2026-06-24); the whole application skeleton (bootstrap, routing aggregation, manual DI, centralized error handling) was scaffolded together as one atomic choice, unchanged through 8 subsequent commits (mostly docs).
- Note: an earlier duplicate draft of this ADR (title "Express as the Primary Web Framework," covering only the framework half of this decision) was superseded/removed in favor of this consolidated version, which also documents the manual-DI/composition-root pattern.
- Conclusion: **Potential ADR created** at `potential-adrs/must-document/API-CORE/express-framework-with-manual-dependency-injection.md` (must-document priority, score 145/150).

A second, independent API-CORE decision was also identified: **REST API Architecture with URL-Based Versioning** (`/api/v1`). Evaluated `src/app.ts`, `src/routes/index.ts`, and the identical `*.routes.ts → *.controller.ts → *.service.ts → *.repository.ts` layering repeated across all 5 feature modules.

- Kept as a separate ADR from the Express/DI decision per the Step 0 framework: "which HTTP framework" and "which API protocol/style" are independently substitutable choices (could build REST, GraphQL, or gRPC on top of Express).
- Scored 130/150: Scope+Impact high (fixes the external contract for every API consumer — URL structure, versioning, JSON envelope shapes), Cost to Change high (a breaking change requires deciding how `/api/v2` coexists with `/api/v1`, or adopting an entirely different protocol), Team Knowledge high (the routes→controller→service→repository convention and shared error/pagination envelope are foundational to nearly all feature work).
- Notably consistent across all 5 independently-built modules with zero deviation since the initial commit — a strong signal of a deliberate, upfront architectural template rather than incidental convention.
- Git history: single "init repository" commit (2026-06-24); no subsequent iteration or second API version.
- Conclusion: **Potential ADR created** at `potential-adrs/must-document/API-CORE/rest-api-architecture-with-url-versioning.md` (must-document priority, score 130/150).
