# Security Freeze V1

## Status: FROZEN
## Version: 1.0
## Date: 2026-06-06

## Security Assumptions
* Supabase Auth infrastructure remains uncompromised.
* Faculty and Admins protect their accounts via external SSO 2FA (Google Workspace).
* Database superuser keys are never exposed to the client.

## Accepted Risks
* Basic GPS spoof detection can be bypassed by highly advanced attackers compiling custom kernels.
* No device cryptographic signing exists to prove the origin of API requests.

## Known Limitations
* Rate limits are IP/User based rather than behavioral.

## Future Improvements
* Implementation of anomaly detection (e.g., detecting if a user marks attendance at two distant locations within 1 minute).
* Integrating advanced third-party anti-spoofing SDKs.

## Resolved Issues (Phase 1-4)
* Added `SECURITY DEFINER` documentation for `resolve_attendance_dispute` and `force_transfer_leadership`.
* Defined `notification_preferences` RLS policy: students can only read and update their own preferences.
