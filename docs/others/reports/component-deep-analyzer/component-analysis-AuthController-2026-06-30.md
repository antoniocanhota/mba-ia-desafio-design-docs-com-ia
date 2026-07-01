# Component Deep Analysis Report: AuthController

**Project:** order-management-api (Order Management System REST API)
**Component:** AuthController
**Primary file:** `src/modules/auth/auth.controller.ts`
**Analysis date:** 2026-06-30

---

## 1. Executive Summary

`AuthController` is the HTTP-facing entry point for authentication in the Order Management API. It is a thin Express controller class that exposes three request handlers — `register`, `login`, and `me` — and delegates all business logic to `AuthService` (for registration/login) and `UserService` (for fetching the authenticated user's profile). It does not talk to the database or Prisma directly, and it does not implement password hashing, JWT signing, or token validation itself; those responsibilities live in `AuthService` (hashing/signing indirectly via `UserService`/`bcrypt`/`jsonwebtoken`) and in the `authenticate` Express middleware (`src/middlewares/auth.middleware.ts`), which is wired at the routing layer rather than inside the controller.

The component's role in the system is narrow and well-bounded: it converts HTTP requests into calls against the auth/user services and converts service results (or thrown errors) into HTTP responses, forwarding all errors to Express's `next()` for centralized handling by `errorMiddleware`. Authentication itself (JWT verification on protected routes) is enforced by route-level middleware (`authenticate` in `auth.routes.ts`) before the controller's `me` handler is even invoked — the controller only reacts to `req.user` being present or absent.

Key findings:
- The controller class itself is extremely small (39 lines) with no business logic — a textbook "thin controller" pattern that pushes rules into `AuthService`/`UserService`.
- Password hashing (bcrypt, 10 rounds) happens in `UserService.createUser`, not in the auth module — meaning `AuthController.register` transitively depends on `UserService`'s hashing policy.
- JWT signing (`AuthService.signToken`) embeds `sub` (user id), `email`, and `role` claims with an expiry driven by `env.JWT_EXPIRES_IN` (default `8h`).
- Role is a client-suppliable field at registration (`ADMIN` or `OPERATOR`, default `OPERATOR`) with **no authorization gate** — anyone can self-register as `ADMIN`. This is flagged as a significant risk in Section 10.
- Test coverage for the three endpoints is present and reasonably thorough (`tests/auth.test.ts`), including negative cases (duplicate email, wrong password, missing token, invalid payload). `tests/orders.test.ts` also exercises the login endpoint indirectly via its `bootstrapAuthenticatedUser` test helper.

---

## 2. Data Flow Analysis

### POST /api/v1/auth/register

```
1. Request enters Express app (src/app.ts) -> mounted at /api/v1 -> /auth (src/routes/index.ts:24)
2. Router match: POST /register (src/modules/auth/auth.routes.ts:10)
3. Middleware: validate({ body: registerSchema }) (src/middlewares/validate.middleware.ts)
   - Zod-parses req.body against registerSchema (auth.schemas.ts:3-8)
   - On failure -> ValidationError(400, VALIDATION_ERROR) -> next(err) -> errorMiddleware
4. Controller: AuthController.register (auth.controller.ts:12-19)
   - calls this.authService.register(req.body)
5. Service: AuthService.register (auth.service.ts:27-29)
   - delegates directly to this.userService.createUser(input)
6. Service: UserService.createUser (user.service.ts:21-34)
   - UserRepository.findByEmail -> Prisma query (user.repository.ts:10-12)
   - if existing user -> throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED')
   - bcrypt.hash(password, 10) -> passwordHash
   - UserRepository.create -> Prisma insert (user.repository.ts:14-16)
   - UserService.toPublic(created) strips passwordHash from the response shape
7. Controller formats response: res.status(201).json(user)  [PublicUser, no password/passwordHash]
8. On any thrown error in steps 3-6 -> caught in controller try/catch -> next(err) -> errorMiddleware.ts
   formats a JSON { error: { code, message, details? } } envelope and sets the HTTP status.
```

### POST /api/v1/auth/login

```
1. Request enters Express app -> /api/v1/auth/login
2. Router match: POST /login (auth.routes.ts:11)
3. Middleware: validate({ body: loginSchema }) validates email/password shape
4. Controller: AuthController.login (auth.controller.ts:21-28)
   - calls this.authService.login(req.body)
5. Service: AuthService.login (auth.service.ts:31-45)
   a. UserRepository.findByEmail(input.email) -> Prisma lookup
      - if not found -> throw UnauthorizedError('Invalid credentials') [401]
   b. bcrypt.compare(input.password, user.passwordHash)
      - if mismatch -> throw UnauthorizedError('Invalid credentials') [401]
   c. AuthService.signToken(user.id, user.email, user.role)
      - jwt.sign({ sub, email, role }, env.JWT_SECRET, { expiresIn: env.JWT_EXPIRES_IN })
   d. returns { user: PublicUser, tokens: { accessToken, expiresIn, tokenType: 'Bearer' } }
6. Controller: res.status(200).json(result)
7. Errors propagate via next(err) -> errorMiddleware -> JSON error envelope
```

### GET /api/v1/auth/me

```
1. Request enters Express app -> /api/v1/auth/me
2. Router match: GET /me (auth.routes.ts:12), guarded by `authenticate` middleware BEFORE the controller runs
3. Middleware: authenticate (src/middlewares/auth.middleware.ts:27-47)
   - reads Authorization header, requires "Bearer <token>" format
   - missing/malformed header -> UnauthorizedError('Missing or invalid Authorization header') [401]
   - jwt.verify(token, env.JWT_SECRET) -> on failure -> UnauthorizedError('Invalid or expired token') [401]
   - on success: req.user = { id: payload.sub, email: payload.email, role: payload.role }
4. Controller: AuthController.me (auth.controller.ts:30-38)
   - defensive check: if (!req.user) throw new UnauthorizedError() [should be unreachable given middleware guarantees]
   - calls this.userService.getById(req.user.id)
5. Service: UserService.getById (user.service.ts:36-40)
   - UserRepository.findById -> Prisma lookup
   - if not found -> throw NotFoundError('User') [404] (edge case: valid token but user deleted after issuance)
   - returns PublicUser (toPublic strips passwordHash)
6. Controller: res.status(200).json(user)
7. Errors -> next(err) -> errorMiddleware
```

---

## 3. Business Rules & Logic

### Overview of the business rules

| Rule Type | Rule Description | Location |
|-----------|-------------------|----------|
| Validation | Email must be valid format, max 255 chars | auth.schemas.ts:4, auth.schemas.ts:11 |
| Validation | Password minimum 8 chars, maximum 72 chars (registration) | auth.schemas.ts:5 |
| Validation | Password only requires min 1 char, max 72 chars on login (no strength check) | auth.schemas.ts:12 |
| Validation | Name minimum 2 chars, maximum 150 chars | auth.schemas.ts:6 |
| Validation | Role restricted to enum ADMIN or OPERATOR, defaults to OPERATOR | auth.schemas.ts:7 |
| Business Logic | Registration rejects duplicate email (case-sensitive exact match) | user.service.ts:22-25 |
| Business Logic | Password stored only as bcrypt hash (10 salt rounds), never persisted in plaintext | user.service.ts:16, 26 |
| Business Logic | Registration response never exposes passwordHash or password | user.service.ts:42-51 |
| Business Logic | Login requires exact email match then bcrypt.compare against stored hash | auth.service.ts:32-39 |
| Business Logic | Login failure (unknown email or wrong password) returns identical generic error to avoid user enumeration | auth.service.ts:33-39 |
| Business Logic | Successful login issues a signed JWT embedding sub, email, and role claims | auth.service.ts:40-49 |
| Business Logic | JWT expiry is configurable via env.JWT_EXPIRES_IN (default 8h) | config/env.ts:9, auth.service.ts:48 |
| Business Logic | Bearer token format strictly enforced ("Bearer " prefix, case-insensitive) on protected routes | auth.middleware.ts:29 |
| Business Logic | Invalid/expired/malformed JWT rejected uniformly with 401 Unauthorized | auth.middleware.ts:44-46 |
| Business Logic | GET /me resolves the current user from the DB by token subject (id), not from token claims directly | auth.controller.ts:33 |
| Business Logic | Role-based access control primitive (requireRole) exists in the middleware but is not used by any auth module route | auth.middleware.ts:49-61 |
| Security Gap | Role is caller-controlled at registration; no restriction prevents self-registration as ADMIN | auth.schemas.ts:7, auth.service.ts:27-29 |

### Detailed breakdown of the business rules

---

#### Business Rule: Registration Input Validation

**Overview:**
All registration requests must satisfy a strict Zod schema (`registerSchema`) before reaching any business logic. This is enforced by the `validate` middleware at the routing layer, not inside the controller itself.

**Detailed description:**
The `registerSchema` (auth.schemas.ts:3-8) requires `email` to be a syntactically valid email address no longer than 255 characters, `password` to be between 8 and 72 characters, `name` to be between 2 and 150 characters, and `role` to be one of the literal strings `ADMIN` or `OPERATOR`, defaulting to `OPERATOR` when omitted. These constraints exist purely at the transport/validation boundary — they run before `AuthController.register` is invoked, via the `validate({ body: registerSchema })` middleware wired in `auth.routes.ts:10`. If validation fails, a `ValidationError` (HTTP 400, code `VALIDATION_ERROR`) is thrown with a `details` array describing each failing field, and the controller method is never reached.

The 72-character password cap is notable: it matches bcrypt's well-known 72-byte input truncation limit, suggesting the schema was deliberately aligned with the hashing algorithm's constraints in `UserService.createUser` (`bcrypt.hash(input.password, BCRYPT_ROUNDS)`, user.service.ts:26). There is no complexity requirement (uppercase/lowercase/digit/symbol) beyond the 8-character minimum, and no check against common/breached password lists.

This validation is entirely decoupled from the controller — `AuthController.register` trusts that `req.body` has already been coerced into a `RegisterInput` shape by the time it executes. This means the controller has zero responsibility for input correctness; all its test coverage for invalid payloads (see `tests/auth.test.ts:30-39`) is effectively testing the `validate` middleware + `auth.schemas.ts`, not the controller's own code path.

**Rule workflow:**
```
Request body -> validate middleware -> registerSchema.parse()
  -> success: req.body replaced with typed RegisterInput, next()
  -> failure: ZodError caught -> ValidationError(400, VALIDATION_ERROR, details[]) -> errorMiddleware
```

---

#### Business Rule: Duplicate Email Prevention on Registration

**Overview:**
A user cannot register with an email address that already exists in the system; the system enforces uniqueness at the application layer before attempting to persist.

**Detailed description:**
`UserService.createUser` (user.service.ts:21-34) performs an explicit `findByEmail` lookup via `UserRepository` before hashing the password or inserting a new row. If a user with the same email already exists, a `ConflictError('Email already registered', 'EMAIL_ALREADY_USED')` is thrown, which the shared error middleware translates into an HTTP 409 response with `error.code = "EMAIL_ALREADY_USED"`. This is confirmed by `tests/auth.test.ts:19-28`, which creates a user via the test factory and then attempts to register again with the same email, asserting a 409 status and the exact error code.

This is a check-then-act pattern executed at the application level rather than relying solely on the database's unique constraint (the Prisma schema likely also enforces uniqueness on `email`, given `findByEmail` uses `where: { email }` in a way consistent with a unique index — see `user.repository.ts:10-12`). The `errorMiddleware` (error.middleware.ts:37-47) also independently handles Prisma's `P2002` unique-constraint-violation code as a fallback safety net, returning a generic `CONFLICT` error if a race condition slips through the initial existence check (e.g., two concurrent registrations with the same email). This dual-layer protection (app-level check + DB-level constraint + middleware fallback) suggests defense-in-depth against race conditions, though the two failure paths produce different error codes (`EMAIL_ALREADY_USED` vs generic `CONFLICT`), which is a minor inconsistency a client would need to handle.

Because `AuthController.register` merely calls `authService.register`, which in turn is a direct passthrough to `userService.createUser`, all of this duplicate-detection logic is entirely outside the `auth.controller.ts` file itself — it lives in the `users` module, one hop away from the analyzed component's boundary.

**Rule workflow:**
```
AuthController.register -> AuthService.register -> UserService.createUser
  -> UserRepository.findByEmail(email)
     -> found: throw ConflictError('Email already registered', 'EMAIL_ALREADY_USED') [409]
     -> not found: proceed to hash + create
```

---

#### Business Rule: Password Storage via bcrypt Hashing

**Overview:**
Plaintext passwords are never persisted; they are hashed with bcrypt (10 salt rounds) before being written to the database, and the hash is never returned in any API response.

**Detailed description:**
When a new user registers, `UserService.createUser` calls `bcrypt.hash(input.password, BCRYPT_ROUNDS)` where `BCRYPT_ROUNDS` is a module-level constant set to `10` (user.service.ts:16, 26). The resulting hash is stored in the `passwordHash` column via `UserRepository.create`. The `AuthController.register` handler receives back only a `PublicUser` object (constructed via the static `UserService.toPublic` method, user.service.ts:42-51), which explicitly whitelists `id`, `email`, `name`, `role`, `createdAt`, and `updatedAt` — `passwordHash` is never included in the object returned to the controller, and therefore never serialized into the JSON HTTP response. This is directly verified by `tests/auth.test.ts:15-16`, which asserts the response body does not contain `passwordHash` or `password` properties.

During login, the same hash is used for verification: `AuthService.login` calls `bcrypt.compare(input.password, user.passwordHash)` (auth.service.ts:36), which performs a constant-time-safe comparison intrinsic to bcrypt's design, mitigating timing-attack risks on password comparison. The cost factor of 10 rounds is a hardcoded constant rather than an environment-configurable value, meaning any future need to tune the hashing cost (e.g., in response to hardware improvements or security audits) requires a code change and redeploy rather than a config change.

Because `AuthController` never directly calls `bcrypt` (all hashing/verification is delegated to `UserService` and `AuthService` respectively), the controller itself has no direct coupling to the hashing algorithm choice — it only interacts with these concerns transitively through the two service dependencies injected into its constructor.

**Rule workflow:**
```
Register: plaintext password -> bcrypt.hash(password, 10) -> passwordHash -> persisted
Login: plaintext password + stored passwordHash -> bcrypt.compare() -> boolean match
Response: PublicUser (whitelisted fields only) -> passwordHash never serialized
```

---

#### Business Rule: Login Credential Verification with Generic Error Messaging

**Overview:**
Login fails uniformly with the same message ("Invalid credentials") whether the email does not exist or the password is wrong, preventing user-enumeration attacks via error-message differentiation.

**Detailed description:**
`AuthService.login` (auth.service.ts:31-45) performs two sequential checks: first, it looks up the user by email via `UserRepository.findByEmail`; if no user is found, it immediately throws `UnauthorizedError('Invalid credentials')`. Second, if a user is found, it compares the supplied password against the stored hash via `bcrypt.compare`; if the comparison fails, it throws the exact same `UnauthorizedError('Invalid credentials')`. Both paths produce HTTP 401 with `error.code = "UNAUTHORIZED"` and an identical message string, so an external observer cannot distinguish "email not registered" from "email registered but wrong password" purely from the response body. This is a deliberate (or at minimum, effectively achieved) security control against email/username enumeration attacks.

This is validated by `tests/auth.test.ts:54-63`, which tests the wrong-password case and asserts a 401 with `UNAUTHORIZED` code; the "unknown email" case is not explicitly tested in the reviewed test file, but the code path is symmetric and shares the identical error construction, so it can be inferred with high confidence that behavior is consistent (not independently verified by a dedicated test — flagged as a minor test-coverage gap in Section 11).

One nuance: while the error *message* and *HTTP status* are identical between the two failure modes, timing differences could theoretically still leak information — the "user not found" path returns immediately without invoking `bcrypt.compare` (which is deliberately slow, ~10 rounds), while the "wrong password" path always incurs the bcrypt comparison cost. This creates a timing side-channel that could allow an attacker to distinguish valid from invalid emails via response latency, though exploiting this in practice requires statistical timing analysis and is a lower-severity, commonly-accepted trade-off in many systems.

**Rule workflow:**
```
AuthService.login(email, password):
  user = findByEmail(email)
  if !user -> throw UnauthorizedError('Invalid credentials') [401, fast path]
  ok = bcrypt.compare(password, user.passwordHash)
  if !ok -> throw UnauthorizedError('Invalid credentials') [401, slow path ~10 bcrypt rounds]
  -> proceed to token issuance
```

---

#### Business Rule: JWT Issuance on Successful Login

**Overview:**
A successful login issues a signed JSON Web Token embedding the user's id, email, and role, valid for a configurable expiry window, returned to the client alongside the public user profile.

**Detailed description:**
Upon successful credential verification, `AuthService.signToken` (auth.service.ts:47-50) constructs a JWT via `jsonwebtoken`'s `jwt.sign()`, with a payload of `{ sub: userId, email, role }` and options `{ expiresIn: env.JWT_EXPIRES_IN }`, signed using the HMAC secret `env.JWT_SECRET` (algorithm defaults to `jsonwebtoken`'s library default, HS256, since no `algorithm` option is explicitly set). The `env.JWT_SECRET` is validated at startup (`config/env.ts:8`) to be at least 16 characters, and `env.JWT_EXPIRES_IN` defaults to `"8h"` if not set in the environment (config/env.ts:9). The resulting token, expiry string, and a hardcoded token type of `'Bearer'` are packaged into an `AuthTokens` object and returned as part of `LoginResult`, which the controller serializes directly into the HTTP response body (`auth.controller.ts:24`).

Embedding `role` directly in the JWT claims means role-based authorization decisions made downstream (e.g., by the `requireRole` middleware in `auth.middleware.ts:49-61`, used by other modules such as orders/products per the codebase's broader routing) trust the token's claims without re-querying the database on every request. This is a standard stateless-JWT trade-off: it improves performance (no DB round-trip needed to check role on protected routes) but means a role change (e.g., an admin demoting a user) does not take effect until the previously issued token expires or is otherwise invalidated — there is no token revocation/blacklist mechanism visible in the analyzed files.

The `expiresIn` value returned to the client in the response body (`env.JWT_EXPIRES_IN`, e.g. `"8h"`) is the same string configured for signing, giving the client a human-readable indication of validity duration without needing to decode the JWT to inspect the `exp` claim.

**Rule workflow:**
```
AuthService.signToken(userId, email, role):
  payload = { sub: userId, email, role }
  options = { expiresIn: env.JWT_EXPIRES_IN }
  token = jwt.sign(payload, env.JWT_SECRET, options)
  return token

LoginResult = {
  user: PublicUser,
  tokens: { accessToken: token, expiresIn: env.JWT_EXPIRES_IN, tokenType: 'Bearer' }
}
```

---

#### Business Rule: Bearer Token Authentication Enforcement on Protected Routes

**Overview:**
The `GET /auth/me` endpoint (and any other route wired with the `authenticate` middleware) requires a valid, non-expired JWT supplied via the `Authorization: Bearer <token>` HTTP header; this check happens entirely at the middleware layer, before the controller executes.

**Detailed description:**
The `authenticate` middleware (`src/middlewares/auth.middleware.ts:27-47`) is applied directly in the route definition for `GET /me` (`auth.routes.ts:12`), not inside `AuthController`. It first checks for the presence of an `Authorization` header and verifies it starts with `"bearer "` in a case-insensitive manner (`header.toLowerCase().startsWith('bearer ')`); if missing or malformed, it throws `UnauthorizedError('Missing or invalid Authorization header')`. It then extracts the token substring (everything after the 7-character `"Bearer "` prefix, trimmed of whitespace) and rejects empty tokens with `UnauthorizedError('Missing bearer token')`. Finally, it calls `jwt.verify(token, env.JWT_SECRET)`, which validates both the signature and the expiry (`exp` claim) of the token; any verification failure (invalid signature, malformed token, expired token) is caught and converted into a generic `UnauthorizedError('Invalid or expired token')`, deliberately not distinguishing between "expired" and "invalid" to avoid leaking token-forgery diagnostic information to an attacker.

On success, the middleware attaches a normalized `AuthUser` object (`{ id, email, role }`, sourced from the JWT's `sub`, `email`, and `role` claims respectively) to `req.user`, using a TypeScript module augmentation of `express-serve-static-core`'s `Request` interface (auth.middleware.ts:12-17) to make `req.user` and `req.id` type-safe across the codebase. `AuthController.me` then performs a defensive `if (!req.user) throw new UnauthorizedError()` check (auth.controller.ts:32) — this is logically unreachable in the current wiring (since `authenticate` always populates `req.user` on the only path that calls `next()` without an error), but serves as a safety net against future refactors that might reorder middleware or reuse the controller method on a route not protected by `authenticate`.

Notably, `req.user.id` (the JWT's `sub` claim) is used to re-fetch the user from the database in `UserService.getById` rather than trusting the JWT's embedded `email`/`role` claims directly for the response. This means `GET /auth/me` always reflects the current database state of the user (e.g., if the name changed after token issuance, the response reflects the new name), except for authorization decisions elsewhere in the system that rely on the token's `role` claim, which could be stale relative to the DB until the token expires.

**Rule workflow:**
```
authenticate middleware:
  header = req.headers.authorization
  if !header || !header.toLowerCase().startsWith('bearer ') -> 401 Missing or invalid Authorization header
  token = header.slice(7).trim()
  if !token -> 401 Missing bearer token
  try: payload = jwt.verify(token, env.JWT_SECRET)
       req.user = { id: payload.sub, email: payload.email, role: payload.role }
       next()
  catch -> 401 Invalid or expired token

AuthController.me:
  if !req.user -> 401 (defensive, unreachable given middleware guarantee)
  user = userService.getById(req.user.id)  [re-fetches fresh state from DB, 404 if deleted]
  200 -> PublicUser
```

---

#### Business Rule: Role Assignment Without Authorization Gate (Security Gap)

**Overview:**
Any unauthenticated caller can register a new account with `role: "ADMIN"` directly through the public registration endpoint, since no authorization check restricts who may set the `role` field.

**Detailed description:**
The `registerSchema` (auth.schemas.ts:7) accepts `role: z.enum(['ADMIN', 'OPERATOR']).default('OPERATOR')` as a client-supplied field on the public, unauthenticated `POST /auth/register` endpoint. There is no check anywhere in `AuthController.register`, `AuthService.register`, or `UserService.createUser` that restricts the `role` value based on the caller's own privileges (which is expected, since registration is by definition performed by unauthenticated users) or that requires an existing `ADMIN` to approve/assign elevated roles. This means the very first API call an anonymous client makes can create an `ADMIN`-role account, which subsequently receives a JWT embedding `role: "ADMIN"` on login, granting access to any route gated by `requireRole('ADMIN')` elsewhere in the system (e.g., in other modules such as products/orders, based on the presence of the generic `requireRole` middleware).

This is architecturally consistent with a system that has no separate "admin invitation" or "seed admin" workflow — the codebase as reviewed provides no alternative privileged-registration path, no admin-approval queue, and no environment-level restriction (e.g., disabling the `role` field in production). The `tests/auth.test.ts` suite does not include a test asserting that `role: 'ADMIN'` is rejected or restricted, and the one register test that does supply `role: 'OPERATOR'` explicitly (test at auth.test.ts:11) simply demonstrates the value is honored, not restricted.

This is flagged as a **High risk** finding in Section 10 (Technical Debt & Risks) rather than as an intentional design decision, because self-service privilege escalation on a system managing orders/inventory/customers is a significant security concern; however, it is presented here strictly as an observed rule/behavior of the code, not as a recommendation for change, per this analysis's read-only scope.

**Rule workflow:**
```
POST /auth/register { email, password, name, role: "ADMIN" }
  -> validate middleware accepts role="ADMIN" (schema permits it unconditionally)
  -> AuthService.register -> UserService.createUser(input) [role passed through untouched]
  -> new user persisted with role=ADMIN
  -> subsequent login issues JWT with role:"ADMIN" claim
  -> grants access to any requireRole('ADMIN')-gated route in the system
```

---

## 4. Component Structure

```
src/modules/auth/
├── auth.controller.ts   # HTTP handlers: register, login, me — thin, delegates to services, uniform try/catch -> next(err)
├── auth.routes.ts       # Express Router wiring: binds validate/authenticate middleware to controller methods
├── auth.schemas.ts      # Zod schemas: registerSchema, loginSchema (+ inferred TS types RegisterInput/LoginInput)
└── auth.service.ts      # AuthService: register() delegates to UserService; login() verifies credentials + signs JWT

One-hop dependencies referenced (outside auth module boundary):
src/modules/users/
├── user.service.ts      # UserService: createUser (hashing + duplicate check), getById, static toPublic()
└── user.repository.ts   # UserRepository: Prisma-backed findById/findByEmail/create

src/middlewares/
├── auth.middleware.ts       # authenticate (JWT verification, req.user population), requireRole (RBAC, unused by auth module)
├── validate.middleware.ts   # Generic Zod-based request validator (body/query/params)
└── error.middleware.ts      # Central error handler: AppError, ZodError, Prisma error mapping -> JSON envelope

src/shared/errors/
├── app-error.ts          # Base AppError class (statusCode, errorCode, details)
└── http-errors.ts        # UnauthorizedError, ValidationError, ConflictError, NotFoundError, etc.

src/config/
└── env.ts                # Zod-validated environment config: JWT_SECRET, JWT_EXPIRES_IN, DATABASE_URL, etc.

src/app.ts                 # Composition root: instantiates AuthService/AuthController with concrete UserRepository/UserService
src/routes/index.ts        # Mounts buildAuthRouter(controllers.auth) under /api/v1/auth
```

---

## 5. Dependency Analysis

```
Internal Dependencies (within analyzed boundary + one hop):

AuthController
  -> AuthService (constructor-injected)          [src/modules/auth/auth.service.ts]
  -> UserService (constructor-injected)           [src/modules/users/user.service.ts]
  -> UnauthorizedError (shared/errors)             [src/shared/errors/index.ts -> http-errors.ts]

AuthService
  -> UserRepository (constructor-injected)         [src/modules/users/user.repository.ts]
  -> UserService (constructor-injected, static toPublic() call only)
  -> bcrypt (compare)                               [external]
  -> jsonwebtoken (sign)                            [external]
  -> env (config)                                   [src/config/env.ts]
  -> UnauthorizedError (shared/errors)

auth.routes.ts (buildAuthRouter)
  -> validate middleware                            [src/middlewares/validate.middleware.ts]
  -> authenticate middleware                        [src/middlewares/auth.middleware.ts]
  -> auth.schemas.ts (registerSchema, loginSchema)
  -> AuthController (type-only import)

auth.middleware.ts (authenticate, requireRole)
  -> jsonwebtoken (verify)                          [external]
  -> env (config)
  -> UnauthorizedError, ForbiddenError (shared/errors)

UserService (one hop)
  -> UserRepository
  -> bcrypt (hash)
  -> ConflictError, NotFoundError (shared/errors)

UserRepository (one hop)
  -> PrismaClient (@prisma/client) -> User model

Composition root (src/app.ts)
  -> instantiates: UserRepository -> UserService -> AuthService -> AuthController
  -> AuthController wired into buildApiRouter via buildAuthRouter (src/routes/index.ts:24)

External Dependencies:
- bcrypt (^5.1.1)          - Password hashing/verification (used in UserService, AuthService)
- jsonwebtoken (^9.0.2)    - JWT signing (AuthService) and verification (auth.middleware.ts)
- zod (^3.23.8)            - Schema validation (auth.schemas.ts, validate.middleware.ts, env.ts)
- express (^4.21.1)        - HTTP routing/middleware framework (RequestHandler, Router)
- @prisma/client (5.22.0)  - ORM/data access, indirectly via UserRepository (one hop, not called directly by AuthController)
- PostgreSQL (via Prisma)  - Persistent storage for User records (implied by prisma/schema.prisma)
```

---

## 6. Afferent and Efferent Coupling

Coupling analyzed at the class/module level (TypeScript classes and function-exporting modules) within the auth module and its immediate one-hop neighbors.

| Component | Afferent Coupling (Ca) | Efferent Coupling (Ce) | Critical |
|-----------|------------------------|-------------------------|----------|
| AuthController | 2 (app.ts, routes/index.ts type import) | 3 (AuthService, UserService, UnauthorizedError) | Low |
| AuthService | 1 (app.ts, wires into AuthController) | 6 (UserRepository, UserService, bcrypt, jsonwebtoken, env, UnauthorizedError) | Medium |
| buildAuthRouter (auth.routes.ts) | 1 (routes/index.ts) | 5 (validate, authenticate, registerSchema, loginSchema, AuthController type) | Low |
| registerSchema / loginSchema (auth.schemas.ts) | 2 (auth.routes.ts, auth.service.ts as type source) | 1 (zod) | Low |
| authenticate middleware | 1 (auth.routes.ts, also reused by other modules' routers per grep evidence in broader codebase) | 3 (jsonwebtoken, env, UnauthorizedError) | High |
| UserService (external, one hop) | 2 (AuthController, AuthService) | 3 (UserRepository, bcrypt, ConflictError/NotFoundError) | High |
| UserRepository (external, one hop) | 2 (AuthService, UserService) | 1 (PrismaClient) | Medium |

Notes:
- `AuthController` has low afferent coupling (only the composition root and route index reference it) and moderate efferent coupling (3 direct dependencies), consistent with a thin controller.
- `authenticate` middleware is marked **High** criticality despite modest coupling counts because it is the sole gatekeeper for all authenticated routes across the entire API (not just `/auth/me` — it is presumed reused by other protected routers per the codebase's module structure), making it a single point of failure for the system's authorization boundary.
- `UserService` is marked **High** criticality because both `AuthController` (directly) and `AuthService` (transitively, since `AuthService.register` is a pure passthrough) depend on it for core account logic (hashing, duplicate detection, profile retrieval).

---

## 7. Endpoints

| Endpoint | Method | Description | Auth Required | Request Body Schema | Success Status |
|----------|--------|--------------|----------------|----------------------|-----------------|
| /api/v1/auth/register | POST | Create a new user account (email, password, name, optional role) | No | registerSchema | 201 |
| /api/v1/auth/login | POST | Authenticate with email/password and receive a JWT access token | No | loginSchema | 200 |
| /api/v1/auth/me | GET | Retrieve the currently authenticated user's public profile | Yes (Bearer JWT via `authenticate` middleware) | None | 200 |

---

## 8. Integration Points

| Integration | Type | Purpose | Protocol | Data Format | Error Handling |
|-------------|------|---------|----------|--------------|-----------------|
| UserService / UserRepository | Internal Module | Account creation, duplicate-email check, profile lookup | In-process method calls | TypeScript objects / Prisma models | Domain errors (ConflictError, NotFoundError) thrown and propagated via next(err) |
| PostgreSQL (via Prisma) | Database | Persisting and querying User records | Prisma Client (SQL under the hood) | Prisma-generated User model | Prisma error codes (P2002, P2025) mapped centrally in error.middleware.ts, not in auth module itself |
| jsonwebtoken library | External Library | Sign JWTs on login; verify JWTs on protected-route access | In-process function call (jwt.sign / jwt.verify) | JWT (JSON payload, HMAC-signed) | try/catch around jwt.verify collapses all failures to UnauthorizedError; jwt.sign failures are not explicitly caught (would surface as unhandled 500 via errorMiddleware fallback) |
| bcrypt library | External Library | Hash passwords on registration; compare on login | In-process async function call | Bcrypt hash string ($2b$... format) | Not explicitly caught in auth module; failures would propagate as unhandled errors to errorMiddleware's generic 500 path |
| env (config/env.ts) | Internal Config Module | Supplies JWT_SECRET and JWT_EXPIRES_IN | Direct import, validated at process startup via Zod | Typed Env object | Fails fast at startup (process.exit(1)) if JWT_SECRET missing/too short; no runtime error handling needed thereafter |
| Express error pipeline (next/errorMiddleware) | Internal Framework Integration | Centralizes all error-to-HTTP-response translation | In-process, Express next(err) convention | JSON error envelope { error: { code, message, details? } } | Single centralized handler (error.middleware.ts) distinguishes AppError, ZodError, and Prisma errors; unknown errors logged and returned as generic 500 |

---

## 9. Design Patterns & Architecture

| Pattern | Implementation | Location | Purpose |
|---------|-----------------|-----------|---------|
| Thin Controller / Controller-Service separation | AuthController delegates all logic to AuthService/UserService | auth.controller.ts:12-38 | Keeps HTTP concerns (status codes, req/res) separate from business logic |
| Dependency Injection (constructor injection) | AuthController(authService, userService); AuthService(users, userService) | auth.controller.ts:7-10, auth.service.ts:22-25 | Testability, inversion of control; dependencies wired centrally in app.ts |
| Repository Pattern | UserRepository abstracts Prisma access behind findById/findByEmail/create | user.repository.ts | Data access abstraction, decouples services from ORM specifics |
| Middleware Chain / Pipeline | validate -> (authenticate) -> controller handler, per-route composition | auth.routes.ts:10-12 | Cross-cutting concerns (validation, auth) applied declaratively at the routing layer, outside controller code |
| Centralized Error Handling | try/catch { next(err) } in every controller method; single errorMiddleware downstream | auth.controller.ts (all 3 methods), error.middleware.ts | Consistent error-to-HTTP-response mapping; controllers stay free of response-formatting logic for errors |
| Factory Function for Router Construction | buildAuthRouter(controller) returns a configured Router | auth.routes.ts:7-15 | Decouples router construction from controller instantiation; supports explicit composition root wiring |
| Data Transfer / View Model shaping | PublicUser type + UserService.toPublic() static method | user.service.ts:7-14, 42-51 | Prevents sensitive fields (passwordHash) from leaking into API responses |
| Custom Exception Hierarchy | AppError base class extended by UnauthorizedError, ConflictError, ValidationError, etc. | shared/errors/app-error.ts, http-errors.ts | Encodes HTTP status + machine-readable error code + optional details per domain error type |
| Schema-Driven Validation (Zod) | registerSchema / loginSchema define both runtime validation and static TS types via z.infer | auth.schemas.ts | Single source of truth for input shape at both compile-time and runtime |
| Type Augmentation (Declaration Merging) | Request.user / Request.id added via `declare module 'express-serve-static-core'` | auth.middleware.ts:12-17 | Type-safe access to auth context attached by middleware, without a custom Request subclass |
| Stateless Authentication (JWT) | No server-side session store; all auth state carried in the signed token | auth.service.ts, auth.middleware.ts | Enables horizontal scalability without shared session storage; trade-off is no built-in revocation |

---

## 10. Technical Debt & Risks

| Risk Level | Component Area | Issue | Impact |
|------------|------------------|-------|--------|
| High | auth.schemas.ts:7 / auth.service.ts:27-29 | `role` field is fully client-controlled on public, unauthenticated `/auth/register` endpoint with no restriction preventing self-assignment of `ADMIN` | Privilege escalation: any anonymous caller can create an admin account and gain access to any `requireRole('ADMIN')`-gated functionality elsewhere in the system |
| High | JWT design (auth.service.ts, auth.middleware.ts) | No token revocation/blacklist mechanism; role/permissions embedded in JWT claims are trusted until natural expiry (default 8h) | A demoted/deactivated user retains their old privileges for up to the full token lifetime after the change is made server-side |
| Medium | user.service.ts:16 (BCRYPT_ROUNDS = 10) | Bcrypt cost factor is a hardcoded constant, not environment-configurable | Cannot be tuned per-environment (e.g., higher rounds in production, lower in test) without a code change and redeploy |
| Medium | auth.service.ts:31-39 (timing side-channel) | "User not found" path returns before the (slow) bcrypt.compare call, while "wrong password" path always incurs it | Theoretical timing-based user-enumeration vector, despite identical error messages/codes |
| Medium | auth.controller.ts:32 (`me` handler) | Defensive `if (!req.user) throw new UnauthorizedError()` check is currently unreachable given the route's middleware guarantees `req.user` is set before the controller runs | Dead code path today; also a latent risk if the route is ever re-wired without the `authenticate` middleware, since the controller alone provides no real protection |
| Medium | auth.service.ts (jwt.sign) / user.service.ts (bcrypt.hash) | No explicit try/catch around bcrypt/jsonwebtoken calls within the auth module; failures fall through to the generic 500 handler in errorMiddleware | Operational errors (e.g., JWT_SECRET misconfiguration surfacing at sign-time rather than startup, though env.ts validates this) would produce an opaque 500 rather than an actionable error |
| Low | auth.service.ts:31 (LoginInput email lookup) | Email matching relies on exact string match via Prisma; no explicit normalization (lowercasing/trimming) visible in the auth module before the DB lookup | Users could unintentionally register/login with what appears to be the same email but differs in case, unless normalization happens elsewhere (e.g., DB collation) not visible in the reviewed files |
| Low | error.middleware.ts vs user.service.ts (duplicate-email handling) | Two distinct error codes can represent the same underlying situation: app-level check yields `EMAIL_ALREADY_USED` (409), while a Prisma P2002 race-condition fallback yields generic `CONFLICT` (409) | Client-side error handling must account for two different `error.code` values representing effectively the same business condition |
| Low | requireRole middleware (auth.middleware.ts:49-61) | Exported and available but not used anywhere within the auth module's own routes | Not a defect per se, but indicates the auth module's own routes (`/auth/me`) do not currently need role differentiation; worth noting as available-but-unused capability at this component boundary |

---

## 11. Test Coverage Analysis

| Component | Unit Tests | Integration Tests | Coverage (endpoint-level) | Test Quality |
|-----------|------------|---------------------|------------------------------|----------------|
| AuthController.register | 0 | 3 (tests/auth.test.ts:6-39) | Happy path, duplicate email, invalid payload | Good: verifies status code, response shape, absence of sensitive fields, and error codes. No dedicated test for role self-assignment (ADMIN) behavior |
| AuthController.login | 0 | 2 (tests/auth.test.ts:41-63) | Happy path, wrong password | Good assertions on token shape and error code; missing an explicit "unknown email" case (relies on symmetry with wrong-password case, not independently verified) |
| AuthController.me | 0 | 2 (tests/auth.test.ts:65-83) | Authenticated success, missing token | Good: verifies profile fields returned and 401 on missing token. Missing explicit tests for expired token and malformed/non-Bearer Authorization header (both handled distinctly in auth.middleware.ts:29-38 but not exercised) |
| AuthService | 0 | Indirectly covered via AuthController integration tests (no isolated unit tests found) | N/A (no direct unit test file for auth.service.ts located) | All behavior verified only through HTTP-level integration tests in tests/auth.test.ts; no mocked-dependency unit tests isolating AuthService logic |
| authenticate middleware | 0 | Indirectly covered via tests/auth.test.ts (`/me` tests) and tests/orders.test.ts (extensive reuse via bootstrapAuthenticatedUser) | Bearer-token happy path and missing-token path covered; malformed header and expired-token paths not explicitly tested | Reused heavily as a dependency of other modules' tests (tests/orders.test.ts uses `bootstrapAuthenticatedUser` at least 8 times), giving it broad indirect exercise, though always via the "valid token" path |
| validate middleware (via registerSchema/loginSchema) | 0 | 1 (tests/auth.test.ts:30-39, invalid register payload) | Register invalid-payload case covered; no dedicated invalid-payload test for login | Verifies VALIDATION_ERROR code and details array structure |

**Test file locations:**
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/auth.test.ts` — primary, direct integration tests for all three AuthController endpoints (6 test cases total).
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/helpers/factories.ts` — shared test infrastructure: `getTestApp()` (builds the real Express app via `buildApp`), `createTestUser()` (direct Prisma insert bypassing the API, with bcrypt-hashed password), `loginAndGetToken()` (calls the real `/auth/login` endpoint), and `bootstrapAuthenticatedUser()` (combines the two, used extensively by other test suites).
- `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/tests/orders.test.ts` — does not test AuthController directly, but exercises the `login` endpoint and `authenticate` middleware indirectly and repeatedly (at least 8 call sites) through `bootstrapAuthenticatedUser()`, providing broad regression coverage for the "valid credentials -> valid token -> authenticated request succeeds" path as a side effect of testing the orders module.

**Coverage gaps identified (descriptive, not prescriptive):**
- No isolated/mocked unit tests exist for `AuthService` or `AuthController` classes in isolation from the real database and HTTP stack; all tests are full-stack integration tests via `supertest` against a real (test) database through Prisma.
- No test exercises registering a user with `role: 'ADMIN'` to confirm/deny the self-escalation behavior documented in Section 3 and Section 10.
- No test exercises an expired JWT or a malformed (non-Bearer-prefixed) Authorization header against `/auth/me`, despite `auth.middleware.ts` having distinct code branches for these cases.
- No test independently verifies the "unknown email" login-failure path in isolation from the "wrong password" path.

---
