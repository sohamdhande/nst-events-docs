# Operations Mode Components

Components designed for the `Live Event Execution` environment.

## LiveScannerOverlay
* **Purpose**: Distraction-free UI maximizing camera feed or QR output.
* **Activation**: Only available 2 hours before/after event.
* **Permissions**: Club Admin, Core Member.

## ManualOverrideToggle
* **Purpose**: Allows Core Members to bypass QR for broken phones.
* **Interaction**: Requires explicit confirmation. Triggers `method = 'MANUAL'` audit log.

## LiveMetricHUD
* **Purpose**: Floating heads-up display showing `Checked In / Capacity`.
* **States**: Turns Warning (Orange) at 90%, Error (Red) at 100%.

## EmergencyActionTrigger
* **Purpose**: The "Lockdown Event" button.
* **Interaction**: Requires slide-to-confirm or explicit text entry to prevent accidental lockouts.

## ProximityCheckInDashboard
* **Purpose**: Visualizes PostGIS GPS validations for the active event radius.
