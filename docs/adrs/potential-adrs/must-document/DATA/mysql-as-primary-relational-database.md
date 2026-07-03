# Potential ADR: MySQL as the Primary Relational Database

**Module**: DATA
**Category**: Technology (Infrastructure Service)
**Priority**: Must Document (Score: 150)
**Date Identified**: 2026-07-01

---

## What Was Identified

The order-management-api uses MySQL 8.0 as its sole persistent data store for every domain entity in the system (users, customers, products, orders, order items, order status history, and the order-numbering sequence). The database is provisioned locally via `docker-compose.yml` (service `oms-mysql`, image `mysql:8.0`), configured with a custom `utf8mb4`/`utf8mb4_unicode_ci` charset/collation, a persistent named volume (`oms_mysql_data`), and a health check gating dependent services. Connectivity is driven entirely through the `DATABASE_URL` environment variable (validated in `src/config/env.ts`), consumed by Prisma's generated client (`src/config/database.ts`).

This is a foundational, infrastructure-level technology choice: every feature module (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS) depends on this database for all persistence, and the schema itself (`prisma/schema.prisma`) is written using MySQL-specific type mappings (`@db.Char(36)`, `@db.VarChar`, `@db.Text`, `ENUM` columns for `UserRole`/`OrderStatus`). A separate `SHADOW_DATABASE_URL` is also configured for Prisma's migration-diffing workflow, indicating the choice extends into the development/CI tooling as well.

The repository currently has a single "init repository" commit (2026-06-24), so no long-term evolution history exists yet — but the choice is already deeply embedded: the entire schema, all migrations, and the seed script are written specifically against MySQL syntax and semantics (e.g., `ENUM` types, `CHAR(36)` for UUIDs, JSON column support for the `Customer.address` field).

## Why This Might Deserve an ADR

- **Impact**: Every module in the system (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS, API-CORE) depends on this database; it is the single source of truth with no alternative or fallback store.
- **Trade-offs**: MySQL was chosen over alternatives like PostgreSQL (no native UUID/JSONB types, weaker JSON querying, different concurrency/locking model) — none of these trade-offs are currently documented anywhere in the repo.
- **Complexity**: Schema uses MySQL-specific features (ENUM columns, `utf8mb4` for full Unicode/emoji support, custom character set/collation) that would require translation work to port to another RDBMS.
- **Team Knowledge**: Every engineer working on any feature module needs to understand this is the persistence layer, how local dev/test databases are provisioned (Docker Compose), and how migrations/shadow databases work.
- **Future Implications**: Switching databases later (e.g., for managed cloud RDS, read replicas, or a different engine) would require a full data migration and schema rewrite — a very high cost-to-change decision.

## Evidence Found in Codebase

### Key Files
- [`docker-compose.yml`](../../../../../docker-compose.yml) - Lines 1-24
  - Defines the MySQL 8.0 service, credentials, health check, and persistent volume for local development
- [`prisma/schema.prisma`](../../../../../prisma/schema.prisma) - Lines 1-9
  - `datasource db { provider = "mysql" ... }` declares MySQL as the target engine, with a shadow database URL for migrations
- [`src/config/env.ts`](../../../../../src/config/env.ts) - Line 7
  - Validates `DATABASE_URL` as a required environment variable
- [`src/config/database.ts`](../../../../../src/config/database.ts)
  - Instantiates the shared `PrismaClient` connected to the MySQL instance
- [`.env.example`](../../../../../.env.example) - Lines 5-6
  - Shows both `DATABASE_URL` and `SHADOW_DATABASE_URL` pointing at local MySQL

### Code Evidence
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    container_name: oms-mysql
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE:-oms}
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
```

```prisma
// prisma/schema.prisma
datasource db {
  provider          = "mysql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

### Impact Analysis
- Introduced: 2026-06-24 (single "init repository" commit — repository has no prior history)
- Modified: 0 subsequent commits (schema and compose files unchanged since init)
- Affects: all 7 Prisma models, all 5 feature modules, and the local dev/CI tooling (migrations, shadow DB, seed script)
- Git history not available beyond the initial commit; no evolution/refactor themes to report yet

### Alternatives (if observable)
No explicit alternatives (e.g., PostgreSQL, SQLite) are mentioned in code comments, config, or commit messages. The choice appears as a starting default with no recorded rationale.

## Questions to Address in ADR (if created)

- What problem was being solved, and why was a relational database chosen over NoSQL alternatives?
- Why MySQL specifically, versus PostgreSQL or another RDBMS?
- What are the implications of MySQL-specific schema choices (ENUM columns, `utf8mb4`, JSON column for `Customer.address`) for portability?
- What is the plan for production hosting (self-managed vs. managed cloud service like AWS RDS/PlanetScale)?
- What are the backup, replication, and scaling strategies as order volume grows?

## Related Potential ADRs
- Prisma ORM for Data Access (same module, tightly coupled — Prisma's MySQL provider and shadow-database migration workflow depend directly on this choice)

## Additional Notes
The project is very early-stage (single commit, `oms-mysql_shadow` referenced only in `.env.example`), so this decision has not yet been battle-tested with real data volume, concurrent load, or production deployment concerns. Documenting it now — while the rationale is still fresh — would be valuable before the decision becomes implicit tribal knowledge.
