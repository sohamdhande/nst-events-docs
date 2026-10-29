# ADR-031: Admin-Managed Privacy Policy Architecture & System Policy Persistence

## Status
Accepted (Scoped Implementation Authorized; Production Launch Subject to Required Institutional Legal Review)

## Context
Currently, the NST Events privacy policy at `/privacy-policy` is implemented as a 140-line static, hard-coded Next.js TSX component (`apps/dashboard/app/(public)/privacy-policy/page.tsx`). Any legal update, institutional contact change, or App Store compliance amendment requires an engineer to modify source code, commit, build a new container image (`ghcr.io/nst-sdc/nst-dashboard`), and perform a Kubernetes rollout.

Furthermore, the Next.js dashboard runs with two stateless replicas (`replicas: 2`) in Kubernetes with no persistent volume mount. Local filesystem storage (`fs.writeFile`) is impossible because it causes split-brain state between replicas and loses all modifications upon pod restart or rollout.

The platform requires a solution that enables authorized Platform Admins to draft, preview, and publish Markdown-formatted policies dynamically through the Admin Ops dashboard without frontend redeployment, while ensuring public visitors and mobile apps see only published content.

However, the NST Events codebase is governed by strict architectural boundaries:
- `apps/api` and `packages/database` are frozen post-audit (`AGENTS.md`, `docs/engineering/adrs/README.md`).
- Modifying the schema or adding API routes is strictly prohibited without formal governance authorization and an approved ADR.
- All documentation-only proposals must clearly distinguish proposed designs from executed validations.

## Decision

1. **Unfreeze Governance Boundary for System Policies (Authorized Implementation Exception)**:
   - Explicit implementation authorization was granted as the minimum scoped exception to the existing architecture freeze for:
     1. `packages/database`: adding `model SystemPolicy`, reciprocal relation on `model User`, and migration `20261101000000_add_system_policies`.
     2. `apps/api`: adding route module `src/modules/policies/` (`GET /v1/policies/:slug`, `GET /v1/admin/policies/:slug`, `PUT /v1/admin/policies/:slug`).
     3. `apps/dashboard`: adding admin editor at `/admin/privacy-policy` and route handler at `/privacy-policy`.
   - The broader architecture freeze remains in effect for all other areas of the repository.
   - **Institutional Legal Review Status**: Implementation authorization provides technical persistence, validation, and editing capabilities. Production launch remains subject to required institutional review by the university's Office of the General Counsel to confirm statutory compliance (e.g. India DPDP Act 2023).

2. **Persistence via PostgreSQL (`SystemPolicy` Model) & Attribution Semantics**:
   - Dedicated `system_policies` table keyed by slug (`slug = 'privacy-policy'`).
   - Stores separate `published_content` and `draft_content` text columns to allow draft authoring without exposing unapproved changes to the public.
   - Reciprocal relation on `model User`: `publishedPolicies SystemPolicy[] @relation("PolicyPublisher")`.
   - Verified and validated with Prisma ORM v5.22.0 (`prisma validate: valid 🚀`).
   - Migration `20261101000000_add_system_policies` created with table DDL, RLS policies, table grants for `nst_app` and `nst_worker`, and safe baseline seed inserting the extracted JSX text as Markdown and initial audit log record (`actor_id = NULL`, `entity_id = NULL`).
   - **Publisher Deletion & Attribution Semantics**:
     - `onDelete: SetNull` on `SystemPolicy.publishedBy` applies strictly to physical SQL row deletion in PostgreSQL. In NST Events, user accounts are soft-deleted via `deleted_at DateTime?`, which does not trigger foreign key nullification.
     - `published_content`, `published_version`, and `published_at` are stored as independent scalar columns on `system_policies`. Public policy serving does not join `users` and remains technically accessible regardless of whether the publisher user is active, soft-deleted, deactivated, or physically dropped.
     - The publisher's identity (name, email, UUID) is permanently recorded in `audit_logs.new_state` on every publication for historical attribution.

3. **Decoupled Dual Versioning & Optimistic Concurrency**:
   - `concurrency_revision` (internal integer): Increments on EVERY write (`SAVE_DRAFT` and `PUBLISH`). Checked conditionally in SQL (`WHERE slug = $slug AND concurrency_revision = $base_revision`) to detect concurrent edits and return HTTP `409 Conflict`.
   - `published_version` (public integer): Increments ONLY upon explicit `PUBLISH` action, displayed in the public header (e.g., "Version 2").
   - **Non-Destructive 409 Conflict Handling**: Client preserves the editor text locally and displays an interactive reconciliation modal comparing local unsaved edits against remote server state.

4. **Content-Size Limit: 100 KiB UTF-8 Bytes**:
   - The authoritative server-side validation enforces **100 KiB of UTF-8 bytes** (`Buffer.byteLength(val, 'utf8') <= 100 * 1024` = 102,400 bytes).
   - JavaScript `.length` (UTF-16 code units) is NOT used as byte count. Multi-byte UTF-8 characters and emojis (e.g. 4-byte glyphs) are validated by byte size.
   - Express body parser configured with route-scoped `express.json({ limit: '256kb' })` for `/v1/admin/policies` to accommodate 102,400 bytes UTF-8 Markdown plus JSON wrapper, while retaining strict 100kb limits on all unrelated routes, with 413 PayloadTooLargeError handling in `errorHandler`.

5. **Public Route Architecture: Route Handler Replacement (`route.ts`)**:
   - **Selected Architecture**: Option 1 (Route Handler Replacement: `apps/dashboard/app/(public)/privacy-policy/route.ts`).
   - Next.js App Router forbids `page.tsx` and `route.ts` at the same route segment. `page.tsx` was replaced with `route.ts`.
   - On API success: returns rendered HTML (with safe markdown rendering, metadata, and cache headers `Cache-Control: public, max-age=60, s-maxage=60, stale-while-revalidate=60`) with HTTP 200.
   - On API failure/timeout/500: returns brutalist service notice HTML with wire-level HTTP 503 and `Cache-Control: no-store, no-cache, must-revalidate, max-age=0`. Zero bundled static fallback text is exposed.
   - Verified via automated wire-level tests (`apps/dashboard/tests/public-privacy-policy-resilience.test.ts`) and successful production build (`next build`).

6. **Append-Only Publication History, Storage Scenarios & Migration RLS**:
   - **Verified `AuditLog` Schema Facts**:
     - `id`: Managed by PostgreSQL `BIGSERIAL` autoincrement sequence (`BigInt` in Prisma).
     - `actor_id`: Nullable UUID referencing `users(id) ON DELETE SET NULL`.
     - `entity_id`: Nullable PostgreSQL `UUID`. Because `'privacy-policy'` is a text slug, `entity_id` MUST be stored as `NULL` for system policies, with `slug` captured in `new_state->>'slug'`.
   - **Distinguishing Migration Execution from Application Writes**:
     - *Application Writes*: Policy mutations in `apps/api` execute as role `nst_app` inside `withUserContext(adminId, ...)`, setting `app.user_id = adminId`. This satisfies the active RLS policy `Allow INSERT on audit_logs` (`actor_id = current_user_id()`).
     - *Migration Seed Execution*: Migrations execute via Prisma migrate CLI using `DATABASE_URL` (or `ADMIN_DATABASE_URL`). Whether the migration seed satisfies `Allow INSERT on audit_logs` depends on the actual migration database role, whether `FORCE ROW LEVEL SECURITY` is enabled on `audit_logs`, and whether `current_user_id()` is NULL. Migration RLS compliance remains an unverified assumption until the exact migration execution context is tested during implementation.
   - **Scenario-Based Storage Growth Estimates (Not an Absolute Bound)**:
     - Storage calculations represent scenario-based estimates, NOT an absolute schema-enforced bound, because event counts are not constrained by the schema:
       - *Scenario A (Realistic: 15 KB policy, 4 publishes/year, 500 drafts/year)*: ~378 KB/year (~1.89 MB over 5 years).
       - *Scenario B (Active Iteration: 30 KB policy, 12 publishes/year, 1,000 drafts/year)*: ~1.24 MB/year (~6.2 MB over 5 years).
       - *Scenario C (Max Payload Stress: 100 KB max content, 12 publishes/year, 2,000 drafts/year)*: ~3.46 MB/year uncompressed (~17.3 MB over 5 years).
     - Actual disk usage is affected by PostgreSQL TOAST compression (PGLZ/LZ4 reducing Markdown by 40–60%), B-Tree index overhead on `actor_id` and `created_at`, WAL volume, backup size, and multi-byte UTF-8 Unicode encoding.

7. **Zero-Dependency Markdown Engine Reuse**:
   - Reuse existing in-house components: `MarkdownRenderer` and `MarkdownEditor`.
   - Enforce pure React Virtual DOM rendering (no `dangerouslySetInnerHTML`), strict XSS immunity for raw HTML, and protocol whitelist validation via `sanitizeUrl()`.

8. **Mobile App Deep Link Compatibility**:
   - Maintain the existing public URL path `/privacy-policy` referenced by the mobile app in `apps/mobile/src/infrastructure/config.ts` (`EXPO_PUBLIC_PRIVACY_POLICY_URL`).

## Consequences

* **Positive:**
  - Platform Admins can update legal policies in real-time with zero engineering involvement or container redeployments.
  - Multi-replica Kubernetes deployments remain consistent via PostgreSQL source of truth.
  - Drafts remain confidential until explicitly published.
  - Full audit trail compliance for App Store / institutional reviews with full-text reconstruction.
  - Zero new third-party npm dependencies; reuses existing hardened Markdown rendering and editor components.
  - Non-silent error states prevent caching of outage pages.
  - Realistic scenario storage estimates confirm modest database growth (< 5 MB/year under stress).
* **Negative:**
  - Requires unfreezing the database schema and adding one database migration touching `users` and creating `system_policies`.
  - Introduces new route handlers in `apps/api`.
  - Requires optimistic locking to prevent concurrent overwrite collisions among Platform Admins.
  - In Next.js App Router, returning wire-level HTTP 503 requires resolving the `page.tsx` vs `route.ts` architecture.
  - Multi-replica pod caching without shared cache can experience compounding staleness up to 180–240 seconds under worst-case boundary conditions.

## Alternatives Considered

* **Next.js Container Filesystem (`fs.writeFile`)**: Rejected. Next.js runs as 2 stateless replicas without persistent storage. Changes would be inconsistent between pods and completely wiped upon container restart or deployment.
* **Third-Party Headless CMS (Contentful, Strapi, Sanity)**: Rejected. Violates project self-hosting requirements on the NST K3s Cluster and introduces external operational dependencies and latency.
* **Git-Backed CMS (Decap / Netlify CMS pushing to GitHub)**: Rejected. Requires complex GitHub Personal Access Token management, introduces Git commit race conditions, and requires webhook synchronization pipelines.
* **Store inside existing `AuditLog` or `Announcement` tables**: Rejected. `AuditLog` is an append-only event ledger and `Announcement` is deferred with an incompatible lifecycle. Repurposing unrelated tables violates schema domain boundaries.
* **Silent Static Content Fallback**: Rejected. Falsely presents expired terms as currently effective law, violating legal transparency.
* **Coexisting `page.tsx` and `route.ts` in same segment**: Rejected. Next.js App Router strictly forbids route handlers and pages at the same route path.

## Related Documents
* [Privacy Policy Management Specification (docs/features/admin/privacy-policy-management.md)](../../features/admin/privacy-policy-management.md)
* [API Routing Matrix (docs/api/02-api-routing-matrix.md)](../../api/02-api-routing-matrix.md)
* [Data Contract (docs/frontend/shared/DATA_CONTRACT.md)](../../frontend/shared/DATA_CONTRACT.md)
* [Database Architecture (docs/engineering/adrs/adr-960-database-architecture.md)](./adr-960-database-architecture.md)
* [Use Audit Logs (docs/engineering/adrs/adr-029-use-audit-logs.md)](./adr-029-use-audit-logs.md)
