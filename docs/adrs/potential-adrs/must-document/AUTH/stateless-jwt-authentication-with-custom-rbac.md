# Potential ADR: Stateless JWT Authentication with Custom RBAC (Build vs. Buy Identity)

**Module**: AUTH
**Category**: Security / Architecture
**Priority**: Must Document (Score: 145/150)
**Date Identified**: 2026-07-01

---

## What Was Identified

The system implements its own authentication and authorization layer rather than integrating a third-party identity provider (e.g., Auth0, Okta, AWS Cognito) or using a session-based approach. `AuthService.login()` verifies credentials with `bcrypt.compare()` against a `passwordHash` stored on the `User` model, then issues a self-signed, stateless JWT (`jsonwebtoken`) containing `sub` (user id), `email`, and `role` claims, signed with `env.JWT_SECRET` and a configurable expiry (`env.JWT_EXPIRES_IN`). Every protected route is guarded by two composable Express middlewares: `authenticate` (verifies the Bearer token and attaches `req.user`) and `requireRole(...roles)` (an RBAC guard checking `req.user.role` against an allow-list, e.g., `requireRole('ADMIN')` on admin-only endpoints). There are exactly two roles, `ADMIN` and `OPERATOR`, encoded directly in the JWT payload rather than looked up per-request from the database.

This is a foundational, domain-critical decision for a user-facing REST API: it determines how every one of the ~25+ protected endpoints across five feature modules (USERS, CUSTOMERS, PRODUCTS, ORDERS, and AUTH itself) authenticates and authorizes requests. No external identity provider, session store, or refresh-token mechanism is present — the JWT itself is the sole source of truth for identity and role for the lifetime of the token.

The repository has a single "init repository" commit (2026-06-24) introducing this pattern fully formed, with no subsequent iteration — indicating it was adopted as the initial architectural baseline for the project rather than evolved incrementally. Git history offers no additional temporal signal beyond this initial commit (no follow-up commits touching `auth.service.ts` or `auth.middleware.ts`), consistent with this being a young, single-iteration codebase built for a design-docs exercise.

## Why This Might Deserve an ADR

- **Impact**: Every protected endpoint in the system (all feature modules except public health checks) depends on `authenticate`/`requireRole`. Changing the identity model (e.g., adopting an external IdP, moving to session cookies, adding refresh tokens) would touch middleware, client contracts, and potentially the token payload/claims shape consumed across all controllers.
- **Trade-offs**:
  - Stateless JWTs avoid a session store/lookup on every request (simpler ops, no Redis/session infra) but make token revocation impossible before expiry — there is no logout/blacklist mechanism visible in the code.
  - Roles are embedded in the JWT at issuance time; a role change for a user only takes effect after the existing token expires and a new one is issued, since `authenticate` never re-reads role from the database.
  - Building auth in-house (vs. Auth0/Cognito/etc.) avoids third-party cost and lock-in but means the team owns password storage, token security, and RBAC correctness end-to-end.
- **Complexity**: Low implementation complexity today (bcrypt + jsonwebtoken + two middlewares, ~150 LOC total), but the *decision surface* (build vs. buy, stateless vs. stateful, two-role model) has long-term implications disproportionate to its current code size.
- **Team Knowledge**: Any engineer adding a new protected route, a new role, or reasoning about session/logout/security behavior must understand this model — it's not optional context.
- **Future Implications**: If the product needs more than two roles, per-resource permissions, token revocation, multi-tenant isolation, or SSO (all plausible for an OMS that may onboard more staff/customers), this foundational choice will need to be revisited; understanding why it was built this way originally will matter for that migration.
- **Temporal Context**: Introduced as-is in the project's single initial commit; no iteration since, meaning there is no commit history to infer follow-up refinement or lessons learned. Effort to change today (early in the app's life) is a fraction of what it will be once more modules/clients depend on the current token shape.

## Evidence Found in Codebase

### Key Files
- [`src/modules/auth/auth.service.ts`](../../../../../src/modules/auth/auth.service.ts) - Lines 20-49: login logic (bcrypt compare) and JWT signing (`signToken`), token payload shape (`sub`, `email`, `role`)
- [`src/middlewares/auth.middleware.ts`](../../../../../src/middlewares/auth.middleware.ts) - Lines 27-58: `authenticate` (Bearer token verification) and `requireRole` (RBAC guard), applied per-router across the app
- [`src/modules/auth/auth.schemas.ts`](../../../../../src/modules/auth/auth.schemas.ts) - Lines 3-14: registration/login input validation, two-role enum (`ADMIN`, `OPERATOR`)
- [`src/config/env.ts`](../../../../../src/config/env.ts) - `JWT_SECRET`, `JWT_EXPIRES_IN` — the only configurable knobs of the auth strategy
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma) - `User` model with `passwordHash`, `UserRole` enum — the persisted identity backing the JWT claims

### Code Evidence
```typescript
// src/modules/auth/auth.service.ts:31-45
async login(input: LoginInput): Promise<LoginResult> {
  const user = await this.users.findByEmail(input.email);
  if (!user) {
    throw new UnauthorizedError('Invalid credentials');
  }
  const ok = await bcrypt.compare(input.password, user.passwordHash);
  if (!ok) {
    throw new UnauthorizedError('Invalid credentials');
  }
  const token = this.signToken(user.id, user.email, user.role);
  return {
    user: UserService.toPublic(user),
    tokens: { accessToken: token, expiresIn: env.JWT_EXPIRES_IN, tokenType: 'Bearer' },
  };
}
```

```typescript
// src/middlewares/auth.middleware.ts:27-58
export const authenticate: RequestHandler = (req, _res, next) => {
  // ... verifies Bearer header, jwt.verify(token, env.JWT_SECRET) ...
};

export function requireRole(...roles: AuthUser['role'][]): RequestHandler {
  return (req, _res, next) => {
    if (!req.user) { next(new UnauthorizedError()); return; }
    if (!roles.includes(req.user.role)) { next(new ForbiddenError('Insufficient permissions')); return; }
    next();
  };
}
```

### Impact Analysis
- Introduced: 2026-06-24 ("init repository") — the pattern appeared fully formed in the project's initial commit, no incremental evolution visible.
- Modified: 0 follow-up commits touching `auth.service.ts` or `auth.middleware.ts` since init.
- Affects: `auth.middleware.ts` is imported and applied across routers in every feature module (USERS, CUSTOMERS, PRODUCTS, ORDERS) in addition to AUTH itself — effectively a cross-cutting concern.
- Recent themes: N/A (no commit history beyond initial creation; project is very young, ~1 week old as of this analysis).

### Alternatives (if observable)
No alternatives are explicitly discussed in code comments, config toggles, or commit messages. The mapping notes confirm: "External: none (no external identity provider)" — the choice to build in-house rather than integrate an IdP is implicit in the absence of any OAuth2/OIDC client library or provider SDK in `package.json`.

## Questions to Address in ADR (if created)

- Why was a custom JWT + bcrypt implementation chosen over an external identity provider (Auth0, Cognito, Keycloak) or a session-based approach?
- Why only two roles (`ADMIN`, `OPERATOR`) embedded directly in the token rather than a more granular permission model or database-backed role lookup?
- What is the intended strategy for token revocation/logout, given tokens are stateless and cannot currently be invalidated before expiry?
- What is the expected `JWT_EXPIRES_IN` value in production, and what's the plan if a user's role needs to change before token expiry?
- Under what conditions (scale, compliance, SSO requirements) would this in-house approach be revisited in favor of a managed identity provider?

## Related Potential ADRs
- None yet identified in other modules (USERS module consumes `UserRepository`/`UserService` for registration but does not introduce a separate architectural decision beyond what's captured here).

## Additional Notes
- RBAC (`requireRole`) and JWT issuance/verification are treated as a single cohesive decision here (per Red Flag 5 guidance) rather than split into separate ADRs, since they form one integrated "authentication + authorization strategy" that was adopted together.
- Password hashing via `bcrypt` (5.1) is noted as supporting evidence within this decision rather than broken out separately — it is a standard, low-controversy implementation detail of the broader "how do we authenticate users" decision.
- No existing ADRs were found in `docs/adrs/generated/` (directory does not exist yet), so no duplicate/related-ADR context applies.
