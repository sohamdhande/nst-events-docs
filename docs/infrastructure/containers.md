# Container Architecture

This document defines the containerization strategy and Dockerfile requirements for the `nst-events` services.

## 1. Application Containers

All application containers must follow best practices for production Node.js deployments:
- Multi-stage builds to minimize image size.
- Using `pnpm` for deterministic dependency resolution.
- Running as a non-root user (`node`) for enhanced security.

### API (`apps/api`)
* **Current State**: `docker/Dockerfile.api` is implemented correctly as a multi-stage `pnpm` build, running the Express server on port 3001.
* **Action**: Verified and ready for production use.

### Worker (`apps/worker`)
* **Current State**: `docker/Dockerfile.worker` is implemented correctly as a multi-stage `pnpm` build, running the worker on port 3002.
* **Action**: Verified and ready for production use.

### Dashboard (`apps/dashboard`)
* **Current State**: **Missing.** There is no Dockerfile for the Next.js dashboard in the repository.
* **Required Change**: A robust multi-stage Dockerfile optimized for Next.js standalone output must be created for the dashboard.

---

## 2. Database Container (PostgreSQL)

> [!WARNING]
> **PostgreSQL Configuration Conflict**
> The `nst-events` repository currently has conflicting definitions for the database container. We cannot finalize the database architecture until this is resolved.

### The Conflict
1. **Docker Compose (`docker/docker-compose.yml`)**: Uses a custom `Dockerfile.postgres` built `FROM postgis/postgis:16-3.4` and installs `postgresql-16-pgtap`.
2. **Kubernetes StatefulSet (`infrastructure/kubernetes/postgres-statefulset.yaml`)**: Uses the image `quay.io/tembo/pg16-pgmq:latest`.

### The Requirement
The final PostgreSQL container must support the exact extensions required by the `nst-events` schema (currently known: `postgis`, `pgcrypto`). 

**Crucially, we must verify the actual queue implementation used by the worker.** 
- If the worker relies on `pgmq`, the database image must include the `pgmq` extension (which standard PostGIS images do not have).
- If the worker relies on `pg_cron`, it must be installed.

### Required Change
**Do not prematurely select a base image.** 
A detailed audit of the worker and database schema is required to determine the exact extension dependencies. Once confirmed, a single, unified `Dockerfile.postgres` must be created and referenced by both local development and Kubernetes manifests.
