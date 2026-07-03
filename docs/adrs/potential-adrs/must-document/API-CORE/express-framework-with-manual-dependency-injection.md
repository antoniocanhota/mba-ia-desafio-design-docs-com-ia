# Potential ADR: Express.js as Primary Web Framework with Manual Dependency Injection (Composition Root)

**Module**: API-CORE
**Category**: Architecture / Technology
**Priority**: Must Document (Score: 145/150)
**Date Identified**: 2026-07-01

---

## What Was Identified

The entire HTTP layer of the Order Management API is built directly on Express 4, with no additional web meta-framework (no NestJS, no Fastify) layered on top. `src/app.ts` constructs the `Express` instance, applies a small fixed set of global middleware (JSON body parsing, request logging, centralized error handling), mounts a single versioned router (`/api/v1`) that aggregates all five feature-module routers, and exposes an unauthenticated `/health` endpoint. `src/server.ts` is the sole process entrypoint that instantiates the shared `PrismaClient`, calls `buildApp`, and manages graceful shutdown on `SIGINT`/`SIGTERM`.

Tightly coupled to the framework choice is an explicit architectural decision to avoid an inversion-of-control (IoC) container entirely. `buildControllers()` in `src/app.ts` manually constructs the full dependency graph for every feature module — repository → service → controller — via plain constructor calls (e.g. `new UserRepository(prisma)`, `new UserService(userRepository)`, `new UserController(userService)`), for all five modules (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS). This "poor man's DI" / composition-root pattern is the single place in the codebase where the whole object graph is wired together, and every new module must be registered here and in `src/routes/index.ts` to be reachable.

Git history shows this structure was established in full in the single initial "init repository" commit (2026-06-24) and has not been modified since — the whole application skeleton (bootstrap, routing aggregation, manual DI wiring, and centralized error handling) was scaffolded together as one atomic architectural choice, rather than evolving incrementally. As of this analysis (2026-07-01) the repository is about one week old with only 8 total commits, so this pattern has not yet been tested by iteration or by adding a 6th module — but it is already the backbone every one of the 5 existing feature modules depends on.

## Why This Might Deserve an ADR

- **Impact**: Every feature module (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS) is wired through `buildControllers()`/`buildApiRouter()`; changing the composition strategy (e.g. introducing an IoC container, or migrating off Express to a meta-framework like NestJS) would require touching every module's construction site and routing registration simultaneously.
- **Trade-offs**: Manual constructor injection keeps the stack simple, dependency-free, and easy to trace (no "magic" container, no decorators/reflection), at the cost of boilerplate that scales linearly with each new module and repository/service/controller triple — there is no automatic wiring, so onboarding a 6th module means manually editing `buildControllers()` and `buildApiRouter()`.
- **Complexity**: Low framework complexity today (bare Express, no ORM-integration magic, no DI reflection), but this is itself a considered trade-off against frameworks like NestJS that bundle routing + DI + validation conventions.
- **Team Knowledge**: Any engineer adding a new module, endpoint, or cross-cutting concern (auth, logging, error handling) needs to understand this exact bootstrap/wiring convention — it is the "front door" of the entire codebase and the only place the object graph is assembled.
- **Future Implications**: As the module count grows, the manual wiring in `buildControllers()` will need either continued discipline (documented convention) or a deliberate future migration to a DI container/framework — a decision best made explicitly and early given the cost of retrofitting IoC once dozens of modules exist.
- **Temporal Context**: Established atomically in the initial commit (2026-06-24) and unchanged through 8 subsequent commits (as of 2026-07-01, ~1 week later); the pattern has not yet been stress-tested by growth, but is already the foundation for 100% of existing routes.

## Evidence Found in Codebase

### Key Files
- [`src/app.ts`](../../../../../src/app.ts) - Lines 26-76
  - `buildControllers()`: manual constructor-based wiring for all 5 feature modules
  - `buildApp()`: Express app assembly — global middleware, health check, versioned router mount, 404 handler, centralized error middleware
- [`src/server.ts`](../../../../../src/server.ts) - Lines 1-27
  - Sole process entrypoint; instantiates shared `PrismaClient`, calls `buildApp`, manages graceful shutdown
- [`src/routes/index.ts`](../../../../../src/routes/index.ts) - Lines 13-31
  - `buildApiRouter()`: aggregates all module routers under `/api/v1`, requires each new module to be explicitly registered

### Code Evidence
```typescript
// src/app.ts:26-53
export function buildControllers(prisma: PrismaClient): Controllers {
  const userRepository = new UserRepository(prisma);
  const userService = new UserService(userRepository);
  const userController = new UserController(userService);

  const authService = new AuthService(userRepository, userService);
  const authController = new AuthController(authService, userService);

  const customerRepository = new CustomerRepository(prisma);
  const customerService = new CustomerService(customerRepository);
  const customerController = new CustomerController(customerService);

  const productRepository = new ProductRepository(prisma);
  const productService = new ProductService(productRepository);
  const productController = new ProductController(productService);

  const orderRepository = new OrderRepository(prisma);
  const orderService = new OrderService(orderRepository, prisma);
  const orderController = new OrderController(orderService);

  return {
    auth: authController,
    users: userController,
    customers: customerController,
    products: productController,
    orders: orderController,
  };
}
```

```typescript
// src/app.ts:55-76
export function buildApp(deps: AppDependencies): Express {
  const app = express();

  app.disable('x-powered-by');
  app.use(express.json({ limit: '1mb' }));
  app.use(requestLogger);

  app.get('/health', (_req, res) => {
    res.status(200).json({ status: 'ok' });
  });

  const controllers = buildControllers(deps.prisma);
  app.use('/api/v1', buildApiRouter(controllers));

  app.use((req, _res, next) => {
    next(new NotFoundError(`Route ${req.method} ${req.originalUrl}`));
  });

  app.use(errorMiddleware);

  return app;
}
```

### Impact Analysis
- Introduced: 2026-06-24 ("init repository" — scaffolded as a complete, atomic commit rather than incrementally)
- Modified: 0 follow-up commits to `src/app.ts`, `src/server.ts`, or `src/routes/index.ts` since introduction (8 total commits in repo, remainder are docs/README changes)
- Last change: 2026-06-24 (no changes since)
- Affects: All 5 feature modules (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS), 100% of HTTP routes, every request passes through this bootstrap
- Recent themes: N/A — no iterative commits touching this code; the whole repo's commit history is dominated by documentation changes after the initial scaffold

### Alternatives (if observable)
No explicit alternatives (e.g. NestJS, Fastify, InversifyJS/tsyringe for DI) are referenced in code comments, configuration, or commit messages. The choice of bare Express + manual DI appears to be an implicit default for a small-scope exercise project rather than a documented trade-off analysis — this absence of recorded rationale is itself a reason to capture the decision formally before the codebase grows.

## Questions to Address in ADR (if created)

- Why was bare Express chosen over a more opinionated meta-framework (NestJS, Fastify) that bundles routing, DI, and validation conventions?
- Why was manual constructor injection chosen over an IoC container (InversifyJS, tsyringe, Awilix)? Was this a deliberate simplicity trade-off or just the path of least resistance for a small exercise scope?
- At what module/team-size threshold should this be revisited (e.g. is there a documented trigger, such as "when we reach N modules, adopt a DI container")?
- What conventions must new contributors follow when adding a module (must register in both `buildControllers()` and `buildApiRouter()` — is this documented anywhere outside the code itself)?
- Is the current `/api/v1` versioning strategy (single URL prefix, no per-route version negotiation) sufficient for anticipated API evolution (e.g. webhooks mentioned in PRD but not yet implemented)?

## Related Potential ADRs
None yet — this is the first potential ADR identified for this codebase. Related future candidates (not currently scoring high enough to document) include the centralized `AppError` error-handling hierarchy (`src/shared/errors/*`, `src/middlewares/error.middleware.ts`) and the shared Zod-based request validation middleware — both are tightly coupled to this bootstrap and may warrant reconsideration if this ADR is written and either pattern's scope grows (e.g. if the error contract becomes a versioned public API surface).

## Additional Notes

- This decision was evaluated as **Step 0, Category 2 (Primary Framework/Platform)**, which guarantees inclusion, and the manual-DI/composition-root pattern was folded into the same candidate ADR because both were introduced together in `src/app.ts` in the same commit and are inseparable in explaining "how the application is built and wired" — documenting Express without documenting how modules are composed would be an incomplete picture of the same decision.
- Two adjacent candidates were considered and explicitly **not** promoted to separate potential ADRs due to insufficient score under the strict filtering criteria (kept below the 75-point threshold): the centralized `AppError` error-handling hierarchy, and the shared `pino`/`pino-http` structured logging setup. Both are legitimate cross-cutting patterns but, on their own, did not meet the Scope+Impact/Cost-to-Change/Team-Knowledge bar independent of the framework decision above at the current codebase scale (~30 TS files, 1-week-old repository). Re-evaluate these if the codebase grows significantly or if the API's error contract becomes a versioned/public surface.
- No `docs/adrs/generated/` directory exists yet in this repository, so no existing-ADR similarity/duplicate check was applicable.
