# Backend Specification Gaps

This document tracks functional gaps between the documented UX specifications and the currently available backend API contracts. These gaps were identified during UI implementation and require dedicated backend tasks to resolve.

## 1. Event Discover: Date Filtering
- **Context**: The `ux/student/screens/discover.md` spec (Section 10) defines a secondary filter for dates (e.g., Today, This week, This month).
- **Gap**: The `GET /v1/events` endpoint does not support time-window filtering via query parameters. 
- **Action Required**: Update `events.schema.ts` and `events.service.ts` to support filtering by date range (e.g., `filter_date_range=TODAY|THIS_WEEK|THIS_MONTH`, or explicit `filter_start_time`/`filter_end_time` params).

## 2. Event Discover: Availability Filtering
- **Context**: The `ux/student/screens/discover.md` spec (Section 10) defines a secondary filter for Availability (e.g., Spots available, Waitlist available).
- **Gap**: The `GET /v1/events` endpoint does not support filtering by remaining capacity. "Open for registration" is currently computed dynamically on the client side using `maxCapacity` and `registrationCount`.
- **Action Required**: Add a computed availability filter parameter to the event list query (e.g., `filter_availability=OPEN|WAITLIST`). This requires joining or subquerying against registration counts directly in the database.
