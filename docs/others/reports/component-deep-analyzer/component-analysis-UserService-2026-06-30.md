# Component Deep Analysis Report: UserService

**Project**: order-management-api (Order Management System REST API)
**Component analyzed**: `UserService`
**Primary file**: `src/modules/users/user.service.ts`
**Analysis date**: 2026-06-30

---

## 1. Executive Summary

`UserService` is a small, focused domain service in the `users` module of the Order Management System REST API. It encapsulates the two core user-account use cases exposed by the application: **user creation** (used both for direct admin-driven user provisioning and for self-service registration via the `auth` module) and **user retrieval by ID** (used both by the `users` module's own controller and by the `auth` module's "current user" endpoint).

The component sits at the boundary between the HTTP/controller layer and the persistence layer (`UserRepository`, backed by Prisma/MySQL). Its two central responsibilities are:

1. Enforcing the **email-uniqueness business rule** at the application layer before insert, and hashing passwords with `bcrypt` prior to persistence.
2. Guaranteeing that raw `User` database records (which include `passwordHash`) are never returned to callers — via the `toPublic` mapping to the `PublicUser` shape.

Key findings:

- `UserService` is intentionally minimal (52 lines): it does not expose update, delete, list, or password-change operations. Its scope is limited to creation and lookup-by-id.
- It is reused directly by `AuthService` (composition, not inheritance) for the `register` and `login` use cases, making `UserService` a shared kernel between the `users` and `auth` bounded contexts.
- The static method `UserService.toPublic()` is a cross-cutting utility called from both `UserService` itself and directly from `AuthService`, which is an unusual coupling pattern (a stateless static method invoked externally on the class rather than through an instance or a separate mapper/utility module).
- There is no dedicated unit test file for `UserService`. Its behavior is validated only indirectly through HTTP-level integration tests in `tests/auth.test.ts` (registration duplicate-email rejection, public-field exposure) and the `GET /users/:id` route is not exercised by any test at all.
- The component has no direct external dependencies beyond `bcrypt`; all persistence access is delegated to `UserRepository`, keeping `UserService` cleanly decoupled from Prisma/SQL details (Dependency Inversion via constructor injection).

---

## 2. Data Flow Analysis

Two independent request flows pass through `UserService`, plus a third flow where it is used purely as a static mapping utility.

### Flow A — Create User (via `POST /api/v1/users` is NOT exposed; actual entry point is `POST /api/v1/auth/register`)

```
1. Request enters via AuthController.register (src/modules/auth/auth.controller.ts:12)
2. Body validated against registerSchema (Zod) in validate middleware (src/middlewares/validate.middleware.ts, wired in src/modules/auth/auth.routes.ts:10)
3. AuthController.register calls AuthService.register(req.body) (src/modules/auth/auth.controller.ts:14)
4. AuthService.register delegates directly to UserService.createUser(input) (src/modules/auth/auth.service.ts:28)
5. UserService.createUser checks uniqueness via UserRepository.findByEmail (src/modules/users/user.service.ts:22)
   -> if found: throws ConflictError('Email already registered', 'EMAIL_ALREADY_USED')
6. Password hashed with bcrypt.hash(input.password, 10) (src/modules/users/user.service.ts:26)
7. UserRepository.create persists the row via Prisma (src/modules/users/user.repository.ts:14-16)
8. Raw Prisma User mapped to PublicUser via UserService.toPublic (src/modules/users/user.service.ts:33, 42-51)
9. PublicUser bubbles back up through AuthService -> AuthController
10. AuthController responds 201 with PublicUser JSON body (src/modules/auth/auth.controller.ts:15)
11. Errors (ConflictError, ZodError, PrismaClientKnownRequestError P2002) intercepted by errorMiddleware (src/middlewares/error.middleware.ts) and translated to structured JSON error responses
```

### Flow B — Get User By Id (`GET /api/v1/users/:id`)

```
1. Request enters via UserController.getById (src/modules/users/user.controller.ts:7)
2. Route guarded by authenticate + requireRole('ADMIN') + params validation (id must be UUID) (src/modules/users/user.routes.ts:12-18)
3. UserController.getById calls UserService.getById(req.params.id) (src/modules/users/user.controller.ts:9)
4. UserService.getById calls UserRepository.findById (src/modules/users/user.service.ts:37)
   -> if null: throws NotFoundError('User')
5. Raw Prisma User mapped to PublicUser via UserService.toPublic (src/modules/users/user.service.ts:39)
6. UserController responds 200 with PublicUser JSON body (src/modules/users/user.controller.ts:10)
7. Errors (NotFoundError, UnauthorizedError, ForbiddenError, ValidationError) handled by errorMiddleware
```

### Flow C — "Current user" lookup (`GET /api/v1/auth/me`), and static mapping reuse in login

```
1. Request enters via AuthController.me (src/modules/auth/auth.controller.ts:30)
2. authenticate middleware decodes JWT and populates req.user (src/middlewares/auth.middleware.ts:27-47)
3. AuthController.me calls UserService.getById(req.user.id) (src/modules/auth/auth.controller.ts:33) -- same code path as Flow B step 4-5
4. Separately, during AuthService.login (src/modules/auth/auth.service.ts:31-45), after bcrypt.compare succeeds,
   UserService.toPublic(user) is invoked statically (not via the injected instance) to shape the login response (src/modules/auth/auth.service.ts:42)
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|-------------------|----------|
| Validation (uniqueness) | Email must be unique across all users; duplicate registration/creation is rejected | src/modules/users/user.service.ts:22-25 |
| Security | Passwords are never stored in plaintext; hashed with bcrypt using a fixed cost factor | src/modules/users/user.service.ts:16, 26 |
| Data exposure control | Only a whitelisted subset of user fields (`PublicUser`) is ever returned to any caller | src/modules/users/user.service.ts:7-14, 42-51 |
| Existence validation | Lookup by ID must fail explicitly (404) when the user does not exist, rather than returning null/undefined | src/modules/users/user.service.ts:36-39 |
| Role assignment | New users are assigned a role of either `ADMIN` or `OPERATOR`, sourced from validated input, with `OPERATOR` as the schema-level default | src/modules/users/user.schemas.ts:11 (enforced at the boundary, consumed as-is by UserService) |
| Input constraints (upstream, enforced before UserService is reached) | Email format/length, password length (8-72), name length (2-150) enforced by Zod schema | src/modules/users/user.schemas.ts:7-12 |

### Detailed breakdown of the business rules

---

### Business Rule: Email Uniqueness Enforcement

**Overview**:
Every user account in the system must have a globally unique email address. `UserService.createUser` enforces this at the application layer before any database write is attempted.

**Detailed description**:
When `createUser` is invoked, the very first action taken (src/modules/users/user.service.ts:22) is an application-level pre-check: `this.users.findByEmail(input.email)`. If a record is found, the method immediately throws a `ConflictError` with the message `'Email already registered'` and the machine-readable error code `EMAIL_ALREADY_USED` (src/modules/users/user.service.ts:24), short-circuiting before any password hashing or persistence work occurs. This is a deliberate "fail fast" design: expensive bcrypt hashing (cost factor 10) is skipped entirely when the request is already known to be invalid, which also has a minor but real performance/DoS-mitigation benefit, since bcrypt hashing is intentionally CPU-expensive.

This rule is doubly enforced: the database schema itself declares `email String @unique` on the `User` model (prisma/schema.prisma:27), and Prisma will raise a `P2002` unique-constraint violation if a race condition allows two concurrent requests to both pass the application-level check before either write completes. The `errorMiddleware` (src/middlewares/error.middleware.ts:37-47) has a dedicated handler for `Prisma.PrismaClientKnownRequestError` with code `P2002`, translating it into a generic `409 CONFLICT` response — but with a different error code (`CONFLICT` vs. `EMAIL_ALREADY_USED`) and message than the application-level check produces. This means the client-visible error contract differs depending on whether the race was caught by `UserService`'s explicit check or by the database's constraint under concurrent load, which is a documented inconsistency (see Technical Debt section).

This business rule is exercised by two distinct entry points that both funnel into `UserService.createUser`: the `users` module (no route currently calls `createUser` directly — see Endpoints section) and the `auth` module's `register` flow (`AuthService.register` at src/modules/auth/auth.service.ts:27-29, which is a pure passthrough to `UserService.createUser`). The integration test `tests/auth.test.ts:19-28` ("rejects register with duplicated email") confirms the rule's externally observable behavior: a 409 status with `error.code === 'EMAIL_ALREADY_USED'`.

**Rule workflow**:
```
createUser(input) called
  -> findByEmail(input.email)
     -> record exists?  YES -> throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED')  [STOP]
     -> record exists?  NO  -> continue
  -> proceed to password hashing and persistence
```

---

### Business Rule: Password Hashing with Fixed Bcrypt Cost Factor

**Overview**:
Plaintext passwords supplied during user creation are never persisted. They are hashed with `bcrypt` using a hard-coded cost factor of 10 before being handed to the repository layer.

**Detailed description**:
The module-level constant `BCRYPT_ROUNDS = 10` (src/modules/users/user.service.ts:16) governs the computational cost of the bcrypt hashing algorithm applied to every new user's password (src/modules/users/user.service.ts:26: `bcrypt.hash(input.password, BCRYPT_ROUNDS)`). A cost factor of 10 is bcrypt's commonly used default and represents a balance between resistance to brute-force/rainbow-table attacks and acceptable request latency. Because the value is a compile-time constant rather than sourced from environment configuration, it cannot be tuned per-environment (e.g., lower for local/test speed, higher for production hardening) without a code change and redeploy.

This rule interacts closely with the authentication flow: `AuthService.login` (src/modules/auth/auth.service.ts:36) independently calls `bcrypt.compare(input.password, user.passwordHash)` to verify credentials at login time. Notably, this comparison logic lives in `AuthService`, not in `UserService`, meaning password verification and password hashing are split across two components — `UserService` owns "hash on create," `AuthService` owns "compare on login." The two implicitly agree on the bcrypt algorithm and the stored hash format, but there is no shared constant or abstraction enforcing that agreement; a change to the hashing scheme in `UserService` (e.g., swapping algorithms or increasing rounds) would need to be manually kept compatible with `AuthService`'s verification logic (bcrypt's format is self-describing so this specific risk is low today, but the coupling is implicit rather than explicit).

The `password` field itself is constrained upstream by Zod validation (`z.string().min(8).max(72)`, src/modules/users/user.schemas.ts:9 and src/modules/auth/auth.schemas.ts:5) before it ever reaches `UserService`. The 72-character maximum aligns with bcrypt's own well-known 72-byte input limit, suggesting an intentional, correct design choice to prevent silent truncation.

**Rule workflow**:
```
createUser(input) called (after uniqueness check passes)
  -> bcrypt.hash(input.password, 10)  [CPU-bound, asynchronous]
  -> passwordHash produced
  -> passed to UserRepository.create() as part of persisted record
  -> plaintext password is discarded (never stored, never logged, never returned)
```

---

### Business Rule: Public Data Exposure Whitelist (Password Hash Never Leaves the Service Boundary)

**Overview**:
Regardless of entry point, no consumer of `UserService` ever receives the raw `passwordHash` or any field not explicitly listed in the `PublicUser` type.

**Detailed description**:
The `PublicUser` type (src/modules/users/user.service.ts:7-14) explicitly enumerates exactly six fields: `id`, `email`, `name`, `role`, `createdAt`, `updatedAt`. The static method `UserService.toPublic(user: User): PublicUser` (src/modules/users/user.service.ts:42-51) performs an explicit field-by-field mapping (not a spread/destructure-and-omit pattern), which is a defensive design choice: even if the Prisma `User` model gains new sensitive fields in the future, `toPublic` will not leak them unless someone deliberately adds them to the mapping. Both `createUser` and `getById` route their return values through this method before responding to callers (src/modules/users/user.service.ts:33, 39).

This rule is critical from a security standpoint since the `User` Prisma model carries `passwordHash` (prisma/schema.prisma:28) as a plain field alongside the public attributes, with no database-level or Prisma-level projection restricting its retrieval — `UserRepository.findById` and `findByEmail` both return the *entire* row (src/modules/users/user.repository.ts:6-12), including `passwordHash`. This means the responsibility for preventing password-hash leakage rests entirely on disciplined use of `UserService.toPublic` by every caller; the repository provides no defense-in-depth at the query level (e.g., no `select` clause excluding `passwordHash`).

The rule's cross-module reach is notable: `AuthService.login` calls `UserService.toPublic(user)` **statically** (src/modules/auth/auth.service.ts:42), directly on the class rather than through its injected `userService` instance, even though `AuthService` already holds a reference to a `UserService` instance (`this.userService`, injected at src/modules/auth/auth.service.ts:24). This is an inconsistent usage pattern — the static method is exported and callable independently of any instance, effectively functioning as a free utility function that happens to live on the `UserService` class. The externally observable guarantee of this rule is validated in `tests/auth.test.ts:6-17`, which explicitly asserts `res.body).not.toHaveProperty('passwordHash')` and `.not.toHaveProperty('password')` after registration.

**Rule workflow**:
```
Any UserService method that resolves a User record (createUser, getById)
  -> obtains full Prisma `User` row (including passwordHash) from UserRepository
  -> passes row through UserService.toPublic(user)
     -> extracts: id, email, name, role, createdAt, updatedAt
     -> discards: passwordHash (and any other non-whitelisted field)
  -> returns PublicUser to caller (controller, or AuthService)
```

---

### Business Rule: User Existence Must Be Explicit (404 on Missing User)

**Overview**:
Looking up a user by ID that does not exist is treated as an explicit business exception (`NotFoundError`), not a null/undefined return, ensuring callers cannot silently mishandle a missing user.

**Detailed description**:
`UserService.getById` (src/modules/users/user.service.ts:36-40) calls `UserRepository.findById(id)`, which returns `Promise<User | null>` per Prisma's `findUnique` semantics (src/modules/users/user.repository.ts:6-8). Rather than propagating the nullable type to callers, `UserService.getById` immediately checks for `null` and throws `NotFoundError('User')` (src/modules/users/user.service.ts:38), which maps to a 404 HTTP status with error code `NOT_FOUND` and message `"User not found"` via the shared `AppError`/`NotFoundError` hierarchy (src/shared/errors/http-errors.ts:27-31). This converts a data-layer nullable result into a well-defined application-layer contract: `getById` either resolves with a fully-formed `PublicUser` or rejects with a typed, catchable error — callers never need to null-check.

This rule underpins two distinct call sites with different security postures: the admin-only `GET /api/v1/users/:id` endpoint (src/modules/users/user.routes.ts, protected by `requireRole('ADMIN')`) and the self-service `GET /api/v1/auth/me` endpoint (src/modules/auth/auth.controller.ts:30-38), where the ID passed is always the JWT subject (`req.user.id`) — the currently authenticated caller's own ID. In the `/auth/me` case, a `NotFoundError` would only occur in an edge case where a valid JWT references a user that has since been deleted from the database (no deletion endpoint currently exists in this codebase, based on the files analyzed, but the defensive check still protects against that possibility, e.g., from external data changes).

The `errorMiddleware` (src/middlewares/error.middleware.ts:14-24) generically handles any `AppError` subclass, so `NotFoundError` requires no special-casing there; it is automatically serialized to `{ error: { code: 'NOT_FOUND', message: 'User not found' } }` with HTTP 404.

**Rule workflow**:
```
getById(id) called
  -> UserRepository.findById(id)
     -> row found?  NO  -> throw NotFoundError('User')  [-> 404 NOT_FOUND]  [STOP]
     -> row found?  YES -> continue
  -> map row via toPublic()
  -> return PublicUser
```

---

### Business Rule: Role Assignment on Creation

**Overview**:
Every created user is assigned exactly one role, `ADMIN` or `OPERATOR`, with `OPERATOR` as the implicit default when not specified by the caller.

**Detailed description**:
`UserService.createUser` does not itself contain role-assignment logic; it trusts `input.role` (src/modules/users/user.service.ts:31) as supplied by the already-validated `CreateUserInput`. The actual default and constraint are declared declaratively in the Zod schema: `role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')` (src/modules/users/user.schemas.ts:11, mirrored in src/modules/auth/auth.schemas.ts:7 for the registration path). This means `UserService` is intentionally "dumb" with respect to role policy — it has no notion of which roles are permissible for self-registration versus admin-provisioned creation, and no authorization check exists in `UserService.createUser` to prevent, for example, an anonymous self-registration request from requesting `role: 'ADMIN'`.

This is architecturally significant: the `POST /api/v1/auth/register` route (src/modules/auth/auth.routes.ts:10) has **no authentication or role-check middleware** ahead of it (unlike `GET /api/v1/users/:id`, which requires `ADMIN`), yet it accepts an optional `role` field that, if set to `'ADMIN'`, would be honored by `UserService.createUser` without further validation. This is flagged in the Technical Debt & Risks section as a potential privilege-escalation-via-self-registration concern, since the enforcement of "who is allowed to create an ADMIN" is not visible anywhere in the analyzed code path for the public registration endpoint.

The Prisma schema reinforces the two-role model with a `UserRole` enum (`ADMIN`, `OPERATOR`; prisma/schema.prisma:11-14) and a database-level default of `OPERATOR` (prisma/schema.prisma:30), providing a secondary layer of validation/default if the value is ever written outside of the Zod-validated path (e.g., via a seed script or migration).

**Rule workflow**:
```
CreateUserInput arrives at UserService.createUser (already Zod-validated upstream)
  -> input.role is either 'ADMIN' or 'OPERATOR' (or defaulted to 'OPERATOR' by Zod before reaching the service)
  -> role passed as-is to UserRepository.create()
  -> no additional authorization/business check performed by UserService itself
```

---

## 4. Component Structure

```
src/modules/users/
├── user.service.ts       # UserService — core business logic: createUser, getById, toPublic mapping (ANALYZED COMPONENT)
├── user.repository.ts     # UserRepository — Prisma-backed data access (findById, findByEmail, create)
├── user.controller.ts     # UserController — Express request handler; delegates to UserService.getById
├── user.routes.ts         # buildUserRouter — wires GET /:id with authenticate + requireRole('ADMIN') + param validation
└── user.schemas.ts        # Zod schemas — createUserSchema, userIdParamSchema, CreateUserInput type
```

Closely coupled external module (consumer of `UserService`, included for data-flow context, not analyzed in depth):

```
src/modules/auth/
├── auth.service.ts         # AuthService — delegates registration to UserService.createUser; login uses UserService.toPublic statically
├── auth.controller.ts      # AuthController — register, login, me (me calls UserService.getById)
├── auth.routes.ts          # POST /register, POST /login, GET /me
└── auth.schemas.ts         # registerSchema (mirrors createUserSchema), loginSchema
```

---

## 5. Dependency Analysis

```
Internal Dependencies:
UserController.getById → UserService.getById → UserRepository.findById → PrismaClient
AuthController.register → AuthService.register → UserService.createUser → UserRepository.{findByEmail, create} → PrismaClient
AuthController.me → UserService.getById → UserRepository.findById → PrismaClient
AuthService.login → UserService.toPublic (static, no instance dependency at call time)
UserService.createUser / getById → UserService.toPublic (internal static helper reuse)
app.ts (composition root) → instantiates UserRepository → UserService → UserController, and wires the same UserService instance into AuthService/AuthController

External Dependencies:
- bcrypt (5.1.1) - Password hashing (used directly inside UserService.createUser); also independently used by AuthService for comparison and by tests/helpers/factories.ts for test fixture creation
- @prisma/client (5.22.0) - Type import only (User type); actual DB calls are performed by UserRepository, not UserService directly
- ../../shared/errors (internal shared module) - ConflictError, NotFoundError - used to signal business rule violations
- ./user.schemas.js (internal, type-only import) - CreateUserInput type
```

`UserService` itself has no direct dependency on Express, Zod, or the HTTP layer — it is a plain, framework-agnostic TypeScript class, which is a positive architectural characteristic (testable in isolation, no coupling to transport concerns).

---

## 6. Afferent and Efferent Coupling

Coupling assessed at the class/module level (TypeScript classes), counting direct compile-time references within the analyzed scope (`src/modules/users`, `src/modules/auth`, `src/app.ts`, `src/shared/errors`).

```
| Component        | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-------------------|------------------------|--------------------------|----------|
| UserService       | 4 (UserController, AuthService, AuthController, app.ts) | 3 (UserRepository, bcrypt, shared/errors) | High |
| UserRepository    | 2 (UserService, AuthService) | 1 (PrismaClient) | Medium |
| UserController    | 1 (app.ts / user.routes.ts) | 1 (UserService) | Low |
| AuthService       | 2 (AuthController, app.ts) | 3 (UserRepository, UserService, bcrypt/jwt) | High |
```

`UserService` has the highest afferent coupling in the users/auth boundary (four distinct consumers depend on it directly), reflecting its role as a shared kernel between two bounded contexts (`users` and `auth`). Its efferent coupling is low (three dependencies), and none of those dependencies are themselves complex — this results in a favorable instability/stability profile for a "core" component: high afferent, low efferent coupling indicates `UserService` is a stable abstraction that many parts of the system rely on, which also means changes to its public method signatures (`createUser`, `getById`, `toPublic`) have a wide blast radius across both modules analyzed.

---

## 7. Endpoints

`UserService` itself exposes no HTTP endpoints directly; it is invoked by two controllers across two route files. Endpoints that transitively invoke `UserService` methods are listed below for data-flow completeness.

| Endpoint | Method | Description | Invokes | Auth Required |
|----------|--------|--------------|---------|----------------|
| /api/v1/users/:id | GET | Retrieve a single user's public profile by ID | UserService.getById | Yes (authenticate + requireRole('ADMIN')) |
| /api/v1/auth/register | POST | Register a new user account (public sign-up) | UserService.createUser (via AuthService.register) | No |
| /api/v1/auth/me | GET | Retrieve the currently authenticated user's public profile | UserService.getById | Yes (authenticate) |
| /api/v1/auth/login | POST | Authenticate and issue a JWT (uses UserService.toPublic only, not createUser/getById) | UserService.toPublic (static) | No |

Note: There is no `POST /api/v1/users` endpoint. `UserService.createUser` is reachable in the current routing configuration exclusively through `/api/v1/auth/register`; the `users` module itself only wires the `GET /:id` route (src/modules/users/user.routes.ts:12-18).

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|--------------|-----------------|
| UserRepository / PrismaClient | Internal data access layer | Persist and retrieve user records from MySQL | Prisma Client API (in-process) / MySQL wire protocol underneath | Prisma-typed objects (`User`) | ConflictError raised at application layer pre-check; Prisma `P2002`/`P2025` errors caught centrally by errorMiddleware, not by UserService |
| bcrypt | External library (native/npm) | One-way password hashing at creation time | In-process function call (async) | String (bcrypt hash format) | No explicit error handling in UserService; a bcrypt failure would propagate as an unhandled rejection to the Express error-handling chain via controller `try/catch` + `next(err)` |
| AuthService | Internal service (same process) | Reuses UserService for registration and public-profile mapping during login | Direct method call (constructor-injected + static call) | PublicUser / User (Prisma type) | Errors from UserService (ConflictError, NotFoundError) propagate unchanged through AuthService to controllers |
| shared/errors (AppError hierarchy) | Internal shared module | Standardized business-error signaling | In-process class instantiation/throw | AppError subclasses (ConflictError, NotFoundError) | Centrally converted to JSON HTTP responses by errorMiddleware |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|------------------|-----------|----------|
| Repository Pattern | UserRepository abstracts Prisma/MySQL access behind findById/findByEmail/create | src/modules/users/user.repository.ts | Decouples UserService from persistence technology; enables substitution/mocking |
| Dependency Injection (constructor injection) | UserService receives UserRepository via constructor; UserController receives UserService via constructor; composition root wires everything in app.ts | src/modules/users/user.service.ts:19; src/app.ts:26-32 | Testability, inversion of control, no service-locator/global singleton |
| Data Transfer Object / View Model Mapping | PublicUser type + toPublic() static mapper | src/modules/users/user.service.ts:7-14, 42-51 | Prevents leakage of sensitive fields (passwordHash) across the service boundary |
| Fail-Fast Validation | Uniqueness check performed before expensive bcrypt hashing and before DB write | src/modules/users/user.service.ts:22-25 | Avoids unnecessary CPU cost and DB round-trips for known-invalid requests |
| Domain Exception Pattern | ConflictError, NotFoundError extend a common AppError base carrying statusCode/errorCode | src/shared/errors/app-error.ts, http-errors.ts | Uniform error shape/handling centralized in errorMiddleware, decoupling UserService from HTTP concerns |
| Static Utility Method on Domain Class | UserService.toPublic invoked both internally and externally (from AuthService) as `UserService.toPublic(...)` rather than via an instance | src/modules/users/user.service.ts:42; src/modules/auth/auth.service.ts:42 | Reuse of mapping logic across modules, though implemented via static coupling rather than a dedicated mapper/instance method |
| Shared Kernel (DDD) | UserService instance is constructed once in the composition root and injected into both the `users` and `auth` module's dependents | src/app.ts:28, 31-32 | Avoids duplicating user-creation/lookup logic between the two modules |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|------------------|--------|---------|
| High | UserService.createUser / auth.routes.ts | Public `POST /api/v1/auth/register` endpoint accepts an optional `role` field (`ADMIN` or `OPERATOR`) with no authentication or authorization gate in front of it, and `UserService.createUser` performs no role-based restriction | An unauthenticated caller can potentially self-register with `role: 'ADMIN'`, resulting in privilege escalation, unless this is mitigated elsewhere outside the analyzed component boundary (not found in the files reviewed) |
| Medium | UserService.createUser | Email-uniqueness check (`findByEmail`) and the subsequent `create` call are not wrapped in a database transaction or advisory lock; a race condition between two concurrent requests with the same email can both pass the pre-check | Under concurrent load, the DB-level unique constraint (P2002) is the actual safety net, but it produces a *different* error code/message (`CONFLICT`/generic) than the application-level check (`EMAIL_ALREADY_USED`), creating an inconsistent client-facing error contract for the same logical failure |
| Medium | UserService | `BCRYPT_ROUNDS = 10` is a hard-coded module constant, not sourced from environment/config (unlike `env.JWT_SECRET`/`env.JWT_EXPIRES_IN` used elsewhere in the codebase) | Cannot be tuned per environment (e.g., reduced for test speed, increased for production hardening) without a code change; also cannot be rotated/upgraded without a data migration strategy for existing hashes |
| Low-Medium | UserService.toPublic | Static method invoked externally via `UserService.toPublic(user)` from `AuthService` rather than through the injected instance (`this.userService`) already available | Slightly unusual coupling: static call bypasses the instance-based dependency graph, which can obscure the true dependency relationship and complicate future refactors (e.g., if toPublic needed to become instance-based or configuration-dependent) |
| Low | UserService | No `update`, `delete`, `list`, or `changePassword`/`changeRole` operations exist on UserService | Not necessarily a defect (may be intentional scope), but represents an incomplete CRUD surface for a "user management" service if broader admin capabilities are expected |
| Low | UserRepository | `findById`/`findByEmail` return the full `User` row including `passwordHash`, with no Prisma `select` projection to exclude it at the query level | All responsibility for preventing hash leakage rests on disciplined use of `toPublic` by every caller; a future new caller of `UserRepository` that forgets to call `toPublic` would leak password hashes with no safety net at the data layer |
| Low | UserService | No test file exists for `UserService` in isolation (see Test Coverage Analysis) | Business rules are only verified indirectly through HTTP integration tests of the `auth` module; the `GET /api/v1/users/:id` path (UserController → UserService.getById) has zero test coverage of any kind found in the repository |

---

## 11. Test Coverage Analysis

No dedicated unit test file for `UserService` (e.g., `user.service.test.ts` or similar) was found anywhere in the repository. Test evidence is limited to integration-level (HTTP, black-box) tests that exercise `UserService` indirectly through the `auth` module's routes.

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|-------------|----------------------|-----------|----------------|
| UserService.createUser | 0 | 3 (indirect, via `tests/auth.test.ts:6-39`: successful register, duplicate-email rejection, invalid-payload rejection) | Partial (only reachable via `/auth/register`, not via a direct unit test of the class) | Good assertions on observable HTTP behavior (status codes, error codes, absence of `passwordHash`/`password` in response body); no assertion on bcrypt hash correctness, cost factor, or repository interaction in isolation |
| UserService.getById | 0 | 1 (indirect, via `tests/auth.test.ts:65-77`: `GET /auth/me` returns authenticated user) | Partial (only the `/auth/me` path is tested; `GET /api/v1/users/:id` — the users module's own route — has no test coverage found) | Adequate for the happy path tested; the `NotFoundError` (404) branch of `getById` is not exercised by any test found in the repository; role-gated access (`requireRole('ADMIN')`) on `/users/:id` is also untested |
| UserService.toPublic | 0 (no direct unit test) | Indirectly validated via response-shape assertions in `tests/auth.test.ts:14-16` (`not.toHaveProperty('passwordHash')`, `not.toHaveProperty('password')`) | Indirect only | The negative assertions (fields that must NOT be present) are a good security-oriented test pattern, but there is no positive assertion enumerating exactly which fields ARE expected via `toMatchObject`-style whitelisting beyond `email`, `name`, `role` |
| UserRepository (dependency of UserService) | 0 | Exercised transitively by the same `tests/auth.test.ts` suite and by `tests/helpers/factories.ts` (which uses `prisma.user.create` directly, bypassing UserRepository/UserService entirely for test fixture setup) | Indirect only | Test fixtures (`createTestUser`, `bootstrapAuthenticatedUser` in tests/helpers/factories.ts:17-31, 76-83) create users by calling Prisma directly rather than through `UserService`/`UserRepository`, meaning the actual repository/service code path is not what generates most test fixtures — only the `register` test explicitly exercises `UserService.createUser` |

**Test infrastructure observed**:
- Test runner: Vitest 2.1.4 (`"test": "vitest run"` in package.json:17)
- HTTP assertion library: Supertest 7.0.0
- Test files located at: `tests/auth.test.ts`, `tests/orders.test.ts`, `tests/setup.ts`, `tests/helpers/factories.ts`
- Tests run against a real Prisma-backed database connection (`prisma` client imported directly in `tests/helpers/factories.ts:3` from `src/config/database.js`) rather than mocks/stubs — these are true integration tests, not isolated unit tests
- No mocking framework usage was found for `UserRepository` or `bcrypt` in the analyzed test files, confirming that `UserService`'s internal logic (uniqueness short-circuit ordering, bcrypt invocation, `toPublic` field mapping) has never been verified in isolation from the database and HTTP stack

**Coverage gaps identified**:
- `GET /api/v1/users/:id` (UserController → UserService.getById) — no test found at all, including no coverage of the `ADMIN`-only role restriction or the UUID param validation.
- `NotFoundError` branch of `UserService.getById` (id not found) — not exercised by any test found.
- No isolated/mocked unit test verifies that `UserService.createUser` calls `findByEmail` before `bcrypt.hash` (the fail-fast ordering business rule) — this ordering is only implicitly correct by code inspection, not proven by a test asserting call order.
- No test verifies the exact bcrypt cost factor (10) used, or that it is applied consistently.
