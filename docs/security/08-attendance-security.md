# Attendance Security

## Mechanisms
1. **Dynamic QR**: A Time-Based One-Time Password (TOTP) seed is generated. The mobile app scans and submits this token.
2. **HMAC Validation**: The Edge Function uses HMAC validation to ensure the TOTP is mathematically valid and generated within the last 15 seconds.
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
