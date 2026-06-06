# Enums

## `event_status`
Tracks the approval and visibility lifecycle of an event.
* `DRAFT`
* `PENDING_APPROVAL`
* `PUBLISHED`
* `ARCHIVED`

## `registration_status`
Manages attendee state and capacity limits.
* `REGISTERED`
* `WAITLISTED`
* `CANCELLED`

## `attendance_status`
The formal record of a user's presence.
* `PRESENT`
* `ABSENT`

## `event_type`
Categorizes the generic event model.
* `WORKSHOP`
* `HACKATHON`
* `GUEST_LECTURE`
* `COMPETITION`
* `SOCIAL`

## `global_role`
* `PLATFORM_ADMIN`
* `FACULTY_ADMIN`
* `STUDENT`

## `club_role`
* `FACULTY_MENTOR`
* `CLUB_ADMIN`
* `CORE_MEMBER`
* `MEMBER`

### `participation_role`
Used in `event_registrations`.
* `ATTENDEE`: User attended event successfully (Default upon check-in).
* `VOLUNTEER`: User actively helped operate event.
* `ORGANIZER`: User was part of official event organizing team.
* `SPEAKER`: User delivered session, workshop, or talk.
* `MENTOR`: User acted as mentor, reviewer, judge, or advisor.

### `competition_result`
Used in `event_results`.
* `WINNER`: 1st Place.
* `RUNNER_UP`: 2nd Place.
* `SECOND_RUNNER_UP`: 3rd Place.
* `TOP_10`: Ranked inside top ten but not podium.
* `PARTICIPANT`: Participated successfully but did not place.

## `dispute_status`
Tracks the state of an attendance dispute.
* `PENDING`
* `APPROVED`
* `REJECTED`

## `handover_status`
Manages the club leadership transfer lifecycle.
* `PENDING`
* `APPROVED`
* `REJECTED`

## `attendance_type_enum`
Defines whether an event requires one check-in or multiple.
* `SINGLE`
* `MULTI_SESSION`
