# Attendance Dispute System

## Purpose
Legitimate attendance can fail due to hardware malfunctions (dead battery, broken camera), GPS failure, or QR code expiration. The Attendance Dispute System ensures no student is unfairly penalized while maintaining absolute audit integrity.

## Workflow
1. **Submission**: Student uses the Mobile App to submit an "Attendance Dispute" within 24 hours of the event ending.
2. **Evidence**: Student provides a reason and can upload evidence (e.g., photo of themselves at the event, Slack message).
3. **Queue**: Dispute enters the "Attendance Dispute Review Queue" on the Dashboard.
4. **Review**: Club Admin or Faculty Mentor reviews the evidence.
5. **Resolution**: Admin clicks `Approve` or `Reject`.

## Permissions
* **Submit**: Student (Mobile)
* **Review/Resolve**: Club Admin, Faculty Mentor, Faculty Admin (Dashboard)

## Evidence Handling
Images are uploaded securely to File Storage (TBD) with RLS restricting access strictly to the reviewing admins.

## Audit Logging
Resolving a dispute triggers an `ATTENDANCE_OVERRIDE` and `RESOLVE_DISPUTE` audit log, explicitly recording the admin's `user_id` and the provided evidence link to prevent fraudulent approvals.
