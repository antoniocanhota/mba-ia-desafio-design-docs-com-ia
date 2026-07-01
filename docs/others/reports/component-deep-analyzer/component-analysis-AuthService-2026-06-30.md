# Component Deep Analysis Report: AuthService

**Project**: Order Management System REST API (order-management-api)
**Component**: AuthService
**Primary File**: `src/modules/auth/auth.service.ts`
**Analysis Date**: 2026-06-30

---

## 1. Executive Summary

`AuthService` is the authentication domain service for the Order Management System REST API. It is a small, focused class (51 lines) with two public responsibilities: user registration (delegated to `UserService`) and credential-based login (password verification plus JWT issuance). It does not implement authorization/session validation itself — that responsibility lives in the separate `authenticate` middleware (`src/middlewares/auth.middleware.ts`), which independently verifies JWTs issued by `AuthService`.

Architecturally, `AuthService` sits in the `modules/auth` module alongside `AuthController`, `auth.schemas.ts` (Zod input validation), and `auth.routes.ts` (Express route wiring). It depends on `UserRepository` (direct Prisma data access for `findByEmail`) and `UserService` (for `createUser` and the static `toPublic` DTO mapper), and on two external libraries: `bcrypt` (password hashing/comparison) and `jsonwebtoken` (JWT signing).

Key findings:
- **Stateless JWT authentication** with no refresh tokens, no logout/revocation mechanism, and no token blacklist — tokens remain valid until natural expiry (`JWT_EXPIRES_IN`, default `8h`).
- **No rate limiting or account lockout** on the login endpoint anywhere in the codebase — the component and its surrounding middleware stack have no brute-force protection.
- **Uniform error messaging** for login failures (`"Invalid credentials"` for both unknown email and wrong password) is a deliberate anti-user-enumeration measure — a positive security practice.
- **Thin service, clean separation of concerns**: registration business logic (duplicate email check, password hashing) is correctly delegated to `UserService`/`UserRepository`, keeping `AuthService` focused purely on authentication (login) and token issuance.
- **Role-based access control (RBAC)** is enabled via JWT claims (`role: 'ADMIN' | 'OPERATOR'`) but is only actively enforced on one route in the entire codebase (`GET /api/v1/users/:id`, restricted to `ADMIN`). Order, customer, and product routes only require authentication, not role checks.
- Test coverage for `AuthService`'s externally observable behavior is good (6 integration tests in `tests/auth.test.ts` covering register, login, and `/me`), but there are **no isolated unit tests** for `AuthService` itself — all coverage is via HTTP integration tests through `supertest`.

---

## 2. Data Flow Analysis

### 2.1 Registration flow

```
1. Client sends POST /api/v1/auth/register (auth.routes.ts:10)
2. validate({ body: registerSchema }) middleware parses/validates payload (auth.schemas.ts:3-8)
   - email format + max 255 chars, password 8-72 chars, name 2-150 chars, role defaults to OPERATOR
   - On failure: ValidationError (400, VALIDATION_ERROR) via validate.middleware.ts:31
3. AuthController.register invoked (auth.controller.ts:12-19)
4. AuthController delegates to AuthService.register(input) (auth.service.ts:27-29)
5. AuthService.register delegates directly to UserService.createUser(input) (user.service.ts:21-34)
   a. UserRepository.findByEmail checks for existing user (user.repository.ts:10-12)
   b. If found: ConflictError('Email already registered', 'EMAIL_ALREADY_USED') thrown (409)
   c. bcrypt.hash(password, 10) computes password hash (user.service.ts:26)
   d. UserRepository.create persists new user via Prisma (user.repository.ts:14-16)
   e. UserService.toPublic(created) strips passwordHash from the returned object (user.service.ts:42-51)
6. AuthController returns 201 with PublicUser JSON (auth.controller.ts:15)
7. On any thrown error, Express next(err) routes to errorMiddleware (error.middleware.ts) which maps
   AppError subclasses to structured JSON with statusCode/errorCode
```

### 2.2 Login flow

```
1. Client sends POST /api/v1/auth/login (auth.routes.ts:11)
2. validate({ body: loginSchema }) middleware validates payload (auth.schemas.ts:10-13)
   - email format, password min 1 / max 72 chars
3. AuthController.login invoked (auth.controller.ts:21-28)
4. AuthController delegates to AuthService.login(input) (auth.service.ts:31-45)
   a. UserRepository.findByEmail(input.email) looks up the user (auth.service.ts:32)
   b. If not found: throw UnauthorizedError('Invalid credentials') (401) (auth.service.ts:33-35)
   c. bcrypt.compare(input.password, user.passwordHash) verifies the password (auth.service.ts:36)
   d. If mismatch: throw UnauthorizedError('Invalid credentials') (401) (auth.service.ts:37-39)
      -- NOTE: identical error message/code for "user not found" and "wrong password" (anti-enumeration)
   e. signToken(user.id, user.email, user.role) issues a JWT (auth.service.ts:40, 47-50)
      - Payload: { sub: userId, email, role }; signed with env.JWT_SECRET; expiresIn from env.JWT_EXPIRES_IN
   f. UserService.toPublic(user) maps the Prisma User to a PublicUser DTO (excludes passwordHash)
5. AuthController returns 200 with { user, tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }
6. Errors propagate to errorMiddleware identically to the registration flow
```

### 2.3 Authenticated request flow (downstream consumer, not AuthService itself but directly dependent on its token format)

```
1. Client sends request with "Authorization: Bearer <token>" header
2. authenticate middleware (auth.middleware.ts:27-46) extracts and validates the header format
3. jwt.verify(token, env.JWT_SECRET) validates signature and expiry, decoding { sub, email, role }
4. req.user is populated with { id, email, role } for downstream handlers
5. Optionally, requireRole(...roles) middleware (auth.middleware.ts:49-61) checks req.user.role
   against an allow-list; throws ForbiddenError (403) if the role is not permitted
6. AuthController.me (or any other authenticated controller) reads req.user for authorization context
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|-------------------|----------|
| Validation | Email must be valid format, max 255 chars (register/login) | `src/modules/auth/auth.schemas.ts:4,11` |
| Validation | Password must be 8-72 characters on registration | `src/modules/auth/auth.schemas.ts:5` |
| Validation | Password must be 1-72 characters on login (non-empty only) | `src/modules/auth/auth.schemas.ts:12` |
| Validation | Name must be 2-150 characters on registration | `src/modules/auth/auth.schemas.ts:6` |
| Business Logic | Role defaults to `OPERATOR` if not specified at registration | `src/modules/auth/auth.schemas.ts:7` |
| Business Logic | Email must be unique; duplicate registration is rejected | `src/modules/users/user.service.ts:22-25` |
| Security | Passwords are hashed with bcrypt (10 salt rounds) before storage | `src/modules/users/user.service.ts:16,26` |
| Security | Password hash is compared via bcrypt.compare, never plaintext | `src/modules/auth/auth.service.ts:36` |
| Security | Uniform "Invalid credentials" error for both unknown email and wrong password | `src/modules/auth/auth.service.ts:33-39` |
| Security | Password hash (`passwordHash`) is never exposed in API responses | `src/modules/users/user.service.ts:42-51` |
| Business Logic | JWT issued on successful login contains sub (user id), email, and role claims | `src/modules/auth/auth.service.ts:49` |
| Business Logic | JWT expiry is centrally configured via `JWT_EXPIRES_IN` env var (default `8h`) | `src/config/env.ts:9`, `src/modules/auth/auth.service.ts:48` |
| Business Logic | Response always reports `tokenType: 'Bearer'` | `src/modules/auth/auth.service.ts:43` |
| Authorization | Only two roles exist: `ADMIN` and `OPERATOR` | `src/modules/auth/auth.schemas.ts:7`, `prisma/schema.prisma:11-14` |
| Authorization | `authenticate` middleware requires a well-formed `Bearer <token>` header | `src/middlewares/auth.middleware.ts:27-38` |
| Authorization | `requireRole` middleware restricts endpoints to an explicit role allow-list | `src/middlewares/auth.middleware.ts:49-61` |
| Configuration | `JWT_SECRET` must be at least 16 characters; app fails to start otherwise | `src/config/env.ts:8,14-24` |

## Detailed breakdown of the business rules

---

### Business Rule: Uniform "Invalid Credentials" Error Response

**Overview**:
When a login attempt fails, whether because the supplied email does not correspond to any user or because the supplied password does not match the stored hash, `AuthService.login` throws the exact same `UnauthorizedError('Invalid credentials')` (auth.service.ts:34 and auth.service.ts:38). Both paths result in identical HTTP status (401), identical error code (`UNAUTHORIZED`), and identical message text.

**Detailed description**:
This is a deliberate security control against user enumeration attacks. If the API distinguished between "email not found" and "wrong password" (e.g., via different messages or status codes), an attacker could probe the `/api/v1/auth/login` endpoint with a list of candidate email addresses and use the differing responses to build a list of valid, registered accounts — a precursor step to credential-stuffing or targeted phishing attacks. By collapsing both failure modes into one indistinguishable outcome, `AuthService` denies the attacker this oracle.

The rule is implemented directly and only within `AuthService.login`: the lookup failure branch (auth.service.ts:32-35) and the password-mismatch branch (auth.service.ts:36-39) are structurally parallel, each independently throwing `new UnauthorizedError('Invalid credentials')`. There is no shared helper function extracting this into a single call site, meaning the invariant is maintained by convention/duplication rather than being structurally guaranteed — a future edit to one branch's message could silently break the anti-enumeration property without the type system or tests catching it unless the existing integration test (`tests/auth.test.ts:54-63`, "rejects login with wrong password") is extended to also assert on the unknown-email case with the same expected message. Currently, `tests/auth.test.ts` only tests the wrong-password branch explicitly; the unknown-email branch is not covered by an assertion in the visible test suite.

Operationally, this rule affects every login attempt in the system and has no configuration or override — it is unconditional. It does not, however, address timing-attack risks: `bcrypt.compare` is only invoked when a user record is found (auth.service.ts:36), meaning a login attempt against a non-existent email returns faster (no bcrypt computation) than one against a valid email with a wrong password. This timing differential is a residual, lower-severity enumeration vector not mitigated by the uniform error message alone.

**Rule workflow**:
```
findByEmail(email) → not found → throw UnauthorizedError('Invalid credentials') [fast path, no bcrypt]
findByEmail(email) → found → bcrypt.compare(password, hash) → mismatch → throw UnauthorizedError('Invalid credentials') [slow path, bcrypt computed]
findByEmail(email) → found → bcrypt.compare(password, hash) → match → issue JWT, return LoginResult
```

---

### Business Rule: Password Hashing with bcrypt (10 Salt Rounds)

**Overview**:
All user passwords are hashed using `bcrypt` with a fixed cost factor of 10 salt rounds (`BCRYPT_ROUNDS = 10`, defined in `src/modules/users/user.service.ts:16`) before being persisted. `AuthService` never handles or stores plaintext passwords beyond the single request lifecycle; it only ever compares an incoming plaintext password against the stored hash via `bcrypt.compare`.

**Detailed description**:
The hashing step itself occurs in `UserService.createUser` (user.service.ts:26), not in `AuthService`, since registration is delegated wholesale by `AuthService.register` (auth.service.ts:27-29) to `UserService.createUser`. This means the actual hashing algorithm, cost factor, and salt generation are entirely outside `AuthService`'s direct code, but `AuthService` is the sole consumer of the resulting `passwordHash` field during login verification (auth.service.ts:36), making the two components tightly coupled around this shared data contract (the `User.passwordHash` field defined in `prisma/schema.prisma:28`).

The cost factor of 10 is a hardcoded module-level constant with no environment-variable override, meaning it cannot be tuned per environment (e.g., lower for local development speed, higher for production security) without a code change. This is consistent with how `tests/helpers/factories.ts:21` independently re-implements the same hashing call (`bcrypt.hash(password, 10)`) for test fixture creation — the constant is duplicated rather than imported/shared, meaning the two values could drift out of sync if one location's factor were changed without noticing the other.

From a data-flow perspective, the hash is computed once at registration and never regenerated or rotated. There is no password-change, password-reset, or forced-rotation endpoint anywhere in the `auth` or `users` modules, meaning once a user is registered, their password (and therefore their hash) is immutable for the lifetime of the account, short of direct database manipulation. This is both a scope observation (no such feature was found) and a potential business-continuity gap: a compromised account has no self-service recovery path.

**Rule workflow**:
```
Registration: plaintext password → bcrypt.hash(password, 10) → passwordHash stored in DB (never plaintext)
Login: plaintext password (from request) + stored passwordHash → bcrypt.compare() → boolean match
No rotation/reset path exists; hash is fixed at registration time for the life of the account.
```

---

### Business Rule: JWT Token Issuance and Claims Structure

**Overview**:
On successful login, `AuthService.signToken` (auth.service.ts:47-50) issues a signed JSON Web Token containing three custom claims — `sub` (the user's UUID), `email`, and `role` — signed with the HMAC secret `env.JWT_SECRET` and an expiry governed by `env.JWT_EXPIRES_IN` (default `"8h"`, configurable via environment variable, validated as a required non-empty string by `envSchema` in `src/config/env.ts:9`).

**Detailed description**:
The token's payload intentionally embeds the user's `role` claim directly, meaning role-based authorization decisions made by `requireRole` (auth.middleware.ts:49-61) rely entirely on the claim baked into the token at issuance time rather than a fresh database lookup on every request. This is a performance-oriented design choice (avoids a DB round-trip per authenticated request) but introduces a **stale-authorization risk**: if an administrator changes a user's role after a token has been issued (e.g., demoting an `ADMIN` to `OPERATOR`, or vice versa), the change will not take effect until the existing token expires and the user logs in again, because there is no token revocation or role-refresh mechanism anywhere in the codebase. Given the default 8-hour expiry, a demoted user could retain elevated privileges for up to 8 hours after the change.

The signing algorithm is implicitly HS256 (the `jsonwebtoken` library default when only a string secret is provided, as is the case here — `env.JWT_SECRET` is a `z.string()` per `src/config/env.ts:8`), meaning the same secret used to sign tokens is also used to verify them in `auth.middleware.ts:41`. `AuthService` and the `authenticate` middleware are therefore symmetric dependents on the same shared `env.JWT_SECRET` value; a mismatch or rotation of this secret between deployments (e.g., in a multi-instance deployment where instances don't share the same environment configuration) would cause tokens issued by one instance to fail verification on another, though in this codebase's monolithic Express structure this risk only materializes with in-place secret rotation, not with topology.

The response returned to the client (`AuthTokens` type, auth.service.ts:10-14) explicitly reports `expiresIn` as the raw configuration string (e.g. `"8h"`) rather than a computed absolute timestamp or a numeric seconds value, and always reports `tokenType: 'Bearer'` as a literal constant. There is no refresh-token issuance, meaning once the access token expires, the only recourse for the client is a full re-login via `/api/v1/auth/login`, re-supplying credentials.

**Rule workflow**:
```
login success → signToken(userId, email, role):
  payload = { sub: userId, email, role }
  options = { expiresIn: env.JWT_EXPIRES_IN }
  token = jwt.sign(payload, env.JWT_SECRET, options)
→ response: { user: PublicUser, tokens: { accessToken: token, expiresIn: env.JWT_EXPIRES_IN, tokenType: 'Bearer' } }

Downstream verification (auth.middleware.ts):
  jwt.verify(token, env.JWT_SECRET) → { sub, email, role, iat, exp }
  req.user = { id: sub, email, role }
```

---

### Business Rule: Registration Delegation and Duplicate Email Prevention

**Overview**:
`AuthService.register` (auth.service.ts:27-29) is a one-line pass-through delegation to `UserService.createUser`, which enforces email uniqueness by first querying `UserRepository.findByEmail` and throwing `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` (HTTP 409) if a user with that email already exists (user.service.ts:22-25), before the new user is ever hashed or persisted.

**Detailed description**:
This rule places the uniqueness check at the application layer (an explicit `findByEmail` query before `create`) rather than relying solely on the database-level `@unique` constraint on `User.email` (`prisma/schema.prisma:27`). This is a "check-then-act" pattern that is vulnerable to a race condition under concurrent requests: if two registration requests for the same email arrive close together, both could pass the `findByEmail` check (both returning `null`) before either `create` call completes, resulting in a second Prisma-level unique-constraint violation (`P2002`) on the losing request. This scenario is explicitly handled — but at a different layer — by `errorMiddleware` (error.middleware.ts:37-46), which catches `Prisma.PrismaClientKnownRequestError` with code `P2002` and maps it to a generic `409 CONFLICT` response, distinct in error code (`CONFLICT` vs `EMAIL_ALREADY_USED`) from the application-level check. This means that under race conditions, callers of `/api/v1/auth/register` may observe a different `error.code` (`CONFLICT` instead of `EMAIL_ALREADY_USED`) than under sequential requests, an inconsistency a strict API contract test would likely need to account for.

Because `AuthService.register` performs no additional logic of its own — no auth-specific side effects such as sending a welcome email, auto-login after registration, or emitting a domain event — its business value is purely as a facade that gives the `auth` module a complete, self-contained public API surface (register + login) without duplicating `UserService`'s logic. This is a positive separation-of-concerns choice: registration business rules (uniqueness, hashing) live once, in `UserService`, and are reused identically whether a user is created via the public registration endpoint or (hypothetically) via an internal admin-provisioning flow that might call `UserService.createUser` directly.

Notably, registration does **not** auto-login the newly created user — the response body of `POST /api/v1/auth/register` is a bare `PublicUser` object (auth.controller.ts:14-15), with no `tokens` field, confirmed by the integration test `tests/auth.test.ts:6-17` which asserts only `res.body` matches user fields and does not reference `res.body.tokens`. A client must make a separate `POST /api/v1/auth/login` call after registering to obtain an access token.

**Rule workflow**:
```
register(input) → UserService.createUser(input):
  findByEmail(input.email) → found → throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED') [409]
  findByEmail(input.email) → not found →
    bcrypt.hash(input.password, 10) → passwordHash
    UserRepository.create({ email, passwordHash, name, role }) → persisted User
    return toPublic(created)  [id, email, name, role, createdAt, updatedAt — no passwordHash]
→ AuthController.register responds 201 with PublicUser (no tokens issued)

Race condition fallback (not in AuthService, but affects its callers):
  Concurrent duplicate insert → Prisma P2002 → errorMiddleware → 409 { error: { code: 'CONFLICT' } }
```

---

### Business Rule: Input Validation Boundaries (Registration vs. Login)

**Overview**:
Registration and login use distinct Zod schemas (`registerSchema` and `loginSchema`, `auth.schemas.ts:3-13`) with intentionally different password constraints: registration requires 8-72 characters, while login only requires 1-72 characters (i.e., merely non-empty).

**Detailed description**:
This asymmetry is a standard and correct practice: password *strength* rules (minimum length, and by extension any future complexity rules) should only be enforced at the point where a password is being *set* (registration), not at the point where it is being *verified* (login). Enforcing an 8-character minimum on login as well would create a scenario where legacy accounts with weaker passwords (if the minimum were ever raised, or if data were migrated from another system) become permanently unable to log in even with their correct, existing password — the login schema's role is purely to ensure a non-empty string is sent, deferring the actual correctness check to `bcrypt.compare` inside `AuthService.login`.

Both schemas cap the password field at 72 characters. This is not an arbitrary number — it corresponds to bcrypt's well-known input limitation, where the algorithm silently truncates any input beyond 72 bytes. By capping at the schema level, the system ensures that validation feedback is explicit (a 400 `VALIDATION_ERROR` with a clear message) rather than allowing a longer password to be silently and confusingly truncated by bcrypt internally, which could otherwise lead to unexpected authentication behavior (e.g., two different long passwords that share the first 72 bytes would hash identically and both "work").

The `email` field is validated identically in both schemas (`z.string().email().max(255)`), and the `role` field is only present in `registerSchema`, defaulting to `'OPERATOR'` (auth.schemas.ts:7) if omitted — meaning any registration request that does not explicitly request `ADMIN` will silently receive the lower-privilege role, and there is no server-side authorization check preventing an unauthenticated caller from directly requesting `role: 'ADMIN'` at registration time. This is a notable finding: **registration is a public, unauthenticated endpoint that allows self-service creation of ADMIN accounts** by simply passing `role: "ADMIN"` in the request body, since no invitation code, approval workflow, or authenticated-admin-only gate exists on `/api/v1/auth/register` (`auth.routes.ts:10`, using only the `validate` middleware, not `authenticate`/`requireRole`).

**Rule workflow**:
```
POST /register body → registerSchema.parse():
  email: valid email, ≤255 chars
  password: 8-72 chars
  name: 2-150 chars
  role: 'ADMIN' | 'OPERATOR', defaults to 'OPERATOR' if omitted
  → validation failure: ValidationError (400, VALIDATION_ERROR, field-level details)

POST /login body → loginSchema.parse():
  email: valid email, ≤255 chars
  password: 1-72 chars (non-empty only, no strength check)
  → validation failure: ValidationError (400, VALIDATION_ERROR, field-level details)
```

---

## 4. Component Structure

```
src/modules/auth/
├── auth.service.ts        # AuthService: register() delegate, login() with bcrypt+JWT, signToken() private helper
├── auth.controller.ts      # AuthController: HTTP handlers (register, login, me) — thin, delegates to services
├── auth.schemas.ts         # Zod schemas: registerSchema, loginSchema + inferred RegisterInput/LoginInput types
└── auth.routes.ts          # buildAuthRouter(): wires POST /register, POST /login, GET /me with middleware chains

Directly related component (authorization, not part of AuthService but consumes its token contract):
src/middlewares/
└── auth.middleware.ts      # authenticate (JWT verification), requireRole (RBAC gate), AuthUser type, Express Request augmentation

Upstream collaborators (in src/modules/users/, not part of AuthService's boundary but directly depended on):
src/modules/users/
├── user.repository.ts      # UserRepository: findById, findByEmail, create (Prisma data access)
└── user.service.ts         # UserService: createUser (hashing+uniqueness), getById, static toPublic() DTO mapper

Cross-cutting collaborators consumed by AuthService:
src/config/env.ts           # env: JWT_SECRET, JWT_EXPIRES_IN (validated via Zod at process startup)
src/shared/errors/          # AppError hierarchy: UnauthorizedError, ConflictError, etc.
src/middlewares/error.middleware.ts  # Global error handler mapping AppError/ZodError/Prisma errors to JSON
```

---

## 5. Dependency Analysis

```
Internal Dependencies:
AuthController → AuthService → UserRepository (findByEmail)
AuthService → UserService (createUser, static toPublic)
AuthService → env (JWT_SECRET, JWT_EXPIRES_IN)
AuthService → UnauthorizedError (shared/errors)
AuthController → UserService (getById, for the /me endpoint)
auth.routes.ts → validate.middleware.ts (schema-based request validation)
auth.routes.ts → auth.middleware.ts (authenticate, for GET /me)
auth.middleware.ts → env (JWT_SECRET) [independent verification path, shares secret with AuthService]
app.ts (composition root) → instantiates AuthService(userRepository, userService), AuthController(authService, userService)

External Dependencies:
- bcrypt (5.1.1) - Password hashing (registration, via UserService) and comparison (login, in AuthService)
- jsonwebtoken (9.0.2) - JWT signing (AuthService.signToken) and verification (auth.middleware.ts)
- zod (3.23.8) - Schema validation for auth.schemas.ts and env.ts
- @prisma/client (5.22.0) - ORM client used transitively via UserRepository (MySQL datasource)
- express (4.21.1) - Routing and middleware framework (transitively, via AuthController/auth.routes.ts)
```

---

## 6. Afferent and Efferent Coupling

| Component | Afferent Coupling | Efferent Coupling | Critical |
|-----------|--------------------|--------------------|----------|
| AuthService | 2 (AuthController, app.ts composition root) | 5 (UserRepository, UserService, env, UnauthorizedError, bcrypt/jwt libs) | High |
| AuthController | 2 (auth.routes.ts, app.ts) | 3 (AuthService, UserService, UnauthorizedError) | Medium |
| auth.middleware.ts (authenticate/requireRole) | 3 (auth.routes.ts, user.routes.ts, app.ts indirectly via all protected routers) | 3 (env, jsonwebtoken, shared/errors) | High |
| auth.schemas.ts | 2 (auth.routes.ts, auth.service.ts as type source) | 1 (zod) | Low |
| auth.routes.ts | 1 (routes/index.ts) | 4 (auth.controller.ts, auth.schemas.ts, validate.middleware.ts, auth.middleware.ts) | Low |

Notes on interpretation: "Afferent" is counted as the number of distinct internal modules/files that import or instantiate the given unit; "Efferent" is the number of distinct internal/external units it imports or calls. `AuthService` is rated High criticality because, despite modest direct afferent coupling (only instantiated once, in `app.ts`, and consumed by one controller), it is the sole issuer of the JWT contract that the entire application's authorization surface (`auth.middleware.ts`, consumed by every protected route in `orders`, `customers`, `products`, `users`) implicitly depends upon. A change to `signToken`'s claim structure would ripple to every protected endpoint in the system without any of them directly importing `AuthService`.

---

## 7. Endpoints

| Endpoint | Method | Description | Auth Required | Role Restriction |
|----------|--------|--------------|----------------|-------------------|
| /api/v1/auth/register | POST | Register a new user (email, password, name, optional role) | No | None (self-service; can request ADMIN role, see Business Rules §4) |
| /api/v1/auth/login | POST | Authenticate with email/password, receive JWT access token | No | None |
| /api/v1/auth/me | GET | Get the currently authenticated user's profile | Yes (Bearer JWT) | None (any authenticated role) |

Note: `GET /api/v1/auth/me` is handled by `AuthController.me`, which internally delegates to `UserService.getById`, not to `AuthService` directly — it is included here because it is part of the `auth` module's exposed route surface (`auth.routes.ts:12`) and is the direct consumer of the `req.user` context populated by the JWT issued by `AuthService.login`.

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|--------------|-----------------|
| UserRepository / MySQL (via Prisma) | Internal / Database | Lookup user by email for login; existence check for registration | Prisma Client (SQL under the hood) | Prisma `User` model / JSON | Errors from Prisma propagate to `errorMiddleware`; `P2002`/`P2025` mapped to 409/404 |
| bcrypt (npm library) | External Library | Password hashing (via UserService) and verification (in AuthService.login) | In-process function call | N/A (binary/native addon) | No explicit error handling around `bcrypt.compare`/`bcrypt.hash` calls — exceptions would propagate as unhandled errors to `errorMiddleware`'s generic 500 handler |
| jsonwebtoken (npm library) | External Library | JWT signing (AuthService.signToken) and verification (auth.middleware.ts) | In-process function call | JWT (base64url JSON + HMAC signature) | `jwt.verify` failures caught explicitly in `auth.middleware.ts:40-46` and mapped to `UnauthorizedError`; `jwt.sign` in `AuthService` has no explicit try/catch (relies on it not throwing under normal config) |
| env config (Zod-validated) | Internal / Configuration | Supplies `JWT_SECRET`, `JWT_EXPIRES_IN` | Process environment variables | Plain strings | Fails fast at process startup (`process.exit(1)`) if `JWT_SECRET` is missing or under 16 chars — not a runtime error surfaced through HTTP |
| UserService | Internal / Service | Registration delegation, DTO mapping (`toPublic`) | Direct method call (in-process) | TypeScript objects | Errors (`ConflictError`, `NotFoundError`) are `AppError` subclasses, propagate to `errorMiddleware` |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|-----------------|----------|---------|
| Dependency Injection (constructor injection) | `AuthService` receives `UserRepository` and `UserService` via constructor | `src/modules/auth/auth.service.ts:22-25` | Testability and decoupling from concrete instantiation; wiring centralized in `app.ts` |
| Facade / Delegation | `AuthService.register` is a pure pass-through to `UserService.createUser` | `src/modules/auth/auth.service.ts:27-29` | Keeps the public `auth` module API cohesive while avoiding duplicating user-creation logic |
| Repository Pattern | `UserRepository` abstracts Prisma data access behind `findById`/`findByEmail`/`create` | `src/modules/users/user.repository.ts` | Data access abstraction, consumed directly by `AuthService` for the login lookup (bypassing `UserService`) |
| DTO / View Model Mapping | `UserService.toPublic` static method strips `passwordHash` from the Prisma `User` entity | `src/modules/users/user.service.ts:42-51` | Prevents sensitive field leakage in API responses; reused by both `AuthService.login` and `UserService.createUser`/`getById` |
| Middleware Chain (Express) | `validate` → `authenticate` → `requireRole` → controller handler, composed per-route | `src/modules/auth/auth.routes.ts`, `src/modules/users/user.routes.ts` | Cross-cutting concerns (validation, authN, authZ) applied declaratively at the routing layer, outside `AuthService` |
| Stateless Token Authentication (JWT) | `AuthService.signToken` issues, `authenticate` middleware verifies, no server-side session store | `src/modules/auth/auth.service.ts:47-50`, `src/middlewares/auth.middleware.ts:27-46` | Horizontally scalable authentication without shared session state; trade-off is no revocation capability |
| Centralized Error Hierarchy | `AppError` base class with typed subclasses (`UnauthorizedError`, `ConflictError`, etc.) caught by one global `errorMiddleware` | `src/shared/errors/*.ts`, `src/middlewares/error.middleware.ts` | Consistent HTTP error response shape (`{ error: { code, message, details? } }`) across the entire API, including all `AuthService`-thrown errors |
| Fail-Fast Configuration Validation | `env.ts` validates all required environment variables via Zod at process startup, calling `process.exit(1)` on failure | `src/config/env.ts:14-25` | Prevents the service from starting with a missing/invalid `JWT_SECRET`, which `AuthService` and `auth.middleware.ts` both depend on |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|------------------|-------|--------|
| High | AuthService / auth.middleware.ts | No token revocation, logout, or refresh-token mechanism; JWTs remain valid until natural expiry (default 8h) | A compromised or leaked token cannot be invalidated server-side; role changes/demotions do not take effect until the token expires |
| High | auth.routes.ts / auth.schemas.ts | `POST /api/v1/auth/register` is unauthenticated and accepts an arbitrary `role` field, allowing self-service creation of `ADMIN` accounts | Privilege escalation risk — any anonymous caller can become an administrator by specifying `role: "ADMIN"` at registration |
| High | Login endpoint (auth.routes.ts, AuthService.login) | No rate limiting, CAPTCHA, or account lockout on repeated failed login attempts anywhere in the codebase | Endpoint is vulnerable to brute-force and credential-stuffing attacks with no mitigation |
| Medium | AuthService.login | Timing side-channel: `bcrypt.compare` is only invoked when the user is found, so response time differs between "email not found" and "wrong password" cases despite identical error messages | Partial user-enumeration vector via timing analysis, undermining the otherwise-correct uniform error message design |
| Medium | UserService.createUser (dependency of AuthService.register) | Email-uniqueness check is "check-then-act" (`findByEmail` then `create`), a TOCTOU race condition under concurrent identical registrations | Under concurrency, duplicate-email requests can surface an inconsistent error code (`CONFLICT` from Prisma P2002 vs. `EMAIL_ALREADY_USED` from the application check) |
| Medium | UserService.createUser / tests/helpers/factories.ts | bcrypt cost factor (`10`) is a hardcoded constant duplicated in two locations (`user.service.ts:16` and `factories.ts:21`) rather than centrally configured | No environment-specific tuning; risk of the two values silently diverging on future edits |
| Medium | AuthService / users module | No password reset, password change, or account recovery mechanism exists anywhere in the codebase | Users with a compromised or forgotten password have no self-service remediation path |
| Low | AuthService.signToken | No explicit error handling around `jwt.sign`; relies on the call never throwing under valid configuration | Low likelihood given `JWT_SECRET` is validated at startup, but any unexpected `jwt.sign` failure would surface as an unhandled 500 rather than a domain-specific error |
| Low | AuthService | No dedicated unit tests exist for `AuthService` in isolation; all coverage is via HTTP integration tests | Slower feedback loop for logic-only changes (e.g., token claim structure); mocking `UserRepository`/`UserService` directly would allow faster, more targeted tests |
| Low | auth.middleware.ts | `requireRole` is defined but used in only one route (`GET /api/v1/users/:id`) across the entire API; `orders`, `customers`, and `products` routes only check authentication, not role | Underused RBAC infrastructure suggests either an intentional single-tier authorization model or an incomplete rollout of role-based restrictions — ambiguous without further product context |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage (Behavioral) | Test Quality |
|-----------|------------|---------------------|--------------------------|----------------|
| AuthService (register) | 0 | 3 (`tests/auth.test.ts:6-39`: successful register+201, duplicate email+409, invalid payload+400) | Good — happy path, conflict, and validation-failure paths all exercised via HTTP | Solid assertions (status code, response shape, absence of `passwordHash`/`password` fields, error code checks); no direct unit-level assertion on `AuthService.register`'s delegation behavior in isolation |
| AuthService (login) | 0 | 2 (`tests/auth.test.ts:41-63`: successful login+200 with token, wrong password+401) | Moderate — covers success and wrong-password failure; does not explicitly test the "unknown email" failure branch as a distinct assertion, nor malformed/missing-body edge cases beyond schema validation | Good assertions on token type and structure (`tokenType: 'Bearer'`, `accessToken` is a string); does not assert on JWT claim contents (`sub`, `email`, `role`) or expiry behavior |
| signToken (private method) | 0 | 0 (indirectly exercised only through the login integration test) | Indirect only | Not directly testable as a private method without reflection or refactor; no test verifies claim structure, algorithm, or expiry configuration directly |
| authenticate middleware | 0 | 2 (`tests/auth.test.ts:65-83`: `/me` with valid token succeeds, `/me` without token returns 401) + indirectly via every `orders.test.ts` authenticated request (`bootstrapAuthenticatedUser`) | Good indirect coverage through downstream consumers (`orders.test.ts`) | No explicit test for malformed `Authorization` header (missing "Bearer " prefix), expired token, or tampered/invalid-signature token scenarios in the visible test files |
| requireRole middleware | 0 | 0 (no test file exercises the `ADMIN`-only `GET /api/v1/users/:id` route with an `OPERATOR` token to confirm 403 behavior, based on files examined) | Low / Unverified | No dedicated `users.test.ts` was found in the `tests/` directory during this analysis; RBAC enforcement on the one protected route appears untested |
| Test infrastructure (tests/helpers/factories.ts) | N/A | N/A | N/A | `createTestUser`, `loginAndGetToken`, and `bootstrapAuthenticatedUser` provide clean, reusable fixtures; `bootstrapAuthenticatedUser` is reused extensively by `tests/orders.test.ts`, making `AuthService.login`'s correctness a load-bearing dependency for the entire authenticated test suite (a regression in login would cascade to unrelated test failures across modules) |

Test files located: `tests/auth.test.ts` (primary), `tests/orders.test.ts` (indirect consumer via `bootstrapAuthenticatedUser`), `tests/helpers/factories.ts` (shared fixtures), `tests/setup.ts` (test environment bootstrap, not read in detail as out of component boundary). No `tests/users.test.ts`, `tests/customers.test.ts`, or `tests/products.test.ts` were identified as present in the `tests/` directory listing performed during this analysis — meaning `requireRole` and role-restricted behavior have no located dedicated test coverage.

---
