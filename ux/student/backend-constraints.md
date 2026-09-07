# Backend Constraints Affecting Student UX

The UX design respects these strict backend constraints. Where a product desire clashes with the backend reality, the backend reality wins.

## 1. No Student Self-Join for Clubs
- **Constraint**: The `club_memberships` table is managed strictly by Club Admins or Platform Admins. There is no `POST /clubs/:id/join` endpoint for students.
- **UX Implication**: "My Clubs" is a read-only list. Students cannot browse a club and click "Join". 

## 2. Automatic Waitlist Promotion
- **Constraint**: The backend handles waitlist promotion automatically via queue workers when capacity opens up. There is no pending "Accept Promotion" state requiring student input.
- **UX Implication**: The UX must handle `WAITLISTED` -> `REGISTERED` state changes silently in the background and rely on Notifications / Home Action Center to alert the student.

## 3. Realtime Updates via SSE
- **Constraint**: Realtime Server-Sent Events (SSE) exist for specific domains (`/sse/events/:id/live` and `/sse/notifications/live`), but are not universally available for all lists (like leaderboard updates).
- **UX Implication**: We use realtime updates specifically on the Event Detail page and the Notification bell. We do not expect the Leaderboard or Discover pages to animate live.

## 4. Administrative States are Hidden
- **Constraint**: Events in `DRAFT` or `PENDING_APPROVAL` are hidden by RLS policies for standard students.
- **UX Implication**: The student UX does not need to handle visibility toggles or draft states. If an event 404s, it's either deleted or reverted to draft.

## 5. Strict Lock States
- **Constraint**: If an event `is_locked` is true, the API rejects all registration, cancellation, and team modification requests.
- **UX Implication**: The UX must eagerly fetch the lock state and disable/hide mutation buttons (Register, Cancel, Leave Team) to prevent frustrating 422 API errors.

## 6. Attendance Evidence Uploads
- **Constraint**: The `/attendance/disputes` endpoint expects `evidenceUrls` (array of strings). The backend does not handle multipart file uploads directly; files must be uploaded to a CDN/bucket first.
- **UX Implication**: The UX must handle the file-upload-to-CDN lifecycle in the client before submitting the dispute payload to the API.
