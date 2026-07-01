# order-management-api - Project Overview

**Generated on**: 2026-06-30 23:58:00

## Summary

`order-management-api` is a small, well-structured Order Management System (OMS) REST API built with Node.js 20+, TypeScript (strict, ESM), Express 4, and Prisma ORM 5 against MySQL 8.0. The codebase follows a layered/modular-monolith architecture: five domain modules (`auth`, `users`, `customers`, `products`, `orders`) each structured as Controller → Service → Repository, wired together via manual constructor-based dependency injection in `src/app.ts`. There is a single external integration surface (MySQL via Prisma) — no third-party APIs, queues, or caches.

The dependency audit found 25 direct dependencies (8 runtime, 17 dev), all on permissive MIT/Apache-2.0 licenses with no deprecated or unmaintained packages. It surfaced one critical vulnerability chain in the `vitest` devDependency (RCE/arbitrary file read, fixed only by a major upgrade), a high-severity ReDoS/DoS chain in `express` (fixable via non-breaking patch), a moderate vulnerability in `uuid`, and an unused direct dependency (`pino-http`) that is never imported in `src`.

Deep-dive analysis of the 11 core business-logic components (Auth, User, Customer, Product, and Order controllers/services, plus the order status state machine) confirms `OrderService` as the architectural and business center of gravity: it is the only component that bypasses its own repository and other modules' repositories to write directly against Prisma inside `$transaction` blocks, orchestrating order creation, stock debit/replenishment, and order-number sequencing. Across nearly all analyzed components, a recurring theme is thin/absent automated test coverage outside integration tests for `auth` and `orders`, with `customers` and `products` having no dedicated tests at all.

## Architecture Overview

- **Pattern**: Layered architecture (Controller → Service → Repository) per domain module, package-by-feature/modular-monolith style, manual composition root (no DI framework).
- **Cross-cutting concerns**: Authentication/authorization, validation, error handling, and structured logging are implemented as Express middleware (`src/middlewares/`); reusable primitives (error taxonomy, pagination helper, logger) live in `src/shared/`.
- **Most architecturally significant component**: `OrderService` (`src/modules/orders/order.service.ts`) — highest complexity, highest coupling, and the only component using raw `PrismaClient.$transaction` directly rather than through its repository.
- **Primary single point of failure**: `PrismaClient` / MySQL — one shared client instance, no connection pooling or retry/circuit-breaker logic, used by all 5 modules.
- **Testing/Infra**: Vitest + Supertest integration tests run against a real MySQL database (via Docker Compose, DB-only); no application Dockerfile, no CI/CD configuration found.
- **Documentation**: `docs/` (PRD.md, RFC.md, FDD.md, ADRs) exists but contains only placeholder content; no additional architectural intent could be cross-referenced.

## Dependencies Health

- 25 direct dependencies audited (8 runtime, 17 dev); all versions verified against npm registry, GitHub Advisory Database, and GitHub repo metadata.
- **Critical**: `vitest@2.1.4` — two critical CVEs (RCE / arbitrary file read); fix requires major upgrade to `4.1.9`.
- **High**: `express@4.21.1` — ReDoS via transitive `path-to-regexp`, plus a `qs`/`body-parser` DoS chain; fixable via non-breaking patch to `4.22.2`. High-severity `tar` advisories reachable transitively through `bcrypt`'s native-build tooling.
- **Moderate**: `uuid@11.0.3` (buffer bounds check, fix at `11.1.1`); `tsx@4.19.2` (esbuild/vite chain, fix at `4.22.4`).
- **Dead dependency**: `pino-http` declared but never imported anywhere in `src` — the project reimplements equivalent logging manually.
- **Legacy majors**: `@prisma/client`/`prisma` (2 majors behind), `express`, `zod`, `typescript`, `eslint` (1-2 majors behind each).
- No deprecated or unmaintained packages; no copyleft licenses; no project `LICENSE` file present.

## Components Analyzed

- **AuthController / AuthService** — Registration, login, and `/auth/me`; issues JWTs, delegates user creation to `UserService`. Flags an unauthenticated registration endpoint that accepts a client-supplied `role` field with no restriction.
- **UserController / UserService** — Single `GET /users/:id` (ADMIN-only) endpoint plus shared user-creation logic used by Auth. No dedicated test file for the Users module; only indirect coverage via `/auth/me`.
- **CustomerController / CustomerService** — Full CRUD with email-uniqueness enforcement and pagination/search. No automated tests exist for the Customers module at all; only used as a Prisma-direct fixture in order tests.
- **ProductController / ProductService** — Full CRUD with SKU-uniqueness enforcement. Zero dedicated test coverage; `OrderService` bypasses this module's repository/service entirely to mutate `Product.stockQuantity` directly.
- **OrderController / OrderService** — The system's core domain logic: server-authoritative pricing, referential integrity checks, a 6-state order status machine, stock debit/replenishment, and sequential order numbering via a singleton `OrderNumberSequence` row. Has the most integration test coverage of any module (8 tests) but no unit tests and no concurrency/race-condition tests, despite check-then-write patterns in stock and sequence handling.
- **order.status.ts** — Pure-function state machine encoding the order status transition graph (7 valid transitions of 36 possible pairs) and stock-impact rules. No dedicated unit tests exist despite being trivially unit-testable; two exported functions have zero callers anywhere in the codebase.

## Critical Findings

### Security Risks
- Single symmetric `JWT_SECRET` used for signing and verifying all tokens, with no rotation or revocation mechanism.
- Unauthenticated `POST /auth/register` accepts a client-supplied `role` field with no restriction, a potential privilege-escalation path to `ADMIN`.
- Coarse-grained RBAC: only `GET /users/:id` enforces `ADMIN`-only access; all `/customers`, `/products`, and `/orders` routes are accessible to any authenticated user regardless of role.
- No rate-limiting/brute-force protection on `/auth/login` or `/auth/register`.
- No CORS middleware and no `helmet`-equivalent security-headers middleware found.
- `vitest` critical RCE/arbitrary-file-read CVEs (devDependency) and `express` ReDoS/DoS CVE (production dependency, non-breaking fix available).

### Technical Debt
- `OrderService` bypasses `CustomerRepository`/`ProductRepository` and writes directly to `Product.stockQuantity` and `OrderNumberSequence` inside its own transaction, coupling the Orders module tightly to the internal schema of Customers and Products.
- No repository interfaces/abstractions anywhere: all services depend on concrete repository classes; `OrderService` additionally depends directly on `PrismaClient`.
- `pino-http` declared as a direct production dependency but never used.
- No dedicated automated tests for the Customers or Products modules; no unit tests for the Orders status state machine or for any service in isolation.
- `error.middleware.ts` directly imports and switches on Prisma-specific error codes, leaking persistence-technology awareness into the HTTP layer.
- No application Dockerfile or CI/CD pipeline configuration found anywhere in the repository.

### Single Points of Failure
- `PrismaClient` / MySQL: one shared connection, one non-clustered database instance, no pooling or retry/circuit-breaker logic — any DB unavailability halts all 5 modules simultaneously.
- `authenticate` middleware: every protected route in the system depends on this single function; a defect affects `/users`, `/customers`, `/products`, `/orders`, and `/auth/me` simultaneously.
- `OrderNumberSequence` table: a single-row counter contended on every order creation, a potential serialization bottleneck/deadlock point under concurrent load.
- `express` and `@prisma/client`: both used across nearly the entire `src` tree with no abstraction layer, concentrating the blast radius of any breaking change or critical vulnerability in either.

## Reports Index

See [MANIFEST.md](./MANIFEST.md) for complete list of all generated reports.
