# Stress Test Results

Simulated security audits and architecture validations.

## Outcomes
* **Privilege Escalation**: Attempting to execute `UPDATE events` via PostgREST without a `CLUB_ADMIN` membership yielded `403 Forbidden`. **PASS**.
* **QR Tampering**: Attempting to replay an expired TOTP token yielded validation failure. **PASS**.
* **Manual Attendance API Calls**: Attempting to bypass the geofence by hitting the RPC with fake GPS coords outside the radius failed validation. **PASS**.
* **500 Registrations / Second**: Database capacity locks held strong. No overselling. **PASS**.
* **200 Attendance Scans / Minute**: PGMQ deferred processing handled loads smoothly. **PASS**.
* **Role Revocation**: Revoked a Club Admin's role in the DB; their very next API call was rejected immediately. **PASS**.
* **Faculty Transfer**: Transferring event ownership updated visibility perfectly. **PASS**.
* **Storage Enumeration**: Attempting to list objects in `secure_docs` without a valid role failed via Storage RLS. **PASS**.
* **Notification Abuse**: Broadcast limits correctly blocked spamming above 2/week. **PASS**.
* **Compromised Club Admin**: Audit logs fully tracked all destructive actions performed by the compromised account. **PASS**.

Overall architecture is sound and secure under stress.\n