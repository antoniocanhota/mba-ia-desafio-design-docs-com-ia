# Component Deep Analysis Report: UserController

**Project:** order-management-api
**Component analyzed:** `UserController`
**Primary file:** `src/modules/users/user.controller.ts`
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`UserController` is a thin HTTP adapter class in the `users` module of the Order Management System REST API. It is the entry point for user-resource read operations exposed under `/api/v1/users`. In its current implementation it exposes exactly one action, `getById`, which retrieves a single user's public profile by UUID.

The component follows a strict **Controller -> Service -> Repository** layered architecture. `UserController` itself contains no business logic, no validation, and no data access code — it delegates entirely to `UserService`, which is injected via constructor (manual dependency injection, no DI framework). Its sole responsibilities are: (1) extracting the validated `id` route parameter, (2) invoking the service, (3) shaping the HTTP response (status code and JSON body), and (4) forwarding any error to the centralized Express error-handling middleware via `next(err)`.

Key findings:
- The controller is extremely small (15 lines) and has very low complexity and high cohesion — it does exactly one thing.
- Authorization (`ADMIN`-only access) and input validation (UUID format) are enforced entirely outside the controller, in the router (`user.routes.ts`) via middleware composition (`authenticate`, `requireRole('ADMIN')`, `validate`).
- All business rules relevant to "user retrieval" (existence check, public-field projection, password-hash exclusion) live in `UserService`, not in the controller itself. The controller is a pure pass-through/adapter.
- `UserController` is also reused indirectly: `AuthController.me` depends on `UserService` (not `UserController`) directly to implement `GET /auth/me`, so the controller class itself is only wired into the `/users` route tree.
- No automated test exercises the `GET /api/v1/users/:id` endpoint directly; the underlying `UserService.getById` logic is only covered indirectly through the `GET /auth/me` test flow. This is documented as a coverage gap in Section 11.

---

## 2. Data Flow Analysis

```
1. HTTP GET /api/v1/users/:id request enters Express app (src/app.ts)
2. Router mounts /users prefix -> buildUserRouter (src/modules/users/user.routes.ts:9-21)
3. Middleware chain executes in order:
   a. authenticate (src/middlewares/auth.middleware.ts:27) - verifies JWT, populates req.user
   b. requireRole('ADMIN') (src/middlewares/auth.middleware.ts:49) - restricts to ADMIN role
   c. validate({ params: idParamSchema }) (src/middlewares/validate.middleware.ts:11) - validates
      req.params.id is a UUID via Zod; on failure raises ValidationError (400)
4. Request reaches UserController.getById (src/modules/users/user.controller.ts:7-14)
5. Controller calls this.users.getById(req.params.id!) -> UserService.getById
   (src/modules/users/user.service.ts:36-40)
6. UserService.getById calls UserRepository.findById(id) -> Prisma Client
   (src/modules/users/user.repository.ts:6-8) -> PostgreSQL/MySQL "users" table
7. If no record found, UserService throws NotFoundError('User') (404)
8. If found, UserService.toPublic(user) strips passwordHash and returns a PublicUser DTO
   (src/modules/users/user.service.ts:42-51)
9. Control returns to UserController, which sets res.status(200).json(user)
   (src/modules/users/user.controller.ts:10)
10. On any thrown error at steps 3, 6, 7 (or unexpected exceptions), UserController.getById
    catches it and calls next(err) (src/modules/users/user.controller.ts:11-13)
11. errorMiddleware (src/middlewares/error.middleware.ts:14-65) inspects the error type
    (AppError, ZodError, Prisma known error, or unknown) and formats the final JSON error
    response with an appropriate HTTP status code
```

Notes:
- The controller performs no data transformation itself; all shaping (DTO projection, field
  whitelisting) happens in `UserService.toPublic`.
- The controller never touches Prisma or the database directly — it is fully decoupled from
  the persistence layer, communicating only through the `UserService` port.

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|------------------|----------|
| Authorization | Only users with role `ADMIN` may call `GET /users/:id` | src/modules/users/user.routes.ts:15 |
| Authentication | A valid, non-expired Bearer JWT is required to access the endpoint | src/middlewares/auth.middleware.ts:27-47 |
| Validation | The `id` route parameter must be a syntactically valid UUID | src/modules/users/user.routes.ts:7, src/modules/users/user.schemas.ts:3-5 |
| Business Logic | User must exist in the database, otherwise a 404 `NOT_FOUND` error is returned | src/modules/users/user.service.ts:38 |
| Data Exposure / Security | Response body must exclude sensitive fields (`passwordHash`); only public fields are returned | src/modules/users/user.service.ts:42-51 |
| Response Contract | Successful lookup responds with HTTP 200 and JSON body containing the public user representation | src/modules/users/user.controller.ts:10 |
| Error Propagation | Any error (validation, auth, not-found, unexpected) is forwarded to centralized error middleware rather than handled locally | src/modules/users/user.controller.ts:11-13 |

### Detailed breakdown of the business rules

---

### Business Rule: ADMIN-Only Authorization

**Overview**:
Access to `GET /api/v1/users/:id` is restricted to authenticated principals whose role is exactly `ADMIN`. This rule is enforced not inside `UserController` but in the routing layer, one level above the controller, through the `requireRole('ADMIN')` middleware factory.

**Detailed description**:
The `buildUserRouter` function (`src/modules/users/user.routes.ts:12-18`) composes a middleware chain for the `GET /:id` route: `authenticate`, then `requireRole('ADMIN')`, then `validate(...)`, and finally `controller.getById`. The `requireRole` middleware (`src/middlewares/auth.middleware.ts:49-61`) is a higher-order function that accepts a variadic list of allowed roles and returns an Express `RequestHandler`. It checks `req.user` (populated by the preceding `authenticate` middleware) and, if the user's role is not included in the allowed list, calls `next(new ForbiddenError('Insufficient permissions'))`, which the error middleware turns into an HTTP 403 response with error code `FORBIDDEN`.

This means the `OPERATOR` role — the default role assigned at registration (see `user.schemas.ts:11`, `role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')`) — is explicitly denied access to look up arbitrary users by ID. Only accounts provisioned with the `ADMIN` role can query other users' profile data through this endpoint. Because the check happens entirely in middleware before the controller is invoked, `UserController.getById` itself contains no authorization logic and cannot be reached by a non-ADMIN caller under normal routing; the rule is architecturally centralized and reusable across other admin-only routes in the system.

If `req.user` is missing entirely (which should not normally happen after `authenticate` succeeds, but is defensively checked), `requireRole` raises `UnauthorizedError` (401) instead of `ForbiddenError`, distinguishing "not authenticated" from "authenticated but not permitted."

**Rule workflow**:
```
Request -> authenticate (verifies JWT, sets req.user)
        -> requireRole('ADMIN')
             if req.user is undefined -> 401 UNAUTHORIZED
             if req.user.role !== 'ADMIN' -> 403 FORBIDDEN
             else -> next()
        -> validate(params) -> controller.getById
```

---

### Business Rule: Authentication Requirement (Valid JWT)

**Overview**:
Every request to the `/users` route tree must carry a valid, non-expired JSON Web Token in the `Authorization: Bearer <token>` header.

**Detailed description**:
The `authenticate` middleware (`src/middlewares/auth.middleware.ts:27-47`) runs first in the chain for `GET /:id`. It inspects the `Authorization` header; if absent or not prefixed with `bearer ` (case-insensitive), it raises `UnauthorizedError('Missing or invalid Authorization header')`. If the header is present but the token substring is empty after trimming, it raises `UnauthorizedError('Missing bearer token')`. Otherwise it attempts `jwt.verify(token, env.JWT_SECRET)`, decoding the payload into `{ sub, email, role, iat, exp }`. On success, it attaches a normalized `AuthUser` object (`{ id, email, role }`) to `req.user` for downstream middleware and handlers to consume. On any verification failure (invalid signature, malformed token, or expiration), it raises a generic `UnauthorizedError('Invalid or expired token')`, deliberately not leaking which specific validation failed, to avoid giving attackers diagnostic information.

This rule guarantees that `UserController.getById` — and the `requireRole` check that precedes it — can always safely assume `req.user` is populated with a trustworthy, server-verified identity by the time (if ever) the controller code runs. The controller itself performs no authentication logic; it relies fully on this upstream guarantee.

**Rule workflow**:
```
Header missing/malformed -> 401 UNAUTHORIZED ("Missing or invalid Authorization header")
Bearer token empty       -> 401 UNAUTHORIZED ("Missing bearer token")
jwt.verify fails         -> 401 UNAUTHORIZED ("Invalid or expired token")
jwt.verify succeeds      -> req.user = { id: sub, email, role } ; next()
```

---

### Business Rule: Path Parameter Must Be a Valid UUID

**Overview**:
The `id` route parameter for `GET /users/:id` must conform to the UUID format before it reaches the controller or the database layer.

**Detailed description**:
Two schema definitions exist for this purpose: `idParamSchema` declared locally inside `user.routes.ts:7` (`z.object({ id: z.string().uuid() })`), and an equivalent, separately exported `userIdParamSchema` in `user.schemas.ts:3-5`. Both use Zod's `.uuid()` validator. The route wiring (`user.routes.ts:16`) uses the locally declared `idParamSchema` (not the exported `userIdParamSchema`), which is a minor duplication — two schemas encode the identical constraint, and only one is actually used at the wiring point. The `validate` middleware (`src/middlewares/validate.middleware.ts:11-37`) parses `req.params` against this schema; on a Zod validation failure it maps each Zod issue into a `{ path, message }` structure and throws a `ValidationError` (HTTP 400, error code `VALIDATION_ERROR`) carrying those `details`.

Because this validation occurs before `UserController.getById` executes, the controller can safely assume `req.params.id` is a syntactically well-formed UUID string when it calls `this.users.getById(req.params.id!)`. The non-null assertion (`!`) on `req.params.id` reflects this assumption: TypeScript cannot statically know the middleware guarantees the field's presence, so the controller author asserts it manually rather than re-checking at runtime. This is a form of implicit contract between the controller and its middleware pipeline — if the router were ever misconfigured to omit the `validate` middleware, the non-null assertion would become unsound and could pass `undefined` through to the service/repository layer.

Note that UUID *syntax* validity does not guarantee the referenced user actually exists — that is a separate rule (see "User Existence Check" below) enforced at the service layer.

**Rule workflow**:
```
req.params.id parsed against z.object({ id: z.string().uuid() })
  invalid format -> 400 VALIDATION_ERROR with details [{ path: "id", message: "..." }]
  valid format   -> req.params.id replaced with parsed value ; next()
```

---

### Business Rule: User Existence Check (404 on Missing Record)

**Overview**:
If no user record exists for the given ID, the API must respond with HTTP 404 and a `NOT_FOUND` error code rather than a null or empty success response.

**Detailed description**:
This rule is implemented entirely in `UserService.getById` (`src/modules/users/user.service.ts:36-40`), not in `UserController`. The service calls `this.users.findById(id)` (delegating to `UserRepository.findById`, which wraps `prisma.user.findUnique({ where: { id } })`). Prisma's `findUnique` returns `null` when no row matches. The service explicitly checks `if (!user) throw new NotFoundError('User')`. `NotFoundError` (`src/shared/errors/http-errors.ts:27-31`) is a subclass of `AppError` that hardcodes HTTP status 404 and error code `NOT_FOUND`, with a message template `${resource} not found` — here rendering as `"User not found"`.

`UserController.getById` does not special-case this outcome; its `try/catch` simply forwards whatever the service throws to `next(err)`, and the centralized `errorMiddleware` (`src/middlewares/error.middleware.ts:14-24`) recognizes `AppError` instances and serializes them using their own `statusCode`, `errorCode`, and optional `details`. This demonstrates the layered error-handling pattern used throughout the codebase: domain/service code decides *what* went wrong and encodes it as a typed error; the controller is agnostic to error semantics and only decides *whether* to catch-and-forward (which it always does here); and the middleware decides *how* to render it over HTTP.

**Rule workflow**:
```
UserRepository.findById(id) -> Prisma findUnique
  result === null -> UserService throws NotFoundError('User')
                   -> UserController catches, calls next(err)
                   -> errorMiddleware responds 404 { error: { code: "NOT_FOUND", message: "User not found" } }
  result !== null -> UserService.toPublic(result) returned to controller -> 200 response
```

---

### Business Rule: Public Projection / Sensitive Field Exclusion

**Overview**:
The API must never expose the `passwordHash` field (or any other internal-only field) in a user representation returned to clients.

**Detailed description**:
`UserService.toPublic` (`src/modules/users/user.service.ts:42-51`) is a static method that receives the full Prisma `User` entity (which includes `passwordHash`) and constructs a new `PublicUser` object containing only `id`, `email`, `name`, `role`, `createdAt`, and `updatedAt`. This is a manual allow-list projection rather than a deny-list/omission — a design choice that is inherently safer against accidental leakage of newly added sensitive fields, since any new column added to the Prisma `User` model in the future would not automatically appear in API responses unless explicitly added to `toPublic`.

`UserController.getById` receives only the already-sanitized `PublicUser` object from `UserService.getById` — it has no access to the raw Prisma entity or the password hash at any point, so there is no risk of the controller itself accidentally serializing sensitive data. This separation of concerns (raw persistence model vs. public DTO) is enforced structurally by the `PublicUser` TypeScript type (`user.service.ts:7-14`), which is a distinct type from the Prisma-generated `User` type and does not include `passwordHash`.

**Rule workflow**:
```
Prisma User { id, email, passwordHash, name, role, createdAt, updatedAt }
  -> UserService.toPublic(user)
  -> PublicUser { id, email, name, role, createdAt, updatedAt }  (passwordHash dropped)
  -> returned to controller -> serialized as JSON response body
```

---

### Business Rule: Error Delegation / No Local Error Handling in Controller

**Overview**:
`UserController.getById` never formats or interprets errors itself; it exclusively delegates error handling to Express's centralized error-handling middleware via `next(err)`.

**Detailed description**:
The controller wraps its single asynchronous operation in a `try/catch` block (`user.controller.ts:8-13`). On success, it manually sets the response status and JSON body (`res.status(200).json(user)`); on any thrown exception — whether a typed `AppError` subclass like `NotFoundError`, an unexpected runtime error, or (in theory, though not reachable given upstream validation) a raw Prisma error — it calls `next(err)` unconditionally, with no conditional branching or inspection of the error type. This is a consistent pattern across all controllers in the codebase (also seen identically in `AuthController`), reflecting an architectural convention: **controllers are dumb pass-throughs for both success and failure paths**, and all interpretation/formatting logic is centralized in `errorMiddleware` (`src/middlewares/error.middleware.ts`).

This has both a design benefit and a consistency risk: the benefit is that error-response formatting is guaranteed uniform across every endpoint in the API, since there is exactly one place (`errorMiddleware`) that decides how errors become JSON. The risk is that this convention is not structurally enforced — a future controller author could add a local `res.status(...).json(...)` inside a `catch` block, silently diverging from this pattern, since nothing in the type system or framework prevents it. Since `getById` is an `async` `RequestHandler`, any promise rejection *not* explicitly caught in the `try/catch` (which cannot happen here, since the entire body is wrapped) would otherwise become an unhandled rejection; Express 4.x (used here, per `package.json`, `express: 4.21.1`) does not automatically catch async errors, so the manual `try/catch` + `next(err)` pattern is a required, not optional, safeguard for this Express major version.

**Rule workflow**:
```
try {
  user = await this.users.getById(id)
  res.status(200).json(user)
} catch (err) {
  next(err)   // always, regardless of error type/origin
}
```

---

## 4. Component Structure

```
src/modules/users/
├── user.controller.ts   # HTTP adapter: single action getById (analyzed component)
├── user.routes.ts       # Express Router wiring: auth, role-guard, validation, controller binding
├── user.schemas.ts      # Zod schemas: createUserSchema, userIdParamSchema (input contracts)
├── user.service.ts      # Business logic: createUser, getById, toPublic (DTO projection)
└── user.repository.ts   # Data access: findById, findByEmail, create (Prisma wrapper)
```

Related files outside the module boundary that directly participate in `UserController`'s
request lifecycle:

```
src/
├── app.ts                              # Composition root: instantiates UserController with UserService
├── routes/index.ts                     # Mounts buildUserRouter at /api/v1/users
├── middlewares/
│   ├── auth.middleware.ts              # authenticate, requireRole (used by user.routes.ts)
│   ├── validate.middleware.ts          # validate() Zod-based param/body/query validator
│   └── error.middleware.ts             # Centralized error formatter (consumes next(err))
└── shared/
    ├── errors/app-error.ts             # AppError base class
    └── errors/http-errors.ts           # NotFoundError, ValidationError, ForbiddenError, etc.
```

`UserController` itself consists of a single class with one constructor parameter and one
public field (`getById`), totaling 15 lines of code — no private helper methods, no local
constants, no additional imports beyond typing.

---

## 5. Dependency Analysis

```
Internal Dependencies:
UserController → UserService (constructor injection, src/modules/users/user.controller.ts:5)
UserService → UserRepository (constructor injection, src/modules/users/user.service.ts:19)
UserService → shared/errors (ConflictError, NotFoundError)
UserRepository → PrismaClient (constructor injection)

Consumers of UserController:
routes/index.ts → buildUserRouter(controllers.users) → UserController.getById bound as route handler
app.ts (buildControllers) → instantiates UserController(userService) and wires into Controllers map

Indirect / sibling dependency (does NOT depend on UserController directly):
AuthController → UserService (direct dependency, bypasses UserController entirely, for GET /auth/me)

External Dependencies:
- express (4.21.1) - RequestHandler typing, routing, request/response objects
- (transitively, via UserService) bcrypt (5.1.1) - password hashing, not used by controller directly
- (transitively, via UserRepository) @prisma/client (5.22.0) - ORM/database client, not used by controller directly
```

`UserController` has no direct external dependencies of its own beyond the `express` type
import (`RequestHandler`). It has zero direct dependency on Prisma, bcrypt, or Zod — those are
consumed by lower layers (`UserService`, `UserRepository`, `user.routes.ts`/`validate.middleware.ts`
respectively).

---

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|-------------------|-------------------|-------------------|
| UserController | 2 (routes/index.ts, tests/helpers/factories.ts indirectly via app) | 1 (UserService) | Low |
| UserService | 3 (UserController, AuthController, AuthService) | 3 (UserRepository, ConflictError, NotFoundError) | Medium |
| UserRepository | 1 (UserService) | 1 (PrismaClient) | Low |
| user.routes.ts (module) | 1 (routes/index.ts) | 3 (auth.middleware, validate.middleware, UserController) | Low |

Notes on interpretation: "Afferent coupling" (Ca) counts classes/modules that depend on the
given component; "Efferent coupling" (Ce) counts classes/modules the given component depends
on. `UserController` has low afferent coupling (only the route builder references it) and
minimal efferent coupling (a single collaborator, `UserService`), which is consistent with its
role as a thin, low-risk adapter. `UserService` has the highest afferent coupling in the module
because it is reused directly by both `UserController` and `AuthController`/`AuthService`,
making it the more architecturally significant (and higher-risk-to-change) component of the two.

---

## 7. Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| /api/v1/users/:id | GET | Retrieve a single user's public profile by UUID. Requires a valid Bearer JWT and `ADMIN` role. Returns 200 with `PublicUser` JSON on success, 400 on invalid UUID param, 401 if unauthenticated, 403 if the caller is not `ADMIN`, 404 if no user exists with the given ID. |

Note: `user.schemas.ts` defines a `createUserSchema` (email/password/name/role) that is not
consumed anywhere within the `users` module's own router (`user.routes.ts` only builds the
`GET /:id` route). User creation is instead handled by the `auth` module's `POST /auth/register`
endpoint (`AuthController.register` → `AuthService.register`, which internally likely calls
`UserService.createUser`). This means `UserController` does not currently expose a `POST /users`
endpoint, even though `UserService.createUser` and `createUserSchema` exist and appear designed
for that purpose — see Section 10 (Technical Debt) for further discussion of this apparent gap.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|-------------|----------------|
| UserService | Internal Service (same process) | Business logic delegation for user retrieval | Direct method call (in-process) | TypeScript objects / Promises | Exceptions propagated via try/catch + next(err) |
| Express Router/Middleware pipeline | Internal Framework | Request routing, auth, validation before controller invocation | In-process function chaining (RequestHandler) | Express Request/Response objects | Errors passed via next(err) to errorMiddleware |
| PostgreSQL/MySQL database (via Prisma, indirect) | External Datastore | Persistent storage of user records | SQL (via Prisma Client, TCP) | Relational rows mapped to TS objects | Not handled by controller; Prisma errors bubble through UserService/UserRepository and are caught centrally by errorMiddleware (P2002, P2025 special-cased) |

`UserController` itself has no direct integration with the database or any external API; all
integrations are mediated through `UserService`.

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|----------------|----------|---------|
| Layered Architecture (Controller-Service-Repository) | UserController → UserService → UserRepository | src/modules/users/*.ts | Separation of HTTP concerns, business logic, and data access |
| Dependency Injection (manual/constructor) | `constructor(private readonly users: UserService)` | src/modules/users/user.controller.ts:5 | Testability and decoupling from concrete service instantiation |
| Composition Root | `buildControllers(prisma)` | src/app.ts:26-51 | Centralized manual wiring of all dependencies at application startup |
| DTO / Data Projection | `PublicUser` type + `UserService.toPublic` | src/modules/users/user.service.ts:7-14, 42-51 | Prevents leaking sensitive fields (passwordHash) across the HTTP boundary |
| Chain of Responsibility (Express middleware) | authenticate → requireRole → validate → controller | src/modules/users/user.routes.ts:12-18 | Layered cross-cutting concerns (authn, authz, validation) applied before business logic |
| Centralized Error Handling | errorMiddleware + AppError hierarchy | src/middlewares/error.middleware.ts, src/shared/errors/*.ts | Uniform error response shape across all endpoints; controller stays free of formatting logic |
| Typed Domain Errors | AppError subclasses (NotFoundError, ValidationError, etc.) | src/shared/errors/http-errors.ts | Encodes HTTP semantics (status code, error code) into the exception type itself |

`UserController` follows the "thin controller" pattern consistently applied across all modules
in this codebase (`AuthController`, `CustomerController`, `ProductController`, `OrderController`
all share the identical `try { ... } catch (err) { next(err) }` shape based on the evidence
gathered).

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|----------------|-------|--------|
| Low | user.controller.ts:9 | Non-null assertion `req.params.id!` relies entirely on upstream middleware (`validate`) always being present in the route chain; no defensive runtime check inside the controller itself | If the route is ever refactored and the `validate` middleware is dropped or reordered, `id` could be `undefined` and reach `UserService.getById(undefined as unknown as string)`, likely producing a confusing Prisma error rather than a clean 400 |
| Low | user.routes.ts:7 vs user.schemas.ts:3-5 | Two functionally identical UUID param schemas exist (`idParamSchema` defined locally in the routes file, and `userIdParamSchema` exported from `user.schemas.ts`), and only the local one is actually used | Duplicated validation logic; risk of the two schemas drifting apart if one is edited without the other being noticed |
| Medium | Module scope / API completeness | `user.schemas.ts` defines `createUserSchema` and `UserService.createUser` exists, but `UserController`/`user.routes.ts` expose no `POST /users` endpoint — user creation is only reachable through `POST /auth/register` | Ambiguous ownership of user-creation business rules between the `auth` and `users` modules; unclear whether `UserService.createUser` is dead code reachable only via `AuthService`, or an intentionally incomplete `users` module API surface |
| Medium | Test coverage | No test file directly targets `GET /api/v1/users/:id` (no `users.test.ts` exists); `UserController.getById` is only exercised indirectly via the unrelated `AuthController.me` → `UserService.getById` path in `tests/auth.test.ts` | The controller's own routing (auth/role/validation middleware chain, response status/shape) is unverified by automated tests; regressions in `user.routes.ts` (e.g., role check misconfiguration) would not be caught |
| Low | user.controller.ts | Single-action controller class — currently minimal, but if additional user-management endpoints (update, delete, list) are added later without corresponding architectural review, the class could grow inconsistently with the rest of the module's design intent | Maintainability risk is currently negligible given the component's small size, but is worth monitoring |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage | Test Quality |
|-----------|------------|--------------------|----------|---------------|
| UserController (getById route) | 0 | 0 (no direct test) | None (0% direct) | N/A — no dedicated test file (`tests/users.test.ts` does not exist) |
| UserService.getById (indirect coverage) | 0 | 1 (via `tests/auth.test.ts:65-77`, `GET /auth/me`) | Partial, indirect | Verifies 200 status and correct `id`/`email` fields returned; does not verify 404 path, does not verify `passwordHash` exclusion explicitly for this endpoint (though implicitly covered by the shape check) |
| UserService.createUser (indirect coverage) | 0 | 3 (via `tests/auth.test.ts`, `POST /auth/register` tests: success, duplicate email, invalid payload) | Good, indirect | Validates 201 success shape and password field exclusion (`tests/auth.test.ts:13-17`), 409 conflict on duplicate email (`tests/auth.test.ts:19-28`), 400 on invalid payload (`tests/auth.test.ts:30-39`) |
| user.routes.ts middleware chain (authenticate, requireRole, validate) | 0 | 0 (no test hits `/api/v1/users/:id` at all) | None | No test verifies that a non-ADMIN user is forbidden (403), that a missing token yields 401, or that an invalid UUID param yields 400 for this specific route |

**Test files examined:**
- `tests/auth.test.ts` — covers `/api/v1/auth/register`, `/api/v1/auth/login`, `/api/v1/auth/me` (relevant because these routes exercise `UserService`, a direct collaborator of `UserController`, but never exercise `UserController` itself)
- `tests/orders.test.ts` — not relevant to `UserController` (covers `orders` module)
- `tests/helpers/factories.ts` — provides `createTestUser`, `bootstrapAuthenticatedUser`, `loginAndGetToken` test helpers used across the suite; no factory or helper specifically targets `GET /users/:id`
- `tests/setup.ts` — test environment bootstrap, no user-controller-specific setup

**Coverage gap summary:** `UserController.getById` and its full middleware chain
(`authenticate` → `requireRole('ADMIN')` → `validate(idParamSchema)` → `getById`) have **no
direct automated test coverage**. All confidence in this endpoint's correctness currently rests
on: (a) code inspection, (b) the fact that `UserService.getById` is indirectly tested via a
different route (`/auth/me`), and (c) the generic, shared nature of the `authenticate`,
`requireRole`, and `validate` middleware (which may be indirectly exercised by other modules'
tests, such as `orders.test.ts`, but this analysis did not find evidence of a role-based 403
test scenario for the `ADMIN`-only restriction specific to this route).

---
