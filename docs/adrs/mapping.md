# Codebase Architecture Mapping

## Project Overview

**Name**: order-management-api (from `package.json`)
**Purpose**: REST API for an Order Management System (OMS) — manages users/auth, customers, products (catalog/stock), and orders (with status workflow and stock movement).
**Type**: Backend REST API service (no frontend in this repo)
**Primary Language**: TypeScript (ESM, `"type": "module"`)
**Primary Framework**: Express 4
**Repository Scale**: Small — ~30 TypeScript files under `src/`, ~1,500 LOC in `src/` + `shared`, single service, no monorepo/microservices split.

The project is part of a "meeting-to-design-docs" exercise: `TRANSCRICAO.md` (raw meeting transcript) and `docs/others/ata-reuniao-webhooks.md` (cleaned meeting minutes) were used to derive `docs/PRD.md`, `docs/RFC.md`, and `docs/FDD.md` (the latter two currently placeholders). This mapping is based purely on the current state of the codebase (`src/`, `prisma/`, `tests/`, config files), not on the aspirational docs.

## Technology Stack

**Language & Runtime**
- TypeScript 5.6 (strict, ESM `NodeNext` module resolution)
- Node.js >= 20 (`engines.node` in `package.json`)
- `tsx` for dev/watch execution and script running

**Web Framework**
- Express 4.21 — HTTP server, routing, middleware pipeline

**Database & ORM**
- MySQL 8.0 (via `docker-compose.yml`, `oms-mysql` service)
- Prisma ORM 5.22 (`@prisma/client`, `prisma` CLI) — schema at `prisma/schema.prisma`, migrations at `prisma/migrations/`

**Authentication & Security**
- `jsonwebtoken` 9.0 — JWT issuance/verification (Bearer tokens)
- `bcrypt` 5.1 — password hashing
- Custom role-based access control (`ADMIN` / `OPERATOR` roles) via `requireRole` middleware

**Validation**
- `zod` 3.23 — request schema validation (per-module `*.schemas.ts`) and environment variable validation (`src/config/env.ts`)

**Logging**
- `pino` 9.5 + `pino-http` 10.3 — structured JSON logging and HTTP request logging middleware
- `pino-pretty` (dev dependency) — human-readable logs in development

**Testing**
- `vitest` 2.1 — test runner (`tests/*.test.ts`)
- `supertest` 7.0 — HTTP assertion library for integration tests against the Express app

**Tooling**
- ESLint 8 + `@typescript-eslint` — linting
- Prettier 3.3 — formatting
- `uuid` 11.0 — UUID generation (used alongside Prisma's `@default(uuid())`)

**Infrastructure**
- Docker Compose (`docker-compose.yml`) — local MySQL 8.0 container with health check, custom charset/collation, persistent volume (`oms_mysql_data`)
- No cloud provider configuration detected (no Terraform/CloudFormation/K8s manifests)
- No message queue, cache layer, or external API integrations detected in current code (webhooks are discussed in meeting docs/PRD but not yet implemented in `src/`)

## System Modules

### Module Index

1. **AUTH** - Authentication & Authorization: Login/registration, JWT issuance, auth middleware, RBAC
2. **USERS** - User Management: CRUD for internal system users (ADMIN/OPERATOR)
3. **CUSTOMERS** - Customer Management: CRUD for customer records
4. **PRODUCTS** - Product Catalog: CRUD for products, stock quantity tracking
5. **ORDERS** - Order Management: Order creation, status workflow, stock debit/replenish, order numbering
6. **API-CORE** - API Application Core: Express app bootstrap, dependency wiring, routing aggregation, shared middleware, error handling, HTTP response helpers
7. **DATA** - Data Layer: Prisma schema, migrations, database connection, seed data
8. **CONFIG** - Configuration & Environment: Environment variable loading/validation

---

### AUTH: Authentication & Authorization

**Purpose**: Authenticates users via email/password login, issues JWTs, and enforces role-based access control on protected routes.

**Location**: `src/modules/auth/*`, `src/middlewares/auth.middleware.ts`

**Key Components**:
- `AuthService` — login (bcrypt password check, JWT signing), delegates registration to `UserService`
- `AuthController` — HTTP handlers for `/auth/register`, `/auth/login`
- `auth.middleware.ts` — `authenticate` (verifies Bearer JWT, attaches `req.user`) and `requireRole(...roles)` (RBAC guard)
- `auth.schemas.ts` — Zod schemas for register/login payloads

**Technologies**: `jsonwebtoken` (JWT sign/verify), `bcrypt` (password hashing), `zod` (input validation)

**Dependencies**:
- Internal: USERS (via `UserRepository`, `UserService`), CONFIG (`env.JWT_SECRET`, `env.JWT_EXPIRES_IN`), shared errors (`UnauthorizedError`, `ForbiddenError`)
- External: none (no external identity provider)

**Patterns**: Layered architecture (controller → service → repository), stateless JWT auth, middleware-based RBAC, custom Express `Request.user` augmentation via TypeScript module declaration

**Key Files**:
- `src/modules/auth/auth.service.ts` — login logic, token signing
- `src/middlewares/auth.middleware.ts` — request-level auth/RBAC enforcement (used across all protected routers)
- `src/modules/auth/auth.controller.ts`, `src/modules/auth/auth.routes.ts`, `src/modules/auth/auth.schemas.ts`

**Scope**: Small — 4 files in module + 1 shared middleware (~150 LOC)

---

### USERS: User Management

**Purpose**: Manages internal system users (the actors who operate the OMS — admins and operators), separate from `CUSTOMERS`.

**Location**: `src/modules/users/*`

**Key Components**:
- `UserRepository` — Prisma-backed data access (`findByEmail`, CRUD)
- `UserService` — user creation (hashes password), `toPublic` (strips `passwordHash` from responses)
- `UserController` / `user.routes.ts` — REST endpoints
- `user.schemas.ts` — Zod validation schemas

**Technologies**: Prisma Client, `bcrypt` (password hashing on creation), `zod`

**Dependencies**:
- Internal: DATA (Prisma `User` model), CONFIG, shared errors/http response helpers
- External: none
- Consumed by: AUTH (registration/login flow reuses `UserRepository`/`UserService`)

**Patterns**: Repository pattern, layered architecture, DTO-style "public user" projection to avoid leaking password hashes

**Key Files**: `src/modules/users/user.service.ts`, `src/modules/users/user.repository.ts`, `src/modules/users/user.controller.ts`

**Scope**: Small — 4 files (~150-200 LOC estimate)

---

### CUSTOMERS: Customer Management

**Purpose**: Manages customer records (name, email, phone, document, JSON address) referenced by orders.

**Location**: `src/modules/customers/*`

**Key Components**:
- `CustomerRepository`, `CustomerService`, `CustomerController`, `customer.schemas.ts`, `customer.routes.ts`

**Technologies**: Prisma Client, `zod`

**Dependencies**:
- Internal: DATA (Prisma `Customer` model, indexed on `document`), shared errors/http helpers
- External: none
- Consumed by: ORDERS (order creation validates customer existence, embeds customer summary in response)

**Patterns**: Layered architecture (controller/service/repository), Zod-validated request schemas

**Key Files**: `src/modules/customers/customer.service.ts`, `src/modules/customers/customer.repository.ts`

**Scope**: Small — 5 files

---

### PRODUCTS: Product Catalog

**Purpose**: Manages the product catalog including SKU, pricing (in cents), stock quantity, and active/inactive status.

**Location**: `src/modules/products/*`

**Key Components**:
- `ProductRepository`, `ProductService`, `ProductController`, `product.schemas.ts`, `product.routes.ts`

**Technologies**: Prisma Client, `zod`

**Dependencies**:
- Internal: DATA (Prisma `Product` model, indexed on `active`/`name`), shared errors/http helpers
- External: none
- Consumed by: ORDERS (stock debit/replenish on status transitions, price snapshot at order-item creation, inactive-product validation)

**Patterns**: Layered architecture, money represented as integer cents (`priceCents`) to avoid floating-point errors, stock quantity managed via atomic Prisma `increment`/`decrement`

**Key Files**: `src/modules/products/product.service.ts`, `src/modules/products/product.repository.ts`

**Scope**: Small — 5 files

---

### ORDERS: Order Management

**Purpose**: Core business module — order creation (with line items, discount, subtotal/total calculation), a finite-state order status workflow (PENDING → PAID → PROCESSING → SHIPPED → DELIVERED, plus CANCELLED), stock debit/replenishment tied to status transitions, order status history/audit trail, and sequential human-readable order numbering (`ORD-000001`).

**Location**: `src/modules/orders/*`

**Key Components**:
- `OrderService` — largest/most complex service in the codebase; orchestrates multi-table Prisma transactions (`$transaction`) spanning `Order`, `OrderItem`, `OrderStatusHistory`, `Product`, `OrderNumberSequence`
- `order.status.ts` — status transition state machine (`canTransition`, `shouldDebitStock`, `shouldReplenishStock`)
- `OrderRepository`, `OrderController`, `order.schemas.ts`, `order.routes.ts`

**Technologies**: Prisma Client (transactions, atomic increment/decrement), `zod`

**Dependencies**:
- Internal: CUSTOMERS, PRODUCTS (validated/mutated within order transactions), DATA, shared errors (`InsufficientStockError`, `InvalidStatusTransitionError`, `ConflictError`, etc.), shared HTTP `paginated()` response helper
- External: none

**Patterns**: Transactional consistency via `prisma.$transaction`, explicit finite-state-machine for status transitions (decoupled into `order.status.ts`), audit-log pattern (`OrderStatusHistory`), sequence-table pattern for human-readable IDs (`OrderNumberSequence` with upsert-based reservation — a potential contention point under concurrency), domain-specific error types

**Key Files**:
- `src/modules/orders/order.service.ts` (256 lines — largest file in the codebase)
- `src/modules/orders/order.status.ts` — encodes the order lifecycle state machine
- `prisma/schema.prisma` — `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence` models

**Scope**: Medium — 6 files, most business logic and complexity concentrated here

---

### API-CORE: API Application Core

**Purpose**: Application bootstrap and cross-module wiring — builds the Express app, composes dependency graph (repositories → services → controllers) manually (no DI framework), aggregates all module routers, applies global middleware, and starts the HTTP server.

**Location**: `src/app.ts`, `src/server.ts`, `src/routes/index.ts`, `src/middlewares/*` (excluding `auth.middleware.ts`, covered under AUTH)

**Key Components**:
- `buildControllers()` / `buildApp()` in `src/app.ts` — manual constructor-based dependency injection
- `src/server.ts` — process entrypoint (creates `PrismaClient`, calls `buildApp`, listens on `env.PORT`)
- `src/routes/index.ts` — mounts all module routers under `/api/v1`
- `request-logger.middleware.ts` (pino-http), `error.middleware.ts` (centralized error handler mapping `AppError` subclasses to HTTP responses), `validate.middleware.ts` (Zod-based request validation middleware factory)

**Technologies**: Express, `pino-http`, `zod`

**Dependencies**:
- Internal: all five feature modules (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS), DATA (`PrismaClient` instantiation), CONFIG
- External: none

**Patterns**: Manual/explicit dependency injection (constructor injection, no IoC container), centralized error-handling middleware translating a custom `AppError` hierarchy into consistent JSON error responses, health-check endpoint (`GET /health`), API versioning via URL prefix (`/api/v1`)

**Key Files**: `src/app.ts`, `src/server.ts`, `src/middlewares/error.middleware.ts`, `src/routes/index.ts`

**Scope**: Small — 6 files, but architecturally central (every module passes through here)

---

### DATA: Data Layer

**Purpose**: Defines the persistent data model and manages database schema evolution and connectivity.

**Location**: `prisma/*`, `src/config/database.ts`

**Key Components**:
- `prisma/schema.prisma` — models: `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`; enums `UserRole`, `OrderStatus`
- `prisma/migrations/` — versioned SQL migrations (single `init` migration so far)
- `prisma/seed.ts` — seed script (`db:seed` npm script)
- `src/config/database.ts` — `PrismaClient` instantiation/export

**Technologies**: Prisma ORM 5.22, MySQL 8.0 (`@db.Char(36)` UUIDs, `@db.VarChar`, JSON column for address, cents-based integer money columns)

**Dependencies**:
- Internal: consumed by every feature module (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS) and API-CORE
- External: MySQL database (via `DATABASE_URL`, provisioned by `docker-compose.yml` in local dev)

**Patterns**: Schema-first ORM modeling, explicit `@@map`/`@@index` for table/column control, soft business keys (`orderNumber`, `sku`) distinct from UUID primary keys, audit trail table (`OrderStatusHistory`), sequence table for atomic counter generation

**Key Files**: `prisma/schema.prisma`, `prisma/migrations/20260519182739_init/migration.sql`, `src/config/database.ts`

**Scope**: Small — schema + migrations + 1 config file, but foundational to the entire system

---

### CONFIG: Configuration & Environment

**Purpose**: Loads and validates process environment variables at startup, failing fast on misconfiguration.

**Location**: `src/config/env.ts`, `.env.example`

**Key Components**:
- `env.ts` — Zod schema (`NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`) parsed from `process.env`, exits process on invalid config

**Technologies**: `zod`, Node's native `--env-file` flag (via `tsx watch --env-file=.env` / `node --env-file=.env`) instead of `dotenv`

**Dependencies**:
- Internal: consumed by AUTH (JWT secret/expiry), API-CORE (`PORT`), DATA (`DATABASE_URL`)
- External: none

**Patterns**: Fail-fast startup validation, typed environment config (single source of truth `Env` type inferred from Zod schema)

**Key Files**: `src/config/env.ts`

**Scope**: Small — single file, but every other module depends on it indirectly

---

## Cross-Cutting Concerns

**Infrastructure**
- Local dev database provisioned via Docker Compose (single `mysql:8.0` service, health-checked, persistent named volume). No production infrastructure-as-code present in the repo.

**Authentication & Authorization**
- Centralized in AUTH module: stateless JWT Bearer tokens, `authenticate` + `requireRole` Express middlewares applied per-router (e.g., admin-only endpoints via `requireRole('ADMIN')`). Two roles: `ADMIN`, `OPERATOR`.

**Data Layer**
- Single Prisma schema/`PrismaClient` shared across all modules (imported once in `server.ts`, injected into each repository). All repositories follow the same constructor-injection pattern (`constructor(prisma: PrismaClient)`).
- Multi-step business operations (order creation, status changes) use `prisma.$transaction` to guarantee atomicity across `Order`, `OrderItem`, `OrderStatusHistory`, and `Product` stock updates.

**API Layer**
- All routes mounted under `/api/v1` (`src/routes/index.ts`). Consistent per-module structure: `*.routes.ts` (Express `Router` factory) → `*.controller.ts` (HTTP glue) → `*.service.ts` (business logic) → `*.repository.ts` (Prisma access).
- Centralized error handling: a custom `AppError` base class (`src/shared/errors/app-error.ts`) with subclasses (`NotFoundError`, `UnauthorizedError`, `ForbiddenError`, `ConflictError`, `ValidationError`, `InsufficientStockError`, `InvalidStatusTransitionError`, `UnprocessableEntityError`, etc.) caught by `error.middleware.ts` and translated into consistent `{ errorCode, message, details }` JSON responses.
- Consistent pagination helper (`src/shared/http/response.ts` → `paginated()`) used at least by ORDERS listing.
- Request validation standardized via a shared `validate.middleware.ts` that wraps Zod schemas per route.

**Logging & Observability**
- Structured logging via `pino`, HTTP access logging via `pino-http` (`request-logger.middleware.ts`), pretty-printed in development via `pino-pretty`. No external log aggregation, tracing, or metrics system detected.

**Testing**
- `vitest` + `supertest` integration-style tests (`tests/auth.test.ts`, `tests/orders.test.ts`) exercise the Express app end-to-end against a real/test database; `tests/helpers/factories.ts` provides test data builders; `tests/setup.ts` configures the test environment.

**Money & Numeric Conventions**
- All monetary values stored/handled as integer cents (`priceCents`, `subtotalCents`, `totalCents`, `discountCents`, `unitPriceCents`) — a deliberate convention avoiding floating-point currency bugs, applied consistently across PRODUCTS and ORDERS.

**Documentation Process (meta, not runtime architecture)**
- The repo also functions as a design-docs exercise: `TRANSCRICAO.md` → `docs/others/ata-reuniao-webhooks.md` → `docs/PRD.md` / `docs/RFC.md` / `docs/FDD.md` / `docs/adrs/`. `RFC.md` and `FDD.md` are currently placeholders ("documento a ser elaborado"), and `docs/PRD.md` should be checked in Phase 2 for planned features (e.g., webhooks) not yet reflected in `src/`.

---

## Guidance for Phase 2

- Suggested analysis order given size and centrality: **DATA** and **API-CORE** first (foundational, cross-cutting), then **AUTH**, then **ORDERS** (most business complexity), then **PRODUCTS**/**CUSTOMERS**/**USERS** (similar CRUD shape — may be analyzed together or quickly).
- Given the small size of this codebase (~30 files, ~1,500 LOC), all 8 modules could reasonably be analyzed in a single Phase 2 pass if desired, well under the 100-150 file warning threshold.
- Strong Step-0 candidates already visible from this mapping: Express (framework), Prisma/MySQL (ORM + infrastructure service), REST over `/api/v1` (API protocol), JWT-based auth (domain-critical for a user-facing API).
