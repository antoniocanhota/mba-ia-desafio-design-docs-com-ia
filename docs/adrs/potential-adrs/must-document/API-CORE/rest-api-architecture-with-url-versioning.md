# Potential ADR: REST API Architecture with URL-Based Versioning (`/api/v1`)

**Module**: API-CORE
**Category**: Architecture (API Protocol/Architecture)
**Priority**: Must Document (Score: 130/150)
**Date Identified**: 2026-07-01

---

## What Was Identified

The service exposes a single, resource-oriented REST API, with every module router mounted under a version-prefixed path (`/api/v1`) in `src/routes/index.ts` and `src/app.ts`. Each of the 5 feature modules follows an identical layered convention: `*.routes.ts` (Express `Router` factory mapping HTTP verbs to resource paths) → `*.controller.ts` (HTTP glue) → `*.service.ts` (business logic) → `*.repository.ts` (Prisma access). Resources use conventional REST verbs and paths (e.g., `GET /orders`, `GET /orders/:id`, `POST /orders`, `PATCH /orders/:id/status`, `DELETE /orders/:id` in `order.routes.ts`), consistent JSON error envelopes (`{ error: { code, message, details } }` from `error.middleware.ts`), and a shared pagination shape (`paginated()` in `src/shared/http/response.ts`) for list endpoints. There is no GraphQL, gRPC, or RPC-style API anywhere in the codebase — REST-over-HTTP/JSON with URL versioning is the sole API paradigm.

This architecture was established in full alongside the rest of the stack in the single "init repository" commit (2026-06-24) and has remained unchanged since: no second API version, no alternate protocol, and no deviation from the `/api/v1` prefix or the routes→controller→service→repository layering across any of the 5 modules that were built on top of it (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS). The consistency across all 5 independently-developed modules (each with its own `*.routes.ts`) signals this was a deliberate, upfront architectural template rather than an incidental pattern.

## Why This Might Deserve an ADR

- **Impact**: Defines the external contract every API consumer (frontend, third-party integrations, the meeting-transcript-derived "webhooks" feature discussed in `docs/PRD.md` but not yet implemented) depends on — URL structure, versioning strategy, and JSON envelope shapes (success and error) are all fixed by this decision.
- **Trade-offs**: REST + URL versioning (`/api/v1`) is simple and cache/tooling-friendly but couples the version to the URL (vs. header-based versioning) and requires either maintaining parallel route trees or careful backward-compatible evolution as the API grows; a GraphQL alternative would have offered client-driven queries and single-endpoint evolution at the cost of added server complexity (schema, resolvers) not currently present anywhere in the stack.
- **Complexity**: Low per-module complexity (identical layered structure repeated 5 times), but the *consistency* of that structure across independently-built modules is itself the architectural asset — any new module (e.g., a future webhooks module) is expected to follow the same pattern.
- **Team Knowledge**: Every engineer adding or modifying an endpoint must understand the routes→controller→service→repository convention, the `/api/v1` prefix, the validation-middleware pattern (`validate.middleware.ts`), and the shared error/pagination envelope shapes — this is foundational for close to 100% of feature work in an API-only service.
- **Future Implications**: Introducing a breaking API change requires deciding how `/api/v2` (or an alternative versioning strategy) coexists with `/api/v1`; adopting a different protocol (e.g., GraphQL for the webhooks feature mentioned in `docs/PRD.md`) would be a major architectural addition, not a natural extension of the current REST-only design.
- **Temporal Context**: Stable and consistently applied since project inception (2026-06-24) across all 5 feature modules, with zero deviation — indicates high confidence/commitment to this pattern as the project's API paradigm.

## Evidence Found in Codebase

### Key Files
- [`src/app.ts`](../../../../../src/app.ts) - Line 67: mounts the aggregated API router under the versioned prefix `/api/v1`
- [`src/routes/index.ts`](../../../../../src/routes/index.ts) - Lines 21-31: resource-oriented sub-router mounting (`/auth`, `/users`, `/customers`, `/products`, `/orders`)
- [`src/modules/orders/order.routes.ts`](../../../../../src/modules/orders/order.routes.ts) - Lines 12-27: representative REST resource router (verbs, resource-nested sub-path `/:id/status`, validation-middleware-per-route)
- [`src/middlewares/error.middleware.ts`](../../../../../src/middlewares/error.middleware.ts) - Lines 17-24, 55-60: consistent `{ error: { code, message, details? } }` JSON envelope enforced across all endpoints regardless of module
- [`src/shared/http/response.ts`](../../../../../src/shared/http/response.ts) - `paginated()` / `PaginatedResponse<T>`: shared list-response contract reused across resource collections

### Code Evidence
```typescript
// src/app.ts:66-67
const controllers = buildControllers(deps.prisma);
app.use('/api/v1', buildApiRouter(controllers));
```

```typescript
// src/routes/index.ts:24-28
router.use('/auth', buildAuthRouter(controllers.auth));
router.use('/users', buildUserRouter(controllers.users));
router.use('/customers', buildCustomerRouter(controllers.customers));
router.use('/products', buildProductRouter(controllers.products));
router.use('/orders', buildOrderRouter(controllers.orders));
```

```typescript
// src/modules/orders/order.routes.ts:16-24 — canonical REST resource pattern repeated across all 5 modules
router.get('/', validate({ query: listOrdersQuerySchema }), controller.list);
router.get('/:id', validate({ params: orderIdParamSchema }), controller.getById);
router.post('/', validate({ body: createOrderSchema }), controller.create);
router.patch(
  '/:id/status',
  validate({ params: orderIdParamSchema, body: updateOrderStatusSchema }),
  controller.changeStatus,
);
router.delete('/:id', validate({ params: orderIdParamSchema }), controller.delete);
```

### Impact Analysis
- Introduced: 2026-06-24 (single "init repository" commit; the layered routes→controller→service→repository pattern and `/api/v1` prefix appear fully formed and consistently across all 5 modules)
- Modified: 0 architectural changes since inception — no second API version, no protocol changes, no deviation from the layering convention in any module
- Affects: All 5 feature modules (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS), `src/routes/index.ts`, `src/app.ts`, and the entire external API contract consumed by any client
- Recent themes: N/A (single-commit history; no thematic commit messages available to draw evolution signal from)

### Alternatives
No explicit alternatives (GraphQL, gRPC, header-based versioning) are referenced in code, comments, or commit messages. `docs/PRD.md` reportedly discusses a planned "webhooks" feature not yet reflected in `src/`, which could represent a future extension or deviation from the pure request/response REST model — worth flagging as a question for the eventual ADR.

## Questions to Address in ADR (if created)

- Why URL-based versioning (`/api/v1`) over alternatives (header-based, no versioning)? What is the plan for introducing `/api/v2` if/when a breaking change is needed?
- Why REST over GraphQL or RPC-style APIs, given the domain has moderately complex nested reads (e.g., orders with items, customer, status history)?
- How will the planned webhooks feature (mentioned in `docs/PRD.md` but not yet implemented) fit into this REST-only architecture — as outbound webhook calls, or will it introduce a new inbound protocol?
- What is the policy for evolving the shared error envelope and pagination contract without breaking existing API consumers?

## Related Potential ADRs
- `express-framework-with-manual-dependency-injection.md` (API-CORE) — the underlying HTTP framework this REST architecture is built on
- `stateless-jwt-authentication-with-custom-rbac.md` (AUTH) — the auth mechanism applied per-router across this REST API
- `mysql-as-primary-relational-database.md` / `prisma-orm-for-data-access.md` (DATA) — the persistence layer each REST resource ultimately reads/writes through

## Additional Notes
The centralized error-handling middleware (`error.middleware.ts`, mapping the `AppError` hierarchy and Prisma/Zod errors to a consistent JSON envelope) was evaluated as a potential standalone ADR but scored below the 75-point threshold in isolation (~65/150: Scope+Impact 25, Cost to Change 20, Team Knowledge 20) and is better understood as a sub-component/implementation detail of this overall REST API contract decision (Red Flag 5 consolidation) rather than a separate architectural decision. If the team later treats the error taxonomy as a deliberately debated, evolving contract (e.g., formal error-code registry, versioned error schemas), it could be reconsidered as its own ADR.
