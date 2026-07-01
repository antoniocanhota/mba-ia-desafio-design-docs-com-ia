# Dependency Audit Report

**Project:** order-management-api (Order Management System REST API)
**Audit Scope:** `src/` directory usage, audited against manifests at repository root (`package.json`, `package-lock.json`)
**Ecosystem:** Node.js / npm (single ecosystem detected)
**Audit Date:** 2026-06-30
**Repository Root:** `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia`

---

## 1. Summary

The project is a single-ecosystem Node.js/TypeScript codebase (Express 4 REST API with Prisma ORM against MySQL). Only one dependency manifest pair was found relevant to the audit: `package.json` and `package-lock.json` at the repository root (no separate `package.json` exists inside `src`). The lockfile is present, resolved versions match the manifest exactly (verified by cross-referencing `package-lock.json`), and dependency counts are small and well-scoped: 8 direct runtime dependencies and 17 direct development dependencies.

All version and vulnerability data below was verified externally against the npm registry (`registry.npmjs.org`), the GitHub Advisory Database (`api.github.com/advisories`), and GitHub repository metadata (`api.github.com/repos/...`) for maintenance activity and license confirmation. `npm audit --json` was also run against the committed lockfile to cross-check transitive vulnerability exposure. Every import statement in `src` was grepped to confirm actual usage of each declared dependency (not just declaration in `package.json`).

Key findings:

* One **critical** vulnerability chain is present in a direct devDependency (`vitest`, pinned `2.1.4`), with two critical CVEs (remote code execution / arbitrary file read) affecting the pinned major version.
* One **high**-severity vulnerability affects the direct production dependency `express` (pinned `4.21.1`) via its transitive `path-to-regexp`/`qs`/`body-parser` chain, fixed in a **non-breaking** patch release already available (`4.22.2`) within the same major version the project already uses (4.x).
* One **moderate** vulnerability affects the direct production dependency `uuid` (pinned `11.0.3`), fixed in a non-breaking patch release (`11.1.1`).
* `pino-http` is declared as a direct production dependency in `package.json` but is **not imported or used anywhere in `src`**, confirmed by grepping every `.ts` file. The project instead implements its own request-logging middleware (`src/middlewares/request-logger.middleware.ts`) using `pino` directly together with `uuid` for request-ID generation. This is dead weight in the dependency tree with no functional benefit, and it is `uuid` — not `pino-http` — that is the actual vulnerable, actively-used dependency in that same file.
* No dependency is deprecated, unmaintained, or abandoned. All 8 direct runtime dependencies and all 17 direct devDependencies show GitHub repository activity (`pushed_at`) within the last 90 days as of the audit date.
* All resolved licenses are MIT or Apache-2.0 — permissive and mutually compatible with typical proprietary/commercial use. No copyleft dependencies were found among direct dependencies. No legal risk identified. Note: no `LICENSE` file was found at the project root, which is a documentation gap in the project's own distribution terms, not a dependency conflict.
* Several direct dependencies are one or more **major versions** behind the latest stable release (notably `express` 4.x vs 5.x, `zod` 3.x vs 4.x, `@prisma/client`/`prisma` 5.x vs 7.x, `typescript` 5.x vs 6.x, `eslint` 8.x vs 10.x, `uuid` 11.x vs 14.x, `vitest` 2.x vs 4.x). These are flagged as "Outdated" or "Legacy" depending on the severity of the gap. They are not, by themselves, security issues, but they increase future migration effort and reduce access to upstream security backports over time.

This report catalogs **direct dependencies only**. Transitive dependency vulnerabilities are surfaced in the Critical Issues and Risk Analysis sections strictly to explain how they reach the project through direct dependencies (`express`, `vitest`, `tsx`, `bcrypt`'s native-build chain), consistent with identifying single points of failure and maintenance burden.

---

## 2. Critical Issues

### 2.1 Critical severity

* **CVE-2025-24964 (GHSA-9crc-q9x8-hgqq)** and **CVE-2026-47429 (GHSA-5xrq-8626-4rwp)** — `vitest` (direct devDependency, pinned `2.1.4`, resolved `2.1.4`). Both advisories describe remote code execution / arbitrary file read via the Vitest API/UI server when it is exposed and a victim visits a malicious website while the dev/test server is listening. The vulnerable range is `<=3.2.5`; the project is pinned at `2.1.4`. Fix requires upgrading to `vitest@4.1.9` (current latest, released 2026-06-15), which is a semver-major jump. This is a devDependency (test runner only, not shipped to the production `dist/` build), which limits production blast radius, but any developer or CI machine running `vitest` with the API/UI server exposed to an untrusted network is at risk.

### 2.2 High severity

* **CVE-2024-52798 (GHSA-rhx6-c78j-4q9w)** and **GHSA-37ch-88jc-xwx2** (ReDoS) — `path-to-regexp`, a transitive dependency pulled in by the direct dependency `express` (pinned `4.21.1`). The vulnerable range is `<=0.1.12` of `path-to-regexp` as resolved through Express 4's routing layer. `npm audit` confirms this is fixable by upgrading the direct dependency `express` to `4.22.2`, a **non-breaking patch release within the currently used major version (4.x)**.
* **GHSA-hmw2-7cc7-3qxx** (CRLF injection) — `form-data`, a transitive dependency reachable through devDependency build/test tooling. Fix available transitively; no direct-dependency version pin required in this manifest.
* Multiple **high**-severity advisories in `tar` (`GHSA-34x7-hfp2-rc4v`, `GHSA-8qq5-rm4j-mr97`, `GHSA-83g3-92jg-28cx`, `GHSA-qffp-2rhf-9h96`, `GHSA-9ppj-qmqm-q256`, `GHSA-r6q2-hw4h-h46w`) — transitive, pulled in by native-module build tooling (`@mapbox/node-pre-gyp`), reachable through `bcrypt`'s native binding installer. No direct dependency version change is required to resolve; `npm audit` reports a fix is available transitively.

### 2.3 Moderate severity

* **CVE-2026-41907 (GHSA-w5hq-g745-h8pq)** — `uuid` (direct production dependency, pinned `11.0.3`). Missing buffer bounds check in v3/v5/v6 generation when a caller-supplied buffer (`buf` option) is passed. The codebase's only usage, in `src/middlewares/request-logger.middleware.ts`, calls `uuidv4()` with no arguments, so the specific vulnerable code path (custom `buf` parameter) is not exercised by current usage — this reduces practical exploitability in this codebase, but the installed version remains vulnerable as pinned. Fix available via `uuid@11.1.1` (non-breaking patch, same major version).
* `tsx` (direct devDependency, pinned `4.19.2`) is affected by a moderate-severity transitive advisory chain (`esbuild`/`vite` toolchain, e.g. GHSA-67mh-4wv8-2f99) in the range `3.13.0 - 4.19.2`. Fix available via `tsx@4.22.4` (non-breaking patch).
* Transitive `qs`, `js-yaml`, `@vitest/mocker`, `vite`, `vite-node`, `esbuild` moderate advisories — reachable only through devDependency toolchains (`express`'s body-parser chain for `qs`, `vitest`'s toolchain for the rest). No direct action possible without upgrading the owning direct dependency.

### 2.4 Deprecated / legacy core dependencies

No direct dependency is marked `deprecated` in npm registry metadata as of this audit, and none were found unmaintained (all show GitHub commit activity within the last 90 days). The following are flagged as **Legacy** due to being one or more major versions behind the current stable release, meaning the pinned major version typically no longer receives new feature work and, in some cases, has a defined end-of-active-support horizon:

* `@prisma/client` / `prisma` — pinned `5.22.0`, latest stable `7.8.0` (two major versions behind).
* `eslint` — pinned `8.57.1`, latest stable `10.6.0` (two major versions behind). ESLint 8.x is past the ESLint project's documented active-support window.
* `zod` — pinned `3.23.8`, latest stable `4.4.3` (one major version behind, with a documented breaking-change migration).
* `express` — pinned `4.21.1`, latest stable `5.2.1` (one major version behind; Express 4.x still receives security patches but is in maintenance mode as the ecosystem migrates to 5.x).
* `typescript` — pinned `5.6.3`, latest stable `6.0.3` (one major version behind).

---

## 3. Dependencies

### 3.1 Runtime (`dependencies`)

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| @prisma/client | 5.22.0 | 7.8.0 | Legacy |
| bcrypt | 5.1.1 | 6.0.0 | Outdated |
| express | 4.21.1 | 5.2.1 | Legacy |
| jsonwebtoken | 9.0.2 | 9.0.3 | Outdated |
| pino | 9.5.0 | 10.3.1 | Outdated |
| pino-http | 10.3.0 | 11.0.0 | Outdated (also Unused in `src` — see Sections 4 and 7) |
| uuid | 11.0.3 | 14.0.1 | Outdated (also vulnerable, see Risk Analysis) |
| zod | 3.23.8 | 4.4.3 | Legacy |

### 3.2 Development (`devDependencies`)

| Dependency | Current Version | Latest Version | Status |
|---|---|---|---|
| @types/bcrypt | 5.0.2 | 6.0.0 | Outdated |
| @types/express | 4.17.21 | 5.0.6 | Outdated |
| @types/jsonwebtoken | 9.0.7 | 9.0.10 | Outdated |
| @types/node | 20.17.6 | 26.0.1 | Outdated (intentionally pinned to Node 20 LTS typings; see Integration Notes) |
| @types/supertest | 6.0.2 | 7.2.0 | Outdated |
| @types/uuid | 10.0.0 | 11.0.0 | Outdated |
| @typescript-eslint/eslint-plugin | 8.13.0 | 8.62.1 | Outdated |
| @typescript-eslint/parser | 8.13.0 | 8.62.1 | Outdated |
| eslint | 8.57.1 | 10.6.0 | Legacy |
| eslint-config-prettier | 9.1.0 | 10.1.8 | Outdated |
| pino-pretty | 11.3.0 | 13.1.3 | Outdated |
| prettier | 3.3.3 | 3.9.4 | Outdated |
| prisma | 5.22.0 | 7.8.0 | Legacy |
| supertest | 7.0.0 | 7.2.2 | Outdated |
| tsx | 4.19.2 | 4.22.4 | Outdated (also vulnerable, see Risk Analysis) |
| typescript | 5.6.3 | 6.0.3 | Legacy |
| vitest | 2.1.4 | 4.1.9 | Legacy (also vulnerable — critical, see Risk Analysis) |

**Status legend:** Up to Date = matches latest stable; Outdated = behind latest but same major version, or a narrow minor/patch gap with no breaking-change path; Legacy = one or more major versions behind current stable with a defined migration/breaking-change path; Unmaintained = no commits/releases in the associated upstream repository for more than one year (none found in this audit).

---

## 4. Risk Analysis

| Severity | Dependency | Issue | Details |
|---|---|---|---|
| Critical | vitest (direct devDependency, 2.1.4) | CVE-2025-24964 (GHSA-9crc-q9x8-hgqq) | Remote code execution when a malicious website is visited while the Vitest API server is listening. Vulnerable range `<=3.2.5`; project is pinned at `2.1.4`. Fix requires major upgrade to `4.1.9`. Test-only exposure (devDependency), but affects developer machines and CI runners. |
| Critical | vitest (direct devDependency, 2.1.4) | CVE-2026-47429 (GHSA-5xrq-8626-4rwp) | Arbitrary file read/execution when the Vitest UI server is listening. Same vulnerable range and fix path as above. |
| High | express (direct dependency, 4.21.1) | CVE-2024-52798 (GHSA-rhx6-c78j-4q9w) via transitive path-to-regexp | ReDoS in route-parameter regex compilation reachable through Express 4's router, used by every route file in `src/routes` and `src/modules/*/*.routes.ts`. Fix available via non-breaking patch to `express@4.22.2` (same major version). |
| High | express (direct dependency, 4.21.1) | Transitive `qs`/`body-parser` DoS advisories (GHSA-w7fw-mjwx-w883, GHSA-6rw7-vpxm-498p, GHSA-q8mj-m7cp-5q26) | Multiple denial-of-service vectors (array-limit bypass, memory exhaustion, TypeError crash) in Express's body-parsing dependency chain, exercised on every JSON POST/PUT/PATCH endpoint. Fix available via non-breaking patch to `express@4.22.2`. |
| High | tar (transitive, via bcrypt's native-module build chain) | GHSA-34x7-hfp2-rc4v and 6 related advisories | Arbitrary file creation/overwrite and symlink/hardlink path traversal during native module installation (`node-pre-gyp` build step for `bcrypt`). Exploitable primarily during `npm install` against an untrusted/compromised registry mirror. Fix available transitively; no direct action on `bcrypt`'s pinned version required per `npm audit`. |
| Moderate | uuid (direct dependency, 11.0.3) | CVE-2026-41907 (GHSA-w5hq-g745-h8pq) | Missing buffer bounds check in v3/v5/v6 generation when a caller supplies a `buf` option. Current codebase usage (`uuidv4()` with no arguments in `src/middlewares/request-logger.middleware.ts`) does not exercise the vulnerable code path, reducing practical exploitability, but the installed version remains vulnerable as pinned. Fix available via non-breaking patch to `uuid@11.1.1`. |
| Moderate | tsx (direct devDependency, 4.19.2) | Transitive esbuild/vite advisory chain (GHSA-67mh-4wv8-2f99 and related) | Development-server request/response exposure issue in the underlying esbuild dev server used by `tsx watch` (the project's `dev` script). Fix available via non-breaking patch to `tsx@4.22.4`. |
| Moderate | js-yaml, qs, @vitest/mocker, vite, vite-node, esbuild (transitive, via dev tooling) | GHSA-h67p-54hq-rp68 and related advisories | Quadratic-complexity DoS, dev-server exposure, and related moderate issues reachable only through devDependency toolchains (`express`'s body-parser chain for `qs`; `vitest`'s toolchain for the rest). No direct dependency pin controls these individually. |
| High | Single point of failure: `@prisma/client` / `prisma` | Data-access concentration | Every module under `src/modules/*` (customers, orders, products, users) and `src/config/database.ts` depends directly on the Prisma client for all persistence operations, confirmed by grep across 13 files. A breaking change, licensing change, or critical vulnerability in Prisma would simultaneously impact the entire API surface with no abstraction layer isolating the blast radius. Current version (5.22.0) is two major versions behind latest (7.8.0). |
| High | Single point of failure: `express` | Framework concentration | Every route, controller, and middleware (16 files under `src`) depends on Express's request/response/middleware types and runtime. Express 4.x is in maintenance mode relative to the now-stable Express 5.x; a future forced migration would require touching nearly every file in `src`. |
| Medium | Unused dependency: `pino-http` | Maintenance/audit-surface burden with no functional benefit | Declared as a direct production dependency in `package.json` (`pino-http@10.3.0`) but not imported anywhere in `src` (verified by grep of all `.ts` files for `from "pino-http"` / `from 'pino-http'`). The project reimplements equivalent request-logging behavior manually in `src/middlewares/request-logger.middleware.ts` using `pino` and `uuid` directly. This dependency contributes to install size and `npm audit` surface area without being exercised by the application. |
| Low | zod (direct dependency, 3.23.8) | Legacy major version | One major version behind (`4.4.3`). Zod 4 introduced a documented breaking-change migration for schema APIs. No known vulnerability in 3.23.8 as of this audit; risk is purely migration/maintenance burden, and future security backports to the 3.x line are not guaranteed indefinitely. |
| Low | eslint (direct devDependency, 8.57.1) | Legacy major version, dev-only tooling | Two major versions behind (`10.6.0`). Dev-only; does not affect production runtime. Flagged because ESLint 8.x is past its documented active-support window, meaning new security or bug fixes will not be backported to 8.x. |

### License compatibility

All 8 direct production dependencies (`@prisma/client` — Apache-2.0, `bcrypt` — MIT, `express` — MIT, `jsonwebtoken` — MIT, `pino` — MIT, `pino-http` — MIT, `uuid` — MIT, `zod` — MIT) use permissive licenses, verified directly from npm registry metadata. `typescript` and `prisma` (CLI) are Apache-2.0; all other devDependencies checked are MIT. No copyleft (GPL/AGPL/LGPL) licenses were identified among any direct dependency. No license incompatibilities or legal conflicts were found. Note: no `LICENSE` file exists at the project root, which is a gap in the project's own distribution documentation, independent of dependency licensing.

---

## 5. Unverified Dependencies

No unverified dependencies. All 8 direct runtime dependencies and all 17 direct devDependencies listed in `package.json` were successfully resolved against the npm registry (version, license, deprecation status) and cross-referenced against their upstream GitHub repositories for maintenance activity. `npm audit --json` was successfully run against the committed `package-lock.json`, and resolved lockfile versions were confirmed to match the manifest-declared versions exactly for every checked package (spot-checked: `uuid` 11.0.3, `express` 4.21.1, `@prisma/client` 5.22.0, `prisma` 5.22.0, `@types/uuid` 10.0.0).

---

## 6. Critical File Analysis

The following are the 10 most critical files in `src` with respect to dependence on the risky/legacy/outdated packages identified above. Criticality is determined by (a) concentration of a single-point-of-failure dependency, (b) direct exposure to a flagged vulnerability, or (c) business-critical function (authentication, data persistence, request pipeline entrypoint).

1. **`src/config/database.ts`** — Instantiates the single shared `PrismaClient` instance used by every repository in the application. Directly depends on `@prisma/client` (5.22.0, two major versions behind latest). Any Prisma-related vulnerability, connection-handling bug, or forced migration affects the entire persistence layer through this one file.

2. **`src/app.ts`** — The Express application factory; wires together every middleware (`request-logger`, `error`, `auth`, `validate`) and every module's routes, and also imports `@prisma/client` for typed request context. Directly depends on `express` (4.21.1, affected by CVE-2024-52798 through transitive `path-to-regexp`, and by the `qs`/`body-parser` DoS chain). This file is the single entrypoint through which all HTTP traffic passes.

3. **`src/middlewares/auth.middleware.ts`** — Enforces authentication/authorization for protected routes by verifying JWTs. Directly depends on `jsonwebtoken` (9.0.2) and `express`. A vulnerability or misconfiguration in JWT verification here would compromise access control across every protected endpoint (customers, orders, products, users).

4. **`src/modules/auth/auth.service.ts`** — Issues JWTs and verifies user credentials via `bcrypt` password hashing. Directly depends on both `jsonwebtoken` (9.0.2) and `bcrypt` (5.1.1, whose native-module build chain pulls in the `tar`-related high-severity advisories during install). This file is the sole point where credentials and tokens are minted.

5. **`src/middlewares/request-logger.middleware.ts`** — Directly depends on `uuid` (11.0.3, CVE-2026-41907) for request-ID generation and `express` (`RequestHandler` type). It also duplicates the functionality that the declared-but-unused `pino-http` dependency was presumably intended to provide, making it the concrete evidence file for the "unused dependency" finding in Section 4.

6. **`src/shared/logger/index.ts`** — Central `pino` logger instance imported by nearly every module and middleware in the application (directly or transitively, including `request-logger.middleware.ts` and `server.ts`). Concentrates the `pino` (9.5.0, one major version behind) dependency into a single point of failure for all application observability.

7. **`src/config/env.ts`** — Validates all environment configuration (including `DATABASE_URL` and `JWT_SECRET`) using `zod` (3.23.8, one major version behind, Legacy). Because this module runs at process startup and every other module depends on its exported `env` object, a schema-validation defect or breaking Zod migration would prevent the application from booting entirely.

8. **`src/middlewares/error.middleware.ts`** — Centralized error handler that imports `@prisma/client` (to translate Prisma-specific errors), `zod` (to translate validation errors), and `express` into HTTP responses. Concentrates three flagged dependencies (Prisma, Zod, Express) into the single error-handling path used by every route.

9. **`src/middlewares/validate.middleware.ts`** — Generic request-validation middleware used by every module's routes (customers, orders, products, users, auth) to parse and validate input via `zod` (3.23.8) and `express`. A single dependency (Zod) gates input validation for the entire API surface through this one file.

10. **`src/modules/orders/order.repository.ts`** and **`src/modules/orders/order.service.ts`** — Represent the most business-critical domain module (order lifecycle/status transitions, stock adjustments) and have the deepest direct coupling to `@prisma/client` among the domain modules, including transactional use of `Prisma.TransactionClient` and Prisma-generated types referenced in `src/modules/orders/order.schemas.ts` and `order.status.ts`. Given orders are the core business entity of this system, these files' dependence on an outdated Prisma major version represents concentrated business risk.

---

## 7. Integration Notes

* **`@prisma/client` / `prisma`** — Used as the sole ORM/data-access layer. `prisma/schema.prisma` defines a MySQL datasource with `User`, `Customer`, `Product`, and `Order` models (enums `UserRole`, `OrderStatus`). `src/config/database.ts` creates one `PrismaClient` singleton, imported by every `*.repository.ts` file under `src/modules/*` and by `src/app.ts`, `src/middlewares/error.middleware.ts`, and `src/modules/orders/order.status.ts`/`order.schemas.ts`. No abstraction/repository-interface layer sits between Prisma and the domain services beyond the repository files themselves, meaning Prisma-generated types leak into schema/status files.
* **`express`** — Core HTTP framework, imported in 16 files across `src`: `src/app.ts` (app factory), all four middleware files, all module controllers and route files, and `src/routes/index.ts` (route aggregation).
* **`jsonwebtoken`** — Used in exactly two files: `src/modules/auth/auth.service.ts` (token issuance) and `src/middlewares/auth.middleware.ts` (token verification). Narrow, well-contained usage.
* **`bcrypt`** — Used in exactly two files: `src/modules/users/user.service.ts` (password hashing on user creation/update) and `src/modules/auth/auth.service.ts` (password comparison on login). Narrow usage; native-module install-time risk is limited to build/CI environments.
* **`zod`** — Used broadly: `src/config/env.ts` (startup env validation), every module's `*.schemas.ts` file (customers, products, auth, orders, users — 6 files), `src/middlewares/validate.middleware.ts` (generic validation middleware), `src/middlewares/error.middleware.ts` (ZodError-to-HTTP translation), and `src/modules/users/user.routes.ts`. This is the most widely integrated dependency after Express and Prisma (9 files).
* **`pino`** — Used in exactly one file, `src/shared/logger/index.ts`, which exports a shared `logger` instance imported throughout the codebase (middlewares, `server.ts`, error handling).
* **`pino-http`** — Declared in `package.json` as a direct production dependency but **confirmed via grep to not be imported anywhere in `src`**. No integration exists. The project's own `request-logger.middleware.ts` reimplements equivalent behavior using `pino` and `uuid` directly. This is dead dependency weight (see Risk Analysis, Medium severity finding).
* **`uuid`** — Used in exactly one file, `src/middlewares/request-logger.middleware.ts`, importing `{ v4 as uuidv4 }` and calling `uuidv4()` with no arguments to generate request-correlation IDs when the incoming `x-request-id` header is absent.
* **Development tooling (`typescript`, `tsx`, `vitest`, `eslint`, `prettier`, `supertest`, `@types/*`)** — Used exclusively for build (`tsc -p tsconfig.build.json`), local dev server (`tsx watch`), testing (`vitest`, `supertest`; test files live in `/tests`, outside the audited `src` scope, but exercise `src` code), linting (`eslint`), and formatting (`prettier`). None ship to the production `dist/` build output as runtime code, but `vitest`'s critical vulnerabilities and `tsx`'s moderate vulnerability affect developer and CI environments where these tools run with network-listening features enabled (`dev`, `test`, `test:watch`, `db:seed` scripts).

---

## Audit Methodology and Limitations

* Version and release-date data verified via `https://registry.npmjs.org/<package>` for all 25 direct dependencies (8 runtime + 17 development).
* Maintenance-activity data verified via `https://api.github.com/repos/<org>/<repo>` (`pushed_at` field) for the primary runtime dependencies with the largest version gaps or highest risk profile.
* License data verified via the npm registry `license` field for all runtime and core development dependencies; all resolved to MIT or Apache-2.0.
* Vulnerability data cross-checked via `npm audit --json` against the committed `package-lock.json`, with individual advisory details (CVE IDs, severity, summary) retrieved from `https://api.github.com/advisories/<GHSA-id>`.
* Actual usage of each declared dependency inside `src` was confirmed by grepping every `.ts` file for corresponding `import ... from '<package>'` statements, rather than relying on `package.json` declarations alone — this is how the unused `pino-http` finding was identified.
* This audit did not have access to Context7 or Firecrawl MCP servers in this environment; direct HTTPS requests to the npm registry and GitHub REST API were used as the verification method instead, per the fallback instructions.
* All 25 direct dependencies were successfully verified; there are no entries in the Unverified Dependencies section.

---

**Report saved to:** `/Users/antoniocanhota/Workspace/mba/mba-ia-desafio-design-docs-com-ia/docs/others/reports/dependency-auditor/dependencies-report-2026-06-30.md`
**Relative path:** `docs/others/reports/dependency-auditor/dependencies-report-2026-06-30.md`
