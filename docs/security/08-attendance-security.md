# Attendance Security

## Mechanisms
1. **Dynamic QR**: A Time-Based One-Time Password (TOTP) seed is generated. The mobile app scans and submits this token.
2. **HMAC Validation**: The Express backend (`POST /attendance/mark`) uses HMAC validation to ensure the TOTP is mathematically valid and was generated within the last 15 seconds.
3. **PostGIS Geofencing**: `ST_DWithin` enforces a maximum radius from the event's location.
4. **GPS Spoof Detection**: Basic native mock-location flag checking on the mobile device.

## Abuse Prevention
* **Proxy Attendance Prevention**: The combination of rotating TOTP + Geofencing makes sharing screenshots impossible.
* **Attendance Abuse Prevention**: Rate limits (5 attempts/min) prevent brute-forcing TOTPs.

## Known Risks
* Advanced Spoofing: Custom rooted Android ROMs bypassing mock location flags.

## Device Collision Detection
Every attendance scan must include `device_id` in the request payload.
The `mark_attendance` RPC performs a device collision check before writing the attendance record.

A collision is defined as: the same `device_id` appearing in `attendance_records` for a different `user_id` in the same session.

**Collision handling**:
* Attendance is NOT rejected — the scan is recorded.
* The record is flagged in `audit_metadata`.
* An `audit_log` entry is created with action `ATTENDANCE_DEVICE_COLLISION`.
* The dashboard surfaces a warning to Club Admin.

**Rationale for not rejecting**:
Hard rejection creates a denial-of-service vector — someone could scan a QR screenshot from another student's phone to block their legitimate attendance. Flagging and auditing is safer than blocking.

## Offline Attendance Security (Operations Mode)

Offline attendance caching is **strictly restricted to organizer devices** (users with `CLUB_ADMIN` or `CORE_MEMBER` roles for the event) operating in Operations Mode. Student devices must **always** validate attendance synchronously against the live TOTP window and geofence.

**Why students cannot use offline mode**: If students could cache attendance payloads offline and submit them later, a malicious student could capture a TOTP screenshot, spoof the `scan_timestamp` in the payload, and replay it from outside the geofence hours later. This would completely defeat the dynamic QR and geofencing security model.

**Organizer trust model**: Organizers are trusted actors whose identity is captured in `audit_metadata.synced_by`. The server does not re-validate the TOTP 15-second window for organizer-submitted offline payloads, but geofence validation is still performed and the entire batch is audited with action `ATTENDANCE_OFFLINE_SYNC`.
