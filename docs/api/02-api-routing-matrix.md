# API Routing Matrix

All operations are handled by the **Express backend**. Clients never call the database directly. The "Execution Type" column describes what Express does internally.

| Operation | Express Internal Execution | Reason | Security Requirements |
|---|---|---|---|
| View Events | Prisma Query | Simple read; RLS filters rows by user context | Valid JWT + RBAC middleware |
| Register Event | PostgreSQL RPC (`register_event`) | Requires `SELECT FOR UPDATE` capacity lock | RBAC check + RPC atomic transaction |
| Join Team | PostgreSQL RPC (`join_team`) | Team size validation and atomic inserts | RBAC check + Validation |
| Leave Team | PostgreSQL RPC (`leave_team`) | Team leadership reassignment logic | RBAC check + Ownership Check |
| Approve Event | PostgreSQL RPC (`approve_event`) | State machine transition + audit logs | `FACULTY_MENTOR` role via RBAC |
| Reject Event | PostgreSQL RPC (`reject_event`) | State transition + notification trigger | `FACULTY_MENTOR` role via RBAC |
| Mark Attendance | Express Route Handler → PostgreSQL RPC | TOTP validation, geofence check, device collision | RBAC + Geofence + Valid Token |
| Generate QR | Express Route Handler | Cryptographic TOTP seed generation | `CLUB_ADMIN` / Core via RBAC |
| Upload Banner | Express Route Handler (pre-signed URL) — **NEEDS REVIEW** | Storage provider TBD | RBAC check before URL is issued |
| Send Notification | Express → PostgreSQL RPC → pgmq | Defers high-latency push delivery to background worker | `CLUB_ADMIN` / Platform via RBAC |
| Promote Member | PostgreSQL RPC | Inserts into `club_memberships` + audit | `CLUB_ADMIN` via RBAC |
| Assign Club Admin | PostgreSQL RPC | High-privilege role escalation | `PLATFORM_ADMIN` via RBAC |
| Create Club | PostgreSQL RPC | Complex setup (memberships, roles, audit) | `PLATFORM_ADMIN` via RBAC |
| Archive Event | PostgreSQL RPC | State machine transition | Club Admin, Faculty Mentor, Faculty Admin, Platform Admin via RBAC |
