# Student Web Application UX Specification

## Purpose
This directory contains the comprehensive, implementation-ready UX architecture and workflows for the NST Events Student Web Application. 

The student experience does not exist in code yet. These documents specify exactly how the student application must behave based strictly on the capabilities already implemented in the backend services.

## Student Target User
The target user is an enrolled student at the university (e.g., ADYPU). Their goal is to discover campus events, participate in clubs, compete on the leaderboard, and manage their own attendance and registrations. They do not care about "data models" or "administration"; they care about commitments, social activities, and action.

## Backend Source-of-Truth Methodology
**The backend defines what is possible. The UX defines how students access those capabilities.**
Every workflow, screen, and state documented here is mapped to existing Prisma models, PostgreSQL enums, Express routes, and RLS policies. No new backend functionality has been invented to solve a UX problem. If the backend cannot support an ideal UX interaction, a constraint is documented and the UX adapts to reality.

## Directory Structure
- **Root**: Core architecture, principles, backend constraints, and the overarching information architecture.
- **`/journeys`**: End-to-end user journeys (e.g., New Student, Event Participation).
- **`/workflows`**: Detailed, step-by-step documentation for every major feature (e.g., Registration, Attendance).
- **`/state-machines`**: Mermaid state diagrams for complex lifecycles (Teams, Disputes).
- **`/screens`**: Detailed interaction specifications for every UI view.

## Core Navigation
1. **Home**: The student's dynamic action center.
2. **Campus**: Discovery hub for events, clubs, and the leaderboard.
3. **My Events**: The student's commitment and participation history.
4. **Profile**: Settings and notification preferences.
