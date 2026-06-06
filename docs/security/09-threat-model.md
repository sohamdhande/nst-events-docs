# Threat Model

Utilizes the STRIDE methodology.

## Assets
* User PII (Emails, Names)
* Attendance Integrity (Leaderboard/Grades)
* Event Data

## Attackers
* Malicious Students (Attempting proxy attendance)
* External Actors (Attempting data scraping)

## Threats & Mitigations
* **S**poofing Identity: Mitigated by strict Supabase JWT verification. (Risk: Low)
* **T**ampering with Data: Mitigated by immutable `audit_logs` and RLS locking down `UPDATE` queries. (Risk: Low)
* **R**epudiation: Mitigated by logging `actor_id` on all critical transactions. (Risk: Low)
* **I**nformation Disclosure: Mitigated by strict RLS `SELECT` policies on private events and secure documents. (Risk: Medium)
* **D**enial of Service: Mitigated by rate limiting and `SELECT FOR UPDATE` locks preventing DB deadlocks. (Risk: Medium)
* **E**levation of Privilege: Mitigated by live role resolution. JWTs cannot be modified to forge admin rights. (Risk: Low)

## Specific Attendance Fraud Mitigations
* **GPS Spoofing**: Mitigated by `expo-location` mock detection on Android; accepted limitation on iOS.
* **QR Screenshot Sharing**: Mitigated by 15-second TOTP rotation.
* **Account Sharing**: Mitigated by `audit_metadata` device fingerprint logging.
* **Attendance Collusion**: Mitigated by audit logs and the dispute review process.

### THREAT: Account Sharing for Attendance Fraud
**Vector:**
  * Student A registers for event but does not attend.
  * Student B is physically present.
  * Student B logs into Student A's account and scans the QR.
  * Student A gets attendance marked without being present.
  * Student B logs back into their own account and scans again.
  * Both students receive attendance credit.

**Why existing controls fail:**
  * Geofence passes — Student B is physically present.
  * TOTP passes — QR is valid.
  * The server cannot distinguish who is holding the phone.

**Mitigation — Layer 1: Device collision detection (PRIMARY)**
  * Every attendance scan captures `device_id` in `audit_metadata`.
  * `mark_attendance` RPC checks: has this `device_id` already marked a different user present in this session?
  * If yes: attendance is still recorded but `audit_metadata.flagged` is set to `true` with `flag_reason = "device_collision"`.
  * A platform audit event is created.
  * Club Admin sees a flagged attendances warning in the dashboard.

**Mitigation — Layer 2: JWT single session enforcement**
  * When Student B logs into Student A's account, Supabase Auth invalidates Student A's existing JWT.
  * Student A's phone is silently logged out.
  * Creates social accountability on a small campus.

**Mitigation — Layer 3: Audit dashboard visibility**
  * Dashboard surfaces a warning when device collision is detected.
  * Club Admin can review flagged records and cross-reference with the dispute system or manually override.

**Mitigation — Layer 4: Social deterrent**
  * Leaderboard points and certificates are visible campus-wide.
  * Fraudulent attendance creates visible anomalies.

**Residual risk:**
  * Student B could use a second phone for Student A's account.
  * This is not detectable without biometric identity verification, which is out of scope for V1.

**Classification**: DETECTABLE, NOT FULLY PREVENTABLE
**Priority**: P0 — implement in `mark_attendance` RPC
