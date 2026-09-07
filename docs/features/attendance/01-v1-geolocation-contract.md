# DOCS-04 — V1 ATTENDANCE GEOLOCATION & ROTATING QR CONTRACT

## A. Product Objective
The objective is reasonable attendance verification with low student friction for first-year students earning credit hours through hackathons, workshops, events, and similar activities. It is NOT a high-security exam/proctoring system. The backend remains authoritative for attendance decisions.

## B. Attendance Session Architecture
The canonical V1 attendance model:
1. Organizer starts Attendance Session
2. Backend captures organizer's current GPS
3. Attendance Session stores venue coordinates
4. QR rotates every 30 seconds
5. Student scans QR
6. Mobile app obtains student's current GPS
7. Mobile sends QR/session token + GPS evidence
8. Backend validates session / QR / student / geofence
9. Backend records attendance

The mobile app must NEVER make the final attendance decision.

## C. Event vs Session Location
- **Event**: The human-readable `location_name` remains the event-level location description. The Event itself does NOT need manually configured latitude/longitude for V1 attendance geofencing.
- **Attendance Session**: Contains the authoritative geofence latitude, authoritative geofence longitude, geofence radius, and session start/end timing. The Attendance Session is the authoritative geofence reference.

## D. Organizer Location Capture
When an organizer starts an Attendance Session:
1. Organizer device requests location permission.
2. Device obtains current GPS coordinates.
3. Coordinates are sent to the backend.
4. Backend validates the location evidence.
5. Backend creates the Attendance Session using that location.
6. Session geofence reference becomes fixed for that session.

The organizer should not manually type latitude/longitude. A map picker is NOT required for V1.

## E. Student Location Capture
When a student scans the QR, the mobile app obtains the student's current location and sends:
- latitude
- longitude
- location timestamp
- location accuracy where available
- attendance session / QR token

The mobile app does NOT calculate attendance eligibility. The backend performs the distance calculation.

## F. Backend Decision Model
The backend evaluates:
1. Valid Attendance Session
2. Session currently open
3. Valid QR/session token
4. Student is explicitly `REGISTERED` (Waitlisted/Cancelled/Soft-deleted are strictly denied)
5. If Event Audience is `SPECIFIC_BATCHES`, the student's *current* Academic Batch must match an allowed batch
6. Geolocation Integrity:
   - Coordinates MUST NOT be NULL, NaN, Infinity, or out of bounds (rejected with `LOCATION_UNAVAILABLE` or `INVALID_LOCATION`).
   - `gps_accuracy` MUST NOT be NULL and MUST be <= 100 meters (rejected with `LOCATION_UNRELIABLE`).
   - `mock_location_detected` MUST NOT be true (rejected with `MOCK_LOCATION_REJECTED`).
7. Student location is sufficiently recent
8. Geofence condition: `distance(student_location, session_location)` is compared against `session.geofence_radius`.
   - If within the permitted radius → Attendance accepted.
   - Otherwise → `OUTSIDE_GEOFENCE`.
9. Student has not already been marked present

### F.1. Machine-Readable Error Contract (SQLSTATE)
V1 enforces a strict machine-readable error contract using PostgreSQL `SQLSTATE` codes. The backend must map these database exceptions into semantic application errors, ensuring that internal database messages (`SQLERRM`) are never leaked to the client.

- `U0001` → `UNAUTHORIZED`
- `U0002` → `WAITLISTED`
- `U0003` → `NOT_REGISTERED`
- `U0004` → `REGISTRATION_NOT_ELIGIBLE`
- `U0005` → `SESSION_CLOSED`
- `U0006` → `EVENT_LOCKED`
- `U0007` → `OUTSIDE_GEOFENCE`
- `U0008` → `MOCK_LOCATION_REJECTED`
- `U0009` → `LOCATION_UNAVAILABLE`
- `U0010` → `INVALID_LOCATION`
- `U0011` → `LOCATION_UNRELIABLE`
- `U0012` → `ACADEMIC_PROFILE_MISSING`
- `U0013` → `ACADEMICALLY_INELIGIBLE`
- `U0014` → `SIGNATURE_ALREADY_CONSUMED`

Any unknown database error MUST be logged internally and masked as a generic HTTP 500 error (`An unexpected error occurred`) to prevent leakage. For offline synchronization, unknown errors are masked as `ATTENDANCE_ERROR`.

### F.1. Dynamic Academic Eligibility
For `SPECIFIC_BATCHES` events, academic eligibility is verified dynamically at the exact moment of attendance creation via the `check_attendance_eligibility` database primitive. 
If a student registered successfully but their Academic Batch was subsequently altered to an ineligible batch, their attendance will be strictly denied. Waitlisted students will also be strictly denied.

## G. QR Rotation
The QR code rotates every 30 seconds. QR rotation MUST NOT be documented as sufficient by itself.

## H. QR Expiration
Each QR token must have an explicit validity window enforced by the backend.
Example conceptual behavior:
QR #1: 10:00:00 → 10:00:30
QR #2: 10:00:30 → 10:01:00
When QR #1 expires, holding/scanning QR #1 later MUST NOT successfully create attendance. Server time is authoritative. Student device time must not determine QR expiration. The documentation must explicitly distinguish QR rotation from QR token expiration.

## I. Duplicate / Replay Handling
The backend must prevent duplicate attendance for the same student/session. Repeated submissions must be idempotent or return an appropriate already-recorded attendance result according to the existing API contract. Do not create duplicate Attendance records.

## J. GPS Accuracy
V1 uses GPS accuracy as a reliability signal. The student location payload MUST include valid accuracy.
**Locked Threshold**: `gps_accuracy` must be <= 100 meters. Missing/NULL accuracy is intentionally rejected for all clients to prevent bypasses. If the value exceeds 100m, attendance is rejected with `LOCATION_UNRELIABLE`.

## K. Location Failure
- **Location permission denied**: attendance cannot complete through geofence verification
- **Location unavailable**: attendance cannot complete through geofence verification
- **GPS reading too stale**: reject or request a fresh reading according to API contract
- **GPS accuracy insufficient**: handle as a location reliability failure according to the backend policy

Do not silently convert these cases into attendance success.

## L. Network Failure
When a student scans with poor connectivity, request times out, request is retried, or duplicate request arrives:
The backend must remain idempotent. A retry must not create duplicate attendance. Continuous network connectivity after the initial successful attendance submission is not required.

## M. Device/Fraud Scope
V1 does not attempt to completely prevent rooted-device / OS-level GPS spoofing.
The V1 defense model is:
QR validity + session validity + student eligibility + fresh GPS + server-side geofence + duplicate prevention.
Advanced device-collision analytics are DEFERRED unless already implemented as part of an existing mandatory security contract. Do not make sophisticated device-fraud detection a prerequisite for attendance.

## N. Disputes / Manual Resolution
Because this attendance system supports academic credit hours, legitimate failures must have an operational recovery mechanism.
Intended recovery path:
Student fails automated attendance → Attendance dispute / administrative correction → Authorized faculty/club operator reviews → Attendance manually resolved according to permissions.

**Important:** Manual attendance (`manual_mark_attendance`) allows authorized administrators to override GPS/geofence failures, but it **MUST NOT** bypass Registration eligibility or Academic Eligibility rules. An unregistered, waitlisted, or academically ineligible student cannot be manually marked present.

Status: **IMPLEMENTED** (Supported via `POST /v1/attendance/disputes` and manual mark APIs).

## O. Session Lifecycle
Create Session → capture venue GPS → open session → QR rotates → students check in → session closes → geofence reference remains immutable for that session.

If the event moves to another physical location: Do NOT mutate the existing active session location.
Preferred behavior: End old session → start a new session → capture new location.

## P. Role Authorization
Only authorized organizers can:
- start Attendance Sessions
- end Attendance Sessions
- review Attendance
- resolve Attendance disputes

Authorized roles follow the finalized matrix: `PLATFORM_ADMIN`, `FACULTY_ADMIN`, `FACULTY_MENTOR`, `CLUB_ADMIN`, `STUDENT`.
Where event authority is Club-scoped, respect the finalized PRIMARY Club / COLLABORATING Club authorization model.

## Q. Data Model
Conceptual Attendance Session fields:
- `session_id` (CURRENT)
- `event_id` (CURRENT)
- `opened_at` (CURRENT as `openAt`)
- `closed_at` (CURRENT as `closeAt`)
- `venue_latitude` (PLANNED)
- `venue_longitude` (PLANNED)
- `geofence_radius` (CURRENT)
- `location_accuracy` (PLANNED)

## R. Privacy
Location data is collected only for attendance verification. There is no continuous location tracking.
Retention duration: **PRODUCT DECISION REQUIRED**

## S. Offline Synchronization 24-Hour Lock
For devices that complete a valid scan offline, the attendance payload must eventually be synchronized to the backend. The backend enforces a hard 24-hour lock boundary (no attendance accepted 24 hours after the event ends). 
- **Authoritative Timestamp**: The 24-hour event lock is evaluated using the **verified scan timestamp**, which is cryptographically constrained by the QR signature.
- **Synchronization Time**: The time at which the backend actually receives the payload (`now()`) is **irrelevant** for the 24-hour lock.
- **Example**: A student successfully scans an offline QR 10 minutes before the 24-hour event lock boundary, but the device only regains connectivity 72 hours later. The attendance is ACCEPTED because the *scan* occurred before the lock.

## T. Edge Cases
- **expired QR**: Reject
- **old QR screenshot**: Reject
- **QR from another event**: Reject
- **QR from a closed session**: Reject (`SESSION_CLOSED` / `U0005`)
- **duplicate scan**: Idempotent success or already recorded (`SIGNATURE_ALREADY_CONSUMED` / `U0014`)
- **duplicate request**: Idempotent handling
- **poor GPS**: Reject or request fresh reading
- **denied location permission**: Cannot verify
- **stale location**: Reject or request fresh reading
- **event moved to another venue**: Require new session
- **organizer GPS unavailable**: Cannot start session
- **student outside geofence**: Reject (`OUTSIDE_GEOFENCE` / `U0007`)
- **student exactly on boundary**: Evaluated by backend math
- **network retry**: Idempotent handling
- **student not registered**: Reject (`NOT_REGISTERED` / `U0003`)
- **student waitlisted**: Reject (`WAITLISTED` / `U0002`)
- **student changed to wrong academic batch**: Reject (`ACADEMICALLY_INELIGIBLE` / `U0013`)
- **missing academic profile**: Reject (`ACADEMIC_PROFILE_MISSING` / `U0012`)
- **session closed during submission**: Reject (`SESSION_CLOSED` / `U0005`)
- **mock location detected**: Reject (`MOCK_LOCATION_REJECTED` / `U0008`)
- **insufficient GPS accuracy**: Reject (`LOCATION_UNRELIABLE` / `U0011`)
- **missing/invalid location coordinates**: Reject (`LOCATION_UNAVAILABLE` / `U0009` or `INVALID_LOCATION` / `U0010`)
- **multiple concurrent attendance requests**: Idempotent handling
- **offline sync > 24 hours after event ends**: Accepted ONLY if the cryptographic scan timestamp was before the 24-hour boundary.

## U. V1 Non-Goals
Explicitly defer:
- continuous GPS tracking
- map-based organizer location selection
- sophisticated GPS spoof detection
- device attestation
- advanced fraud scoring
- biometric verification
- always-on location monitoring

## U. Current vs Planned vs Deferred
- **CURRENT / IMPLEMENTED**: Manual resolution API, Role Authorization, session endpoints, QR TOTP crypto (ADR-005), `geofenceRadius` (Data Contract).
- **PLANNED**: `venue_latitude`, `venue_longitude`, `location_accuracy` fields for the Attendance Session.
- **DEFERRED**: Map picker for organizers, advanced fraud scoring, continuous GPS tracking.
- **PRODUCT DECISION REQUIRED**: Exact default geofence radius, GPS accuracy threshold, data retention duration.

## V. Cross-References
- [ADR-005: Attendance Crypto and Validation Specifications](../../architecture/ADR-005-attendance-crypto-and-validation.md)
- [API Routing Matrix](../../api/02-api-routing-matrix.md)
- [Data Contract](../shared/DATA_CONTRACT.md)

## W. Exact Documentation Files Changed
- `docs/features/attendance/01-v1-geolocation-contract.md` (NEW)

## X. Remaining Product Decisions
- Geofence radius default value
- Exact GPS accuracy threshold
- Student location retention duration

## Y. Final Decision
DOCUMENTATION COMPLETE WITH PRODUCT DECISIONS REQUIRED
