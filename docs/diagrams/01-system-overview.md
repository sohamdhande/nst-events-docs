# 01 System Overview

This diagram illustrates the high-level system overview of the NST-Connect platform, highlighting the interactions between different user roles, client applications, and backend services.

```mermaid
graph TD
    subgraph Users
        Student[Student]
        ClubAdmin[Club Admin]
        FacultyMentor[Faculty Mentor]
        FacultyAdmin[Faculty Admin]
        PlatformAdmin[Platform Admin]
    end

    subgraph Client Applications
        MobileApp[Mobile App (Expo React Native)]
        Dashboard[Web Dashboard (Next.js)]
    end

    subgraph Backend Infrastructure
        Backend[Backend API / Supabase]
        Database[(PostgreSQL Database)]
        Storage[(Supabase Storage)]
        Notifications[Notification Service / pgmq]
    end

    subgraph External Integrations
        Slack[Slack Integration]
    end

    Student -->|Uses| MobileApp
    ClubAdmin -->|Uses| Dashboard
    ClubAdmin -->|Uses| MobileApp
    FacultyMentor -->|Uses| Dashboard
    FacultyAdmin -->|Uses| Dashboard
    PlatformAdmin -->|Uses| Dashboard

    MobileApp -->|API / RPC| Backend
    Dashboard -->|API / RPC| Backend

    Backend -->|Reads/Writes| Database
    Backend -->|Uploads/Downloads| Storage
    Backend -->|Publishes| Notifications
    Notifications -->|Alerts| Slack
    Notifications -->|Push| MobileApp
```
