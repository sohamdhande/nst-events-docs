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
* **Notifications**: Expo Push Notifications integrated with `pgmq`.

## User Roles
* **Student**: Base role; can view events, register, and mark attendance.
* **Member**: Associated with specific clubs.
* **Core Member**: Elevated club privileges.
* **Club Admin**: Full management over a specific club's events and roster.
* **Faculty Mentor**: Faculty oversight for clubs.
* **Faculty Admin**: Administrative capabilities over academic/faculty-led events.
* **Platform Admin**: Superuser with complete system access.

## Finalized Architecture Domains (FROZEN V1)
The following core architectural domains have been completely documented, strictly reviewed, and frozen for the V1 release:
1. **Database Architecture** (`docs/database/`): 21+ documents covering ER diagrams, indexes, queues, JSONB rules, PostGIS geofencing, soft-delete views, and registration `SELECT FOR UPDATE` locks.
2. **API & Security Architecture** (`docs/api/`, `docs/security/`): Strict "Zero Trust" model, PGMQ integration, edge functions, live role resolution without JWT claim storage, and comprehensive Table Policy Matrices.
3. **Mobile Navigation Architecture** (`docs/mobile/`): Context-aware Home feed driven by `get_home_feed()` delta-sync, 5-tier Priority System, Pessimistic Registration, Waitlist states, and exact Domain Governance (Profile vs Campus).
4. **Events & Attendance Models**: Defined generic JSONB `events`, 4-step approval workflows, Dynamic QR TOTP, and HMAC validations.

## Open Questions
* Integration strategy for legacy campus systems (if applicable).
* Offline capabilities and synchronization for the mobile app during poor connectivity (Deferred to post-V1).

## Active Phase
* **Architecture Phase**: Completed. The documentation repository acts as the complete, implementation-grade blueprint.
* **Implementation Phase**: Ready to commence `.sql` migrations, Edge Function development, and React Native UI prototyping based on the frozen documents.

## Team Structure
* **Total Size**: 6 Developers
* **Roles**: 
  * Frontend (Mobile)
  * Frontend (Web)
  * Backend/Database
  * UI/UX Design
  * QA/Testing
  * Project/Product Manager
