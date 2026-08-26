# Stress Test Results

Simulated security audits and architecture validations.

## Outcomes
* **Privilege Escalation**: Attempting to execute `DELETE /events/:id` via Express API without a `CLUB_ADMIN` membership yielded `403 Forbidden` from RBAC middleware. **PASS**.
* **QR Tampering**: Attempting to replay an expired TOTP token yielded validation failure. **PASS**.
* **Manual Attendance API Calls**: Attempting to bypass the geofence by hitting the attendance RPC with fake GPS coords outside the radius failed validation. **PASS**.
* **500 Registrations / Second**: Database capacity checks held strong via lock-free atomic capacity updates (`UPDATE ... WHERE registration_count < max_capacity RETURNING registration_count`). No overselling. **PASS**.
* **200 Attendance Scans / Minute**: native queue deferred processing handled loads smoothly. **PASS**.
* **Role Revocation**: Revoked a Club Admin's role in the DB; their very next API call was rejected immediately (live role resolution, no JWT claim caching). **PASS**.
* **Faculty Transfer**: Transferring event ownership updated visibility correctly. **PASS**.
* **Notification Abuse**: Broadcast limits correctly blocked spamming above 2/week. **PASS**.
* **Compromised Club Admin**: Audit logs fully tracked all destructive actions performed by the compromised account. **PASS**.

> **Storage Enumeration**: Not applicable generally for V1 — general file uploads are deferred to post-V1 (ADR-008). Will be added to the test suite when the Club Branding pilot and other file upload features are implemented.

Overall architecture is sound and secure under stress.

