# API Routing Matrix

| Operation | Execution Type | Reason | Security Requirements |
|---|---|---|---|
| View Events | Direct Query | Simple read, filtered by RLS | Valid JWT |
| Register Event | RPC | Requires checking capacity via `SELECT FOR UPDATE` | RLS + Live Capacity Check |
| Join Team | RPC | Requires team size validation and atomic inserts | RLS + Validation |
| Leave Team | RPC | Requires team leadership reassignment logic | RLS + Ownership Check |
| Approve Event | RPC | Triggers state machine transitions and audit logs | `FACULTY_MENTOR` role |
| Reject Event | RPC | State transition + notifications | `FACULTY_MENTOR` role |
| Mark Attendance | Edge Function / RPC | Needs cryptographically secure TOTP validation | Geofence + Valid Token |
| Generate QR | Edge Function | Compute cryptographic TOTP seed securely | `CLUB_ADMIN` / Core |
| Upload Banner | Direct Storage | Native multipart upload handling | Storage RLS |
| Send Notification | RPC → PGMQ | Defers high-latency push delivery | `CLUB_ADMIN` / Platform |
| Promote Member | RPC | Inserts into `club_memberships` + audit | `CLUB_ADMIN` |
| Assign Club Admin| RPC | High-privilege role escalation | `PLATFORM_ADMIN` |
| Create Club | RPC | Complex setup (creates tables/folders/roles) | `PLATFORM_ADMIN` |
| Archive Event | RPC | State machine transition | `CLUB_ADMIN` + Date Check |\n