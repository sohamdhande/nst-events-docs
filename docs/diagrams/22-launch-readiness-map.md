# 22 NST-Connect Architecture Map

This final mega-diagram provides a holistic view of the NST-Connect architecture, suitable for stakeholder presentation. It aggregates roles, modules, APIs, and databases into a single, comprehensive view.

```mermaid
graph TD
    subgraph Users
        Student[Student]
        ClubAdmin[Club Admin]
        FacultyMentor[Faculty Mentor]
        Admin[Platform Admin]
    end

    subgraph Client Tier
        Mobile[Mobile App (Expo)]
        Web[Web Dashboard (Next.js)]
    end

    subgraph Core Modules
        AuthM[Authentication]
        EventsM[Events]
        AttendanceM[Attendance & QR]
        LeaderboardM[Leaderboard]
        NotificationsM[Notifications]
        ClubsM[Club Management]
    end

    subgraph Service Tier (Supabase)
        API[PostgREST API & Edge Functions]
        RLS[Row Level Security]
    end

    subgraph Data Tier
        DB[(PostgreSQL)]
        Storage[(Object Storage)]
    end

    Student --> Mobile
    ClubAdmin --> Mobile
    ClubAdmin --> Web
    FacultyMentor --> Web
    Admin --> Web

    Mobile --> AuthM
    Mobile --> EventsM
    Mobile --> AttendanceM
    Mobile --> LeaderboardM
    Mobile --> NotificationsM

    Web --> AuthM
    Web --> EventsM
    Web --> AttendanceM
    Web --> LeaderboardM
    Web --> NotificationsM
    Web --> ClubsM

    AuthM --> API
    EventsM --> API
    AttendanceM --> API
    LeaderboardM --> API
    NotificationsM --> API
    ClubsM --> API

    API --> RLS
    RLS --> DB
    RLS --> Storage
```
