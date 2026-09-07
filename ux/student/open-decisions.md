# Open UX Decisions

This document tracks product/UX decisions where the backend supports multiple approaches and a specific product recommendation is required.

## 1. Prioritization in "My Events"

**Question**: How should events in the "My Events" view be prioritized? Should it be purely chronological, or grouped by status?

**Backend Support**: `GET /users/me/registrations` returns all registrations with their statuses (`REGISTERED`, `WAITLISTED`).

**Recommendation**: Group by status using tabs: "Upcoming", "Waitlisted", and "Past".
**Reason**: A student's mental model separates "Things I am definitely going to" from "Things I might go to if space opens." Mixing waitlisted events chronologically with confirmed events causes anxiety.
**Tradeoff**: Requires clicking tabs rather than a single unified scroll.

## 2. Leaderboard Score Details

**Question**: Should the Leaderboard expose the exact source of a student's points (e.g., "5 points for Workshop A, 10 points for Hackathon B")?

**Backend Support**: The `leaderboard_scores` table contains a `reason` and `source_id`, but the `/leaderboard/students` endpoint primarily returns aggregated rank and total points from a materialized view.

**Recommendation**: Do not expose point-by-point history in the V1 UI. Show only Total Points and Rank.
**Reason**: Calculating live point history requires complex joins against the source tables, circumventing the performance benefits of the materialized view.
**Tradeoff**: Students cannot audit exactly how they earned their score.

## 3. Notification Grouping

**Question**: Should notifications be grouped by type (Invites, Updates) or purely chronological?

**Backend Support**: `GET /notifications` returns a chronologically sorted list, but includes a `type` string field.

**Recommendation**: Purely chronological grouping, with unread items styled distinctly.
**Reason**: Students primarily use notifications as an activity feed. Sorting by type hides recent critical alerts below older items of a different type.
**Tradeoff**: Team invites might get buried if a lot of announcements are sent simultaneously. (Mitigated by putting Pending Invites on the Home dashboard).
