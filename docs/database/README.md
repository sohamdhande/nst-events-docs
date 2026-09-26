# Database Architecture Documentation

This directory contains the authoritative PostgreSQL schema, PostGIS configuration, RLS policies, queue mechanics, and Prisma ORM documentation for the NST-Events platform.

---

## Authoritative Documentation Suite

The following 9 core documents establish the verified source of truth for all database operations, migrations, and application data models:

1. **[PostgreSQL Architecture](architecture.md):** Engine version (PostgreSQL 16), verified extensions (`postgis`, `pgcrypto`), K3s StatefulSet topology, and operational memory tuning.
2. **[Database Schema & Domain Catalog](schema.md):** Complete domain-by-domain architectural explanation of all 24 application tables, primary keys, foreign keys, unique constraints, and indexes.
3. **[Prisma ORM Integration](prisma.md):** Prisma 5.22.0 configuration, monorepo client consumption (`apps/api`, `apps/worker`), raw SQL boundaries, and the enum migration trap (`55P04`).
4. **[PostGIS & Geospatial Architecture](postgis.md):** EPSG:4326 (WGS 84), `geography(Point, 4326)`, `ST_DWithin` geofencing, Phase 30 GPS accuracy buffer, and anti-fraud location validation.
5. **[Row Level Security & Authorization](rls.md):** Defense-in-depth model, transaction-local user context (`withUserContext`), helper functions, and the complete 24-table RLS policy matrix.
6. **[Functions, Triggers, & Views Catalog](functions-and-triggers.md):** Complete catalog of PL/pgSQL stored procedures (RPCs), audit triggers, global role protection, public profile views, and materialized views.
7. **[Queue Architecture & Background Processing](queues.md):** Definitive documentation of the `notification_jobs` table queue, `FOR UPDATE SKIP LOCKED` claiming, async receipt polling, and DLQ handling.
8. **[Migration History & Deployment Strategy](migrations.md):** 103-migration history, destructive changes, Expand-and-Contract zero-downtime rules, and Kubernetes pre-deploy migration Jobs.
9. **[Connections, Pooling, & Roles](connections-and-roles.md):** Role separation (`postgres`, `nst_app`, `nst_worker`), connection pool sizing, and the persistent SSE listener connection.

---

## Historical & Topic Deep-Dives
The legacy numbered documents provide historical reference for specific subsystem decisions:
- [02. ER Diagram](02-er-diagram.md)
- [03. Table Catalog](03-table-catalog.md)
- [04. Enums](04-enums.md)
- [06. Indexing Strategy](06-indexing-strategy.md)
- [08. JSONB Governance](08-jsonb-governance.md)
- [09. Soft Delete Strategy](09-soft-delete-strategy.md)
- [10. Attendance Data Model](10-attendance-data-model.md)
- [14. Leaderboard Data Model](14-leaderboard-data-model.md)
- [19. Scalability Review](19-scalability-review.md)
- [20. Database Freeze V1](20-database-freeze-v1.md)
- [21. Realtime LISTEN/NOTIFY Contract](21-realtime-listen-notify-contract.md)
