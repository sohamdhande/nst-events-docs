# Attendance Management Flow

## QR Flow & Presentation Mode
Triggered via Operations Mode. Renders the cryptographic TOTP seed optimized for high-visibility projector display.

## Attendance Monitoring
Real-time dashboard feed updated via SSE (Server-Sent Events) pushed by the Express backend after each successful `mark_attendance` RPC write.

## Manual Verification & Audit Logging
Manual attendance overrides are performed via the `POST /events/:id/attendance/manual` endpoint, which invokes the `manual_mark_attendance` RPC. Access is restricted to Platform Admins, Faculty Admins, and authorized Club Admins (for events of their primary club). When invoked, the action bypasses QR, TOTP, and geofence validation, but is securely recorded in the `audit_logs` table (as `method = 'MANUAL'`) in strict accordance with ADR-005, preventing untraceable grade inflation.

## Flagged Attendances Panel
**Visibility**: shown when one or more `attendance_records` in the current session have `audit_metadata.flagged = true`

**Displays**:
* Warning banner: "X attendance(s) flagged for review"
* List of flagged records showing:
  * Student name
  * Scan timestamp
  * Flag reason
  * Device ID (truncated)
  * Colliding student name (if `device_collision`)

**Actions available per flagged record**:
* Mark as Reviewed (clears flag, adds review note to audit log)
* Revoke Attendance (removes attendance record, adds to audit log)
* Open Dispute (creates a manual dispute record for the student)

**Access**: Club Admin, Faculty Mentor, Faculty Admin, Platform Admin
