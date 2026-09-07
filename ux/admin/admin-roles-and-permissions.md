# Admin Roles and Permissions

This document outlines the administrative roles within NST Events and comprehensively maps their capabilities across the platform. The documentation is reverse-engineered directly from the `apps/dashboard/lib/auth-helpers.ts` implementation and the frontend components.

---

## Authorization Architecture

NST Events uses a dual-layered authorization system:
1. **Global Roles**: Defined on the user profile (`PLATFORM_ADMIN`, `FACULTY_ADMIN`, `STUDENT`). These grant broad, platform-wide permissions.
2. **Club Roles**: Defined contextually via the `club_memberships` table (`CLUB_ADMIN`, `CORE_MEMBER`, `FACULTY_MENTOR`, `MEMBER`). These grant targeted permissions scoped only to a specific club and its events.

**Important Note on Enforcement**: The matrix below documents how the frontend UI displays or hides capabilities using helpers from `lib/auth-helpers.ts`. However, these are strictly UI guards. The ultimate source of truth and enforcement is the backend API and PostgreSQL Row-Level Security (RLS) policies.

---

## Role Definitions

### Global Roles

| Role | Description |
|------|-------------|
| **`PLATFORM_ADMIN`** | Superuser. Has unrestricted access to all platform settings, all clubs, all events, user management, and technical monitoring (Audit Logs, Queues). |
| **`FACULTY_ADMIN`** | Institutional administrator. Can manage the academic catalog, oversee all events, and view the student directory. Cannot modify platform admins, view system logs, or perform destructive user operations. |
| **`STUDENT`** | The default role for all end-users. Administrative capabilities are entirely dependent on their contextual Club Roles. |

### Club Roles (Contextual)

These roles only apply within the context of a specific club and the events owned by that club.

| Role | Description |
|------|-------------|
| **`FACULTY_MENTOR`** | A faculty member assigned to oversee a club. Primarily responsible for reviewing/approving events and managing attendance disputes. Can also lock events and manage attendance. |
| **`CLUB_ADMIN`** | The highest student authority within a club. Can manage club details, members, and all events. |
| **`CORE_MEMBER`** | An organizational member of a club. Can manage events and operations but cannot modify club settings or memberships. |
| **`MEMBER`** | A standard club member. No administrative privileges. |

---

## Capability Matrix

The following tables map specific UI actions to the roles required to perform them, along with the specific `auth-helpers.ts` function that guards the action.

### Event Management (`/events/[id]`)

| Action | Guard Function | Required Role(s) | Notes |
|--------|----------------|------------------|-------|
| **Edit Draft Event** | `canManageEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER` | Event must be in `DRAFT` state. |
| **Submit for Approval** | `canManageEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER` | Changes state to `PENDING_APPROVAL`. |
| **Direct Publish (Bypass)** | `isPlatformAdmin`, `isFacultyAdmin` | `PLATFORM_ADMIN`, `FACULTY_ADMIN` | Direct DRAFT → PUBLISHED transition. |
| **Approve/Reject Event** | `canApproveEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `FACULTY_MENTOR` | Approvals page or Event detail. |
| **Lock/Unlock Event** | `canLockEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER`, `FACULTY_MENTOR` | Manually freezes operations. |
| **Manage Attendance** | `canManageEvent`, `canApproveEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER`, `FACULTY_MENTOR` | Create sessions, generate QR, end sessions. |
| **Mark Manual Attendance**| `canMarkAttendanceManually`| `PLATFORM_ADMIN` | Bypass QR scanning via UUID. |
| **Manage Teams** | `canManageEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER` | Remove members, transfer leadership, cancel teams. |
| **Manage Registrations** | `canManageEvent` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN`, `CORE_MEMBER` | View registrations list. |

### Club Management (`/clubs/[id]`)

| Action | Guard Function | Required Role(s) | Notes |
|--------|----------------|------------------|-------|
| **Edit Club Details** | `canManageClubDetails` | `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `CLUB_ADMIN` | Name, description, banner. |
| **Change Club Status** | `isPlatformAdmin` | `PLATFORM_ADMIN` | Active, Inactive, Dissolved. |
| **Manage Memberships** | `canManageClubMemberships`| `PLATFORM_ADMIN`, `CLUB_ADMIN` | Add/Remove members, change club roles. |

### User Management (`/admin/users`)

| Action | Guard Function | Required Role(s) | Notes |
|--------|----------------|------------------|-------|
| **View Directory** | `canViewStudentDirectory` | `PLATFORM_ADMIN`, `FACULTY_ADMIN` | Access the `/admin/users` page. |
| **Add/Remove Student** | `canManageStudentDirectory`| `PLATFORM_ADMIN` | Revokes platform access (soft delete). |
| **Change Academic Batch**| `canChangeAcademicBatch` | `PLATFORM_ADMIN`, `FACULTY_ADMIN` | Target *must* be a pure `STUDENT` with no admin/club roles. |
| **Change Global Role** | `canChangeGlobalRole` | `PLATFORM_ADMIN` | Target cannot be self. |
| **Force Logout User** | `canRevokeUserSessions` | `PLATFORM_ADMIN` | Revokes all active refresh sessions. |

### Platform Operations (`/admin/*`)

| Action | Guard Function | Required Role(s) | Notes |
|--------|----------------|------------------|-------|
| **Recalculate Leaderboard**| `canRecalculateLeaderboard`| `PLATFORM_ADMIN` | Triggers materialized view refresh on DB. |
| **View Audit Logs** | `isPlatformAdmin` | `PLATFORM_ADMIN` | Access `/admin/audit-logs`. |
| **Monitor Queues / DLQ** | `isPlatformAdmin` | `PLATFORM_ADMIN` | Access `/admin/queues`. |
| **View Academic Catalog** | `canViewAcademicCatalog` | `PLATFORM_ADMIN`, `FACULTY_ADMIN` | View programs and batches. |
| **Manage Academic Catalog**| `canManageAcademicCatalog` | `PLATFORM_ADMIN` | Create/edit programs and batches. |

---

## Edge Cases and Protections

1. **Self-Demotion Protection**: `canChangeGlobalRole` strictly returns `false` if the `actor.id === target.id`. A Platform Admin cannot accidentally remove their own privileges.
2. **Academic Batch Integrity**: `canChangeAcademicBatch` enforces that only pure students (no global admin roles, no club admin roles) can have their academic batch altered by an administrator.
3. **Session Revocation**: `canRevokeUserSessions` protects against self-revocation (`actor.id === target.id`).
4. **Last Admin Rule**: While the UI allows selecting demotion for a Platform Admin, the backend API enforcing the `LAST_PLATFORM_ADMIN` rule will reject the transaction if they are the only remaining active Platform Admin. The frontend handles this gracefully with a specific error message.
