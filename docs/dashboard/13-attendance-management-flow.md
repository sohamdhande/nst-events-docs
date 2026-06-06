# Attendance Management Flow

## QR Flow & Presentation Mode
Triggered via Operations Mode. Renders the cryptographic TOTP seed optimized for high-visibility projector display.

## Attendance Monitoring
Real-time dashboard table reflecting successful RLS inserts from the mobile clients.

## Manual Verification & Audit Logging
If a Core Member manually marks a student present, the action is securely recorded in the `audit_logs` table (as `method = 'MANUAL'`), preventing untraceable grade inflation.
