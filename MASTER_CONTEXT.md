# Master Context: NST-Events

## Project Summary
NST-Events is a production-grade campus platform for Newton School of Technology (NST), Pune. Target deployment is for 3000+ active students. The platform centralizes Event Management, Club Operations, dynamic QR-based Attendance, Announcements, and comprehensive Analytics. The initial build timeline is 2 months with a 6-developer team.

## Current Architecture
* **Frontend**: React Native (Expo) - Mobile application for students and faculty.
* **Web Dashboard**: Next.js - Administrative and operational dashboard.
* **Backend**: Supabase - Serverless backend providing API and real-time capabilities.
* **Database**: PostgreSQL (hosted via Supabase) - Relational data store.
* **Authentication**: Google OAuth restricted to `@adypu.edu.in` and `@newtonschool.co` domains.
* **Authorization**: PostgreSQL Row Level Security (RLS) ensuring strict data access.
* **Storage**: Supabase Storage for media, assets, and document uploads.
* **Notifications**: Expo Push Notifications.

## User Roles
* **Student**: Base role; can view events, register, and mark attendance.
* **Member**: Associated with specific clubs.
* **Core Member**: Elevated club privileges.
* **Club Admin**: Full management over a specific club's events and roster.
* **Faculty Mentor**: Faculty oversight for clubs.
* **Faculty Admin**: Administrative capabilities over academic/faculty-led events.
* **Platform Admin**: Superuser with complete system access.

## Accepted Decisions
* [ADR-001] Use of Supabase for Backend-as-a-Service to accelerate development.
* [ADR-002] React Native (Expo) chosen over native development for cross-platform velocity.
* [ADR-003] Dynamic QR Codes for attendance to prevent proxy check-ins.



## Newly Finalized Decisions (Active)
* **Event Type Architecture**: Single `events` table with `event_type` and `metadata (jsonb)`. (Accepted)
* **Event Approval Workflow**: State machine lifecycle: `DRAFT` → `PENDING_APPROVAL` → `PUBLISHED` → `ARCHIVED`. (Accepted)
* **Faculty Permissions**: Faculty can create events, approve events, view attendance, view analytics, and view reports. They CANNOT mark attendance, generate QR, modify attendance, or change platform settings.
* **Leaderboard**: Individual and Club leaderboards are separated. Points awarded only for meaningful participation. No points for mere registration.

## Open Questions
* Integration strategy for legacy campus systems (if applicable).
* Offline capabilities and synchronization for the mobile app during poor connectivity.
* Rate limits and usage caps for the Supabase backend.

## Active Sprint
* **Sprint 1: Foundation & Authentication**
  * Set up code repositories.
  * Implement Google OAuth.
  * Provision Supabase environments.

## Team Structure
* **Total Size**: 6 Developers
* **Roles**: 
  * Frontend (Mobile)
  * Frontend (Web)
  * Backend/Database
  * UI/UX Design
  * QA/Testing
  * Project/Product Manager
