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
