# Architectural Analysis Report

**Project:** Order Management System REST API
**Scope analyzed:** `src/` (with supporting context from `prisma/`, `tests/`, `docker-compose.yml`, `package.json`, and `docs/`)
**Generated on:** 2026-06-30

---

## 1. Executive Summary

This project is a small, well-structured **Order Management System (OMS) REST API** built with **Node.js 20+, TypeScript (strict mode, ESM), Express 4, and Prisma ORM 5** against a **MySQL 8.0** database. The codebase follows a **layered/modular monolith architecture**, organizing business capabilities into five self-contained feature modules (`auth`, `users`, `customers`, `products`, `orders`) under `src/modules/`, each internally structured as **Controller → Service → Repository**, a classic 3-tier layered pattern applied per-module (a "vertical slice" or "package-by-feature" organization).

Dependency wiring is done via **manual constructor-based dependency injection** in `src/app.ts` (`buildControllers`), with no DI container/framework. Cross-cutting concerns (authentication/authorization, validation, error handling, structured logging) are implemented as **Express middleware** in `src/middlewares/`, and generic reusable primitives (error taxonomy, pagination helper, logger) live in `src/shared/`.

The system exposes a single external integration surface: a **MySQL relational database** accessed exclusively through Prisma. There are no external third-party APIs, no message queues, no caching layer, and no separate microservices — this is a single-process, single-deployable monolith. Authentication is self-contained **JWT-based** (HS256, signed with a shared secret, `jsonwebtoken` + `bcrypt`).

The most architecturally significant and highest-risk component is `OrderService` (`src/modules/orders/order.service.ts`), which orchestrates cross-entity transactional business logic (order creation, stock debit/replenishment, status-machine transitions) directly against Prisma using raw `$transaction` calls — it is the most complex, most coupled, and most business-critical unit in the system. The database (`PrismaClient`, `src/config/database.ts`) is the single most depended-upon component and the system's primary single point of failure, as there is no connection pooling configuration, no retry/circuit-breaker logic, and a single shared client instance is passed everywhere.

The project includes automated integration tests (`vitest` + `supertest`) that exercise the real database, a seed script, and Docker Compose for local MySQL provisioning, but **no Dockerfile for the application itself**, no CI/CD configuration, and no Kubernetes/infrastructure-as-code — indicating the containerization/deployment story is currently limited to the database dependency only. The `docs/` folder (PRD.md, RFC.md, FDD.md, ADRs) is present but contains only placeholder/empty content, so no additional formal architectural documentation could be cross-referenced.

---

## 2. System Overview

```
mba-ia-desafio-design-docs-com-ia/
├── src/
│   ├── app.ts                       # Composition root: builds Express app, wires DI, mounts middleware/routes
│   ├── server.ts                    # Process entrypoint: bootstraps HTTP server, graceful shutdown
│   ├── config/
│   │   ├── database.ts              # PrismaClient factory/singleton
│   │   └── env.ts                   # Zod-validated environment configuration loader
│   ├── middlewares/
│   │   ├── auth.middleware.ts       # JWT authentication + role-based authorization (RBAC)
│   │   ├── error.middleware.ts      # Centralized error-handling middleware
│   │   ├── request-logger.middleware.ts  # Request/response structured logging + request-id
│   │   └── validate.middleware.ts   # Zod-based request validation (body/query/params)
│   ├── modules/                     # Domain modules (package-by-feature), each Controller→Service→Repository
│   │   ├── auth/
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.routes.ts
│   │   │   ├── auth.schemas.ts
│   │   │   └── auth.service.ts      # (no repository; delegates to UserRepository/UserService)
│   │   ├── users/
│   │   │   ├── user.controller.ts
│   │   │   ├── user.repository.ts
│   │   │   ├── user.routes.ts
│   │   │   ├── user.schemas.ts
│   │   │   └── user.service.ts
│   │   ├── customers/
│   │   │   ├── customer.controller.ts
│   │   │   ├── customer.repository.ts
│   │   │   ├── customer.routes.ts
│   │   │   ├── customer.schemas.ts
│   │   │   └── customer.service.ts
│   │   ├── products/
│   │   │   ├── product.controller.ts
│   │   │   ├── product.repository.ts
│   │   │   ├── product.routes.ts
│   │   │   ├── product.schemas.ts
│   │   │   └── product.service.ts
│   │   └── orders/
│   │       ├── order.controller.ts
│   │       ├── order.repository.ts
│   │       ├── order.routes.ts
│   │       ├── order.schemas.ts
│   │       ├── order.service.ts     # Core transactional business logic (stock, status machine)
│   │       └── order.status.ts      # Order status-transition state machine (pure functions)
│   ├── routes/
│   │   └── index.ts                 # Top-level API router, mounts all module routers under /api/v1
│   └── shared/
│       ├── errors/
│       │   ├── app-error.ts         # Base AppError class
│       │   ├── http-errors.ts       # Concrete HTTP error taxonomy (400/401/403/404/409/422 + domain errors)
│       │   └── index.ts             # Barrel export
│       ├── http/
│       │   └── response.ts          # Pagination helper/response shaping
│       └── logger/
│           └── index.ts             # Pino logger factory (redaction, pretty-printing in dev)
├── prisma/
│   ├── schema.prisma                 # Data model: User, Customer, Product, Order, OrderItem,
│   │                                  #   OrderStatusHistory, OrderNumberSequence (+ enums)
│   ├── migrations/20260519182739_init/migration.sql
│   └── seed.ts                       # Deterministic seed script (users, customers, products, orders)
├── tests/
│   ├── setup.ts                      # Vitest global setup: DB connect + per-test cleanup
│   ├── helpers/factories.ts          # Test data factories + auth/login helpers (uses supertest against buildApp)
│   ├── auth.test.ts
│   └── orders.test.ts
├── docs/
│   ├── PRD.md / RFC.md / FDD.md      # Present but empty/placeholder (3 lines each)
│   ├── TRACKER.md
│   ├── adrs/README.md                # ADR folder present, no individual ADRs recorded
│   └── others/reports/MANIFEST.md
├── docker-compose.yml                 # MySQL 8.0 service only (no app container)
├── package.json / tsconfig.json / tsconfig.build.json / vitest.config.ts / .env.example
```

**Architectural pattern identified:** Layered architecture (Controller → Service → Repository) applied per-module in a package-by-feature/modular-monolith style, with a manual composition root for dependency injection. No hexagonal/ports-and-adapters abstraction is present — services depend directly on Prisma-typed repository classes, and `OrderService` additionally depends directly on `PrismaClient` for transaction orchestration, meaning the persistence technology is not fully abstracted behind interfaces.

---

## 3. Critical Components Analysis

**On coupling metrics:** *Afferent coupling (Ca)* is the number of other components in this codebase that directly import/depend on a given component (incoming dependencies) — a proxy for how many things would break or need retesting if the component's interface changed. *Efferent coupling (Ce)* is the number of other internal components a given component itself imports/depends on (outgoing dependencies) — a proxy for how exposed the component is to changes elsewhere in the system. These were determined by static tracing of `import` statements across every file in `src/`, `prisma/`, and `tests/`, counting only first-order, intra-project dependencies (external npm packages such as `express`, `zod`, `bcrypt`, `jsonwebtoken`, `pino`, `@prisma/client` are excluded from these counts but discussed separately in Sections 5 and 7). Counts reflect direct references only (no transitive closure).

| Component | Type | Location | Afferent Coupling | Efferent Coupling | Architectural Role |
|---|---|---|---|---|---|
| PrismaClient (singleton) | Infrastructure / Data Access | `src/config/database.ts` | 8 (app.ts, server.ts, all 5 repositories, OrderService, tests/setup.ts, tests/factories.ts) | 1 (env.ts) | Central data-access gateway; single DB connection for the whole process |
| env (config) | Configuration | `src/config/env.ts` | 5 (database.ts, auth.middleware.ts, auth.service.ts, logger/index.ts, app bootstrap chain) | 0 | Validates and centralizes all runtime configuration (Zod schema) |
| app.ts (`buildApp`/`buildControllers`) | Composition Root | `src/app.ts` | 2 (server.ts, tests/factories.ts) | ~15 (all controllers/services/repositories, middlewares, routes/index, shared/errors) | Application composition root; wires manual DI graph and Express pipeline |
| server.ts | Entrypoint / Process Bootstrap | `src/server.ts` | 0 (process entrypoint) | 4 (app.ts, env.ts, database.ts, logger) | Starts HTTP server, handles graceful shutdown (SIGINT/SIGTERM) |
| routes/index.ts (`buildApiRouter`) | Routing / API Gateway (internal) | `src/routes/index.ts` | 1 (app.ts) | 5 (all 5 module route builders + controller types) | Mounts all module routers under `/api/v1` |
| auth.middleware.ts (`authenticate`, `requireRole`) | Middleware — Security | `src/middlewares/auth.middleware.ts` | 5 (auth, users, customers, products, orders routes) | 2 (env.ts, shared/errors) | JWT verification and RBAC gate for nearly all protected endpoints |
| validate.middleware.ts | Middleware — Validation | `src/middlewares/validate.middleware.ts` | 5 (all 5 module route files) | 1 (shared/errors) | Zod-based request validation for body/query/params; normalizes to ValidationError |
| error.middleware.ts | Middleware — Error Handling | `src/middlewares/error.middleware.ts` | 1 (app.ts, mounted last) | 3 (shared/errors, shared/logger, @prisma/client, zod) | Centralized exception translator (AppError, ZodError, Prisma known errors → HTTP responses) |
| request-logger.middleware.ts | Middleware — Observability | `src/middlewares/request-logger.middleware.ts` | 1 (app.ts) | 1 (shared/logger) | Assigns request ID, logs request/response with duration |
| shared/errors (AppError + http-errors + index) | Shared Kernel — Error Taxonomy | `src/shared/errors/` | ~13 (all services, all controllers indirectly, both middlewares) | 0 | Domain-agnostic exception hierarchy consumed system-wide |
| shared/http/response.ts (`paginated`) | Shared Kernel — Utility | `src/shared/http/response.ts` | 3 (CustomerService, ProductService, OrderService) | 0 | Standardizes paginated response shape across list endpoints |
| shared/logger/index.ts | Shared Kernel — Observability | `src/shared/logger/index.ts` | 3 (server.ts, error.middleware.ts, request-logger.middleware.ts) | 1 (env.ts) | Centralized Pino logger with redaction of sensitive fields |
| AuthController | Controller | `src/modules/auth/auth.controller.ts` | 1 (auth.routes.ts) | 3 (AuthService, UserService, shared/errors) | HTTP entrypoint for register/login/me |
| AuthService | Service | `src/modules/auth/auth.service.ts` | 1 (AuthController) | 5 (env, shared/errors, UserRepository, UserService, auth.schemas) | Credential verification, JWT issuance; delegates user creation to UserService |
| auth.routes.ts | Router | `src/modules/auth/auth.routes.ts` | 1 (routes/index.ts) | 4 (validate, auth.schemas, AuthController, authenticate) | Defines `/auth/register`, `/auth/login`, `/auth/me` |
| auth.schemas.ts | Validation Schema | `src/modules/auth/auth.schemas.ts` | 2 (auth.routes.ts, auth.service.ts) | 0 | Zod schemas for register/login payloads |
| UserController | Controller | `src/modules/users/user.controller.ts` | 1 (user.routes.ts) | 1 (UserService) | HTTP entrypoint for `GET /users/:id` (admin-only) |
| UserService | Service | `src/modules/users/user.service.ts` | 3 (UserController, AuthController, AuthService) | 3 (UserRepository, shared/errors, bcrypt) | User creation (hashing), retrieval, public-DTO shaping; shared by Auth module |
| UserRepository | Repository | `src/modules/users/user.repository.ts` | 3 (UserService, AuthService, app.ts) | 1 (PrismaClient) | Direct Prisma access for `User` model |
| user.routes.ts | Router | `src/modules/users/user.routes.ts` | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, UserController) | Defines `GET /users/:id` with `authenticate` + `requireRole('ADMIN')` |
| user.schemas.ts | Validation Schema | `src/modules/users/user.schemas.ts` | 2 (user.routes.ts, UserService types) | 0 | Zod schemas for user id param and creation input |
| CustomerController | Controller | `src/modules/customers/customer.controller.ts` | 1 (customer.routes.ts) | 1 (CustomerService) | HTTP entrypoint for customer CRUD + list |
| CustomerService | Service | `src/modules/customers/customer.service.ts` | 1 (CustomerController) | 3 (CustomerRepository, shared/errors, shared/http/response) | Business logic: email-uniqueness checks, pagination, CRUD orchestration |
| CustomerRepository | Repository | `src/modules/customers/customer.repository.ts` | 1 (CustomerService) | 1 (PrismaClient) | Direct Prisma access for `Customer` model (search filter, pagination) |
| customer.routes.ts | Router | `src/modules/customers/customer.routes.ts` | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, customer.schemas) | Defines full CRUD routes, all behind `authenticate` |
| customer.schemas.ts | Validation Schema | `src/modules/customers/customer.schemas.ts` | 2 (customer.routes.ts, CustomerService) | 0 | Zod schemas incl. nested address object |
| ProductController | Controller | `src/modules/products/product.controller.ts` | 1 (product.routes.ts) | 1 (ProductService) | HTTP entrypoint for product CRUD + list |
| ProductService | Service | `src/modules/products/product.service.ts` | 2 (ProductController; also referenced indirectly at the schema level by OrderService, not a direct import) | 3 (ProductRepository, shared/errors, shared/http/response) | Business logic: SKU-uniqueness checks, pagination, CRUD orchestration |
| ProductRepository | Repository | `src/modules/products/product.repository.ts` | 1 (ProductService) | 1 (PrismaClient) | Direct Prisma access for `Product` model |
| product.routes.ts | Router | `src/modules/products/product.routes.ts` | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, product.schemas) | Defines full CRUD routes, all behind `authenticate` |
| product.schemas.ts | Validation Schema | `src/modules/products/product.schemas.ts` | 2 (product.routes.ts, ProductService) | 0 | Zod schemas for product CRUD and list filters |
| OrderController | Controller | `src/modules/orders/order.controller.ts` | 1 (order.routes.ts) | 2 (OrderService, shared/errors) | HTTP entrypoint for order list/get/create/status-change/delete |
| OrderService | Service | `src/modules/orders/order.service.ts` | 1 (OrderController) | 6 (OrderRepository, PrismaClient directly, shared/errors, order.status, shared/http/response, order.schemas) | **Core domain logic**: transactional order creation, stock debit/replenish, status-machine enforcement, order-number sequencing. Highest complexity/business-criticality in the system |
| OrderRepository | Repository | `src/modules/orders/order.repository.ts` | 1 (OrderService) | 1 (PrismaClient) | Direct Prisma access for `Order`/list/detail with relations |
| order.routes.ts | Router | `src/modules/orders/order.routes.ts` | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, order.schemas) | Defines list/get/create/status-patch/delete routes, all behind `authenticate` |
| order.schemas.ts | Validation Schema | `src/modules/orders/order.schemas.ts` | 2 (order.routes.ts, OrderService) | 1 (@prisma/client OrderStatus enum) | Zod schemas for order creation, status update, list filters |
| order.status.ts | Domain Logic (pure state machine) | `src/modules/orders/order.status.ts` | 1 (OrderService) | 1 (@prisma/client OrderStatus enum) | Encodes the order status transition graph and stock-impact rules as pure functions |
| prisma/schema.prisma | Data Model Definition | `prisma/schema.prisma` | N/A (source of generated client used by all repositories) | 0 | Defines relational schema: User, Customer, Product, Order, OrderItem, OrderStatusHistory, OrderNumberSequence |
| prisma/seed.ts | Infrastructure — Data Seeding Script | `prisma/seed.ts` | 0 (standalone script) | 1 (@prisma/client, bcrypt) | Populates local/dev DB with deterministic sample data |
| tests/setup.ts + tests/helpers/factories.ts | Test Infrastructure | `tests/setup.ts`, `tests/helpers/factories.ts` | 0 (consumed by test files only) | 2 (config/database.ts, src/app.ts) | Test lifecycle management and reusable fixtures; builds real app instance against real DB |

---

## 4. Dependency Mapping

```
High-Level Runtime Dependency Flow:

  server.ts
     │
     ▼
  app.ts (composition root)
     │
     ├──► requestLogger middleware ──► shared/logger
     ├──► [GET /health]
     ├──► routes/index.ts (buildApiRouter)
     │        │
     │        ├─► /auth     → auth.routes    → authenticate(only /me) → AuthController → AuthService ──► UserRepository
     │        │                                                                          └──────────────► UserService ─► UserRepository
     │        ├─► /users    → user.routes    → authenticate + requireRole(ADMIN) → UserController → UserService → UserRepository
     │        ├─► /customers→ customer.routes→ authenticate → CustomerController → CustomerService → CustomerRepository
     │        ├─► /products → product.routes → authenticate → ProductController → ProductService  → ProductRepository
     │        └─► /orders   → order.routes   → authenticate → OrderController   → OrderService     → OrderRepository
     │                                                                                    │
     │                                                                                    └─► PrismaClient.$transaction
     │                                                                                          (direct cross-entity writes:
     │                                                                                           Order, OrderItem, Product.stockQuantity,
     │                                                                                           OrderStatusHistory, OrderNumberSequence)
     │
     ├──► errorMiddleware (catch-all) ──► shared/errors, shared/logger, Prisma error codes, ZodError
     └──► NotFoundError fallback

  ALL Repositories ──► PrismaClient (src/config/database.ts) ──► MySQL 8.0 (single database, single schema)

  ALL Services ──► shared/errors (domain exception taxonomy)
  Controllers ──► never touch Prisma directly (repository access is layer-isolated)
  OrderService ──► is the ONLY service that bypasses its own repository for writes and
                    operates directly on PrismaClient.$transaction, additionally reaching
                    into Product and OrderNumberSequence tables outside its own module boundary

  Cross-module coupling:
  AuthService ──► UserRepository + UserService (auth module has no own persistence; fully delegates to users module)
  OrderService ──► reaches across module boundaries into Customer and Product tables via raw Prisma transaction
                    (bypasses CustomerRepository/ProductRepository — direct schema coupling, not service-to-service)
```

---

## 5. Integration Points

| Integration | Type | Location | Purpose | Risk Level |
|---|---|---|---|---|
| MySQL 8.0 | Relational Database | `prisma/schema.prisma`, `src/config/database.ts`, `docker-compose.yml` | Sole system of record for users, customers, products, orders, order items, status history, order-number sequence | High — single point of failure, no pooling/replica config visible |
| Prisma ORM / `@prisma/client` | Data Access Layer / DB Driver | All repository files, `OrderService` | Type-safe query builder and migration tool over MySQL | Medium — version-pinned (5.22.0), tightly coupled into service layer in `OrderService` |
| JWT (jsonwebtoken, HS256) | Authentication mechanism (self-issued, not a third-party IdP) | `src/modules/auth/auth.service.ts`, `src/middlewares/auth.middleware.ts` | Stateless bearer-token authentication and role claims | High — symmetric secret (`JWT_SECRET`) shared between signing and verification; no rotation/revocation mechanism |
| bcrypt | Cryptographic library (local, not a network integration) | `src/modules/users/user.service.ts`, `src/modules/auth/auth.service.ts`, `prisma/seed.ts` | Password hashing (10 rounds) | Low |
| Pino / pino-http / pino-pretty | Logging (local sink, stdout) | `src/shared/logger/index.ts`, `src/middlewares/request-logger.middleware.ts` | Structured application logging with field redaction | Low — no external log-aggregation integration currently wired (no Sentry/Datadog/ELK shipper found) |
| Docker Compose (MySQL only) | Infrastructure / local dev dependency | `docker-compose.yml` | Provisions local MySQL instance with healthcheck | Medium — infra exists only for the DB; no equivalent for the app itself |
| Environment variables (`.env` / `.env.example`) | Configuration source | `src/config/env.ts` | Supplies DB URL, JWT secret, port, log level | High — plaintext secrets in environment; `.env.example` ships a weak default `JWT_SECRET` pattern that could be mistakenly reused |

No external third-party APIs (payment gateways, email/SMS providers, cloud storage, message queues, caches) were found anywhere in `src/`, `prisma/`, or `tests/`.

---

## 6. Architectural Risks & Single Points of Failure

| Risk Level | Component | Issue | Impact | Details |
|---|---|---|---|---|
| Critical | PrismaClient / MySQL | Single shared DB connection, single database instance, no read replica or pooling configuration visible | System-wide outage | `src/config/database.ts` instantiates one `PrismaClient` for the entire process lifetime; `docker-compose.yml` defines one non-clustered MySQL container; any DB unavailability halts all 5 modules simultaneously |
| Critical | JWT_SECRET (symmetric) | Single shared secret used for signing and verifying all tokens, no rotation, no revocation list | Full authentication bypass if leaked | `src/config/env.ts` requires only 16+ chars minimum; `.env.example` provides a human-guessable placeholder that could ship to production if unchanged; compromise of this one value grants system-wide impersonation of any role |
| High | OrderService | Business-critical, highly coupled component doing direct cross-module Prisma transaction work outside repository abstraction | Data integrity risk; hard to test/change in isolation | `order.service.ts` bypasses `CustomerRepository`/`ProductRepository` and writes directly to `Product.stockQuantity` and `OrderNumberSequence` inside its own transaction — couples the Orders module tightly to the internal schema of Customers and Products modules |
| High | OrderNumberSequence table | Single-row counter (`id=1`) used via `upsert`/`increment` as a centralized sequence generator | Serialization bottleneck / write contention under concurrent order creation | Every order creation contends on the same DB row (`reserveOrderNumber`), which under MySQL's default isolation can serialize order creation throughput and is a potential deadlock/contention point at scale |
| High | authenticate middleware (`src/middlewares/auth.middleware.ts`) | Every protected route in the system depends on this single middleware function | System-wide auth outage/bypass if defective | High afferent coupling (5 route files); a bug here affects `/users`, `/customers`, `/products`, `/orders`, and `/auth/me` simultaneously |
| Medium | No application Dockerfile / CI pipeline | Deployment path for the API service itself is undocumented in the repo | Operational/deployment risk, not purely architectural | `docker-compose.yml` only provisions MySQL; no `Dockerfile`, `.github/workflows`, or other CI/CD definitions were found in the inspected paths |
| Medium | AuthService / UserService coupling | `AuthService` directly depends on both `UserRepository` and `UserService`, duplicating the persistence dependency path | Increases ripple effect of changes to the Users module | `auth.service.ts` imports `UserRepository` for `findByEmail` and separately `UserService` for `createUser`/`toPublic`, meaning Auth has two coupling paths into Users instead of one |
| Medium | No repository interfaces/abstractions | All services depend on concrete repository classes, not interfaces; `OrderService` depends on concrete `PrismaClient` | Reduced testability/substitutability; tests run against a real DB rather than mocks | Confirmed by reading all `*.service.ts` and `*.repository.ts` files — no `interface`/port abstraction layer exists between services and Prisma |
| Low | Centralized error middleware inspects Prisma error codes directly | `error.middleware.ts` imports `Prisma.PrismaClientKnownRequestError` and switches on `P2002`/`P2025` | Leaks persistence-technology awareness into the HTTP layer | Tight coupling between the web layer and the specific ORM's error model; a database/ORM migration would require changes here |
| Low | `vitest.config.ts` forces `singleFork`/`fileParallelism: false` | Test suite cannot run in parallel | Slower feedback loop, not a production risk | Necessitated by all tests sharing one real MySQL database with destructive `beforeEach` cleanup in `tests/setup.ts` |

---

## 7. Technology Stack Assessment

- **Runtime:** Node.js ≥ 20 (per `package.json` `engines`), ESM (`"type": "module"`)
- **Language:** TypeScript 5.6.3, strict mode enabled (`strict`, `noUncheckedIndexedAccess`, `noImplicitOverride`, `noFallthroughCasesInSwitch`), path alias `@/*` configured but imports observed in code use relative `.js`-suffixed ESM paths rather than the alias
- **Web framework:** Express 4.21.1 — classic middleware-based HTTP framework; routing composed via `Router()` per module
- **ORM / Data layer:** Prisma 5.22.0 with `@prisma/client` 5.22.0, targeting MySQL 8.0 via `mysql://` connection string; shadow database configured for migrations (`SHADOW_DATABASE_URL`)
- **Validation:** Zod 3.23.8, used uniformly across all modules for request body/query/params validation and for environment variable validation
- **Authentication/Security:** `jsonwebtoken` 9.0.2 (HS256 bearer tokens), `bcrypt` 5.1.1 (password hashing, 10 rounds)
- **Logging/Observability:** `pino` 9.5.0 + `pino-http`/`pino-pretty` for structured, redacted logging; no APM/tracing/metrics library present
- **Testing:** `vitest` 2.1.4 (test runner), `supertest` 7.0.0 (HTTP assertions against the real Express app), tests run against a live MySQL instance rather than mocks/in-memory DB
- **Tooling:** ESLint 8.57.1 + `@typescript-eslint` 8.13.0, Prettier 3.3.3, `tsx` 4.19.2 (dev/seed runner)
- **IDs:** `uuid` 11.0.3 used for request-ID generation (not for entity IDs — those use Prisma's `@default(uuid())`)
- **Architectural style:** Modular monolith, layered (Controller-Service-Repository) per domain module, manual dependency injection via a composition root (no IoC/DI framework such as InversifyJS or NestJS)
- **No presence of:** message brokers, caching layers (Redis/Memcached), API gateways, service mesh, GraphQL, WebSocket/real-time transport, or any third-party SaaS integration

---

## 8. Security Architecture and Risks

- **Authentication model:** Stateless JWT bearer tokens signed with a single symmetric secret (`JWT_SECRET`, HS256 implied by `jsonwebtoken` defaults). Tokens carry `sub` (user id), `email`, and `role` claims and are verified in `src/middlewares/auth.middleware.ts`. There is no token refresh mechanism, no revocation/blacklist, and no rotation strategy — a leaked `JWT_SECRET` or a leaked long-lived token (default `JWT_EXPIRES_IN=8h`) cannot be invalidated before natural expiry.
- **Authorization model:** Coarse-grained RBAC with exactly two roles (`ADMIN`, `OPERATOR`), enforced via `requireRole(...)` middleware. Only the `GET /users/:id` route currently enforces `ADMIN`-only access; all other authenticated routes (`/customers`, `/products`, `/orders`) are accessible to any authenticated user regardless of role, meaning `OPERATOR` accounts can create/delete customers, products, and orders and change order status with no additional authorization checks observed in the controllers or services.
- **Secret management:** `JWT_SECRET` and DB credentials are sourced from environment variables via `.env` (loaded with Node's native `--env-file`), validated for minimum length (16 chars) but not for entropy/strength. `.env.example` includes a human-readable placeholder secret and default MySQL credentials (`oms_user`/`oms_password`, `root_password`) that pose a risk if carried into production unchanged; no `.env` file with real secrets was found in the repository (good — not committed), but there is no evidence of a secrets manager/vault integration.
- **Password handling:** Passwords hashed with bcrypt at 10 rounds before storage (`user.service.ts`, `auth.service.ts`, `prisma/seed.ts`); plaintext passwords are never persisted. Zod enforces `min(8).max(72)` password length at registration (72 is bcrypt's input byte limit).
- **Sensitive-data redaction:** The Pino logger (`src/shared/logger/index.ts`) explicitly redacts `Authorization`/`cookie` headers and any field named `password`, `passwordHash`, `token`, `accessToken` — a positive, deliberate security control against credential leakage in logs.
- **Input validation:** All mutating endpoints validate `body`/`query`/`params` via Zod schemas through `validate.middleware.ts` before reaching controllers, reducing injection and malformed-payload risk at the boundary. Prisma's parameterized queries additionally protect against SQL injection.
- **Error response hygiene:** `error.middleware.ts` returns structured error codes/messages but does not leak stack traces to clients; internal/unexpected errors are logged server-side with full context and returned to the client only as a generic `INTERNAL_SERVER_ERROR`. However, `ConflictError` messages from Prisma `P2002` do leak the name of the unique-constrained column(s) (e.g., email/SKU) to the client, which is a minor information-disclosure consideration.
- **CORS:** No CORS middleware (e.g., `cors` package) was found configured anywhere in `src/app.ts` or elsewhere — cross-origin requests are not explicitly handled, which could either block legitimate browser clients or, if a permissive proxy/gateway is added later without review, become a gap.
- **Rate limiting / brute-force protection:** No rate-limiting middleware was found on `/auth/login` or `/auth/register`, leaving credential-stuffing and brute-force attacks against the login endpoint unmitigated at the application layer.
- **HTTP hardening:** `app.disable('x-powered-by')` is set (removes Express fingerprinting header), but no `helmet` or equivalent security-headers middleware (CSP, HSTS, X-Frame-Options, etc.) was found.
- **Body size limits:** `express.json({ limit: '1mb' })` bounds request payload size, mitigating trivial payload-based DoS.
- **Multi-tenancy / data isolation:** Not applicable — single-tenant system; no tenant-scoping logic exists or is needed given current scope.

---

## 9. Infrastructure Analysis

- **Containerization:** Only the database is containerized. `docker-compose.yml` defines a single `mysql:8.0` service (`oms-mysql`) with a named volume (`oms_mysql_data`) for persistence, a healthcheck (`mysqladmin ping`), and UTF-8MB4 charset/collation configuration. No Dockerfile exists for the Node.js API itself anywhere in the inspected paths, so there is no defined container build for the application layer.
- **Orchestration:** No Kubernetes manifests, Helm charts, or other orchestration configuration were found in the repository.
- **CI/CD:** No `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, or other pipeline definitions were found in the inspected paths — build/test/deploy automation is not currently codified in the repository.
- **Runtime deployment model (inferred from scripts):** `package.json` defines `build` (tsc → `dist/`), `start` (`node --env-file=.env dist/server.js`), and `dev` (`tsx watch`) scripts, implying the intended production runtime is a plain compiled Node.js process reading environment variables from a `.env` file or the process environment — no evidence of a process manager (PM2), systemd unit, or serverless adapter.
- **Database migrations:** Managed via Prisma Migrate (`db:migrate` → `prisma migrate dev`, `db:reset` → `prisma migrate reset --force`), with one migration currently present (`20260519182739_init`) and a configured shadow database for migration diffing.
- **Environment configuration:** `.env.example` documents all required variables (`NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `SHADOW_DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, MySQL credentials); `src/config/env.ts` performs strict runtime validation via Zod and exits the process (`process.exit(1)`) on invalid configuration, a fail-fast pattern that prevents the app from starting in a misconfigured state.
- **Health check:** A single unauthenticated `GET /health` endpoint (`src/app.ts`) returns a static `{ status: 'ok' }` — it does not verify database connectivity, so it cannot be relied upon as a true readiness probe for orchestration platforms.
- **Graceful shutdown:** `src/server.ts` handles `SIGINT`/`SIGTERM` by closing the HTTP server and disconnecting Prisma before exiting, a sound operational pattern for containerized/orchestrated environments even though no container definition currently exists for the app itself.

---

*Note: `docs/PRD.md`, `docs/RFC.md`, and `docs/FDD.md` were inspected but contain only placeholder content (3 lines each), and `docs/adrs/` contains no recorded architectural decision records — no additional architectural intent or rationale could be cross-referenced from project documentation beyond what is evidenced directly in the source code.*
