# Event Management Flow

## State Machine Overview
1. **Draft**: Initial creation by Club Admin / Faculty.
2. **Pending Approval**: Locked for edits. Awaiting Faculty Mentor/Admin.
3. **Published**: Visible to students. Accepting registrations.
4. **Archived**: Past event. Read-only.

## Multi-Club Ownership
Events can be mapped to multiple clubs during the Draft phase. Approvals require sign-off from the primary club's Faculty Mentor.

## Faculty Event Creation
Faculty Mentors and Faculty Admins possess the `Events → Create Event` capability directly. They are not merely review-only actors.

## Multi-Club Association Management
NST-Connect models `Event ↔ Multiple Clubs`. 
* **Club Association Selector**: Dashboard UI component used during drafting.
* **Ownership Rules**: The creator designates one `Primary Club` (which dictates the Faculty Mentor approval routing) and multiple `Secondary Clubs`.
* **Associated Club List**: Rendered prominently on the Event Details screen.
