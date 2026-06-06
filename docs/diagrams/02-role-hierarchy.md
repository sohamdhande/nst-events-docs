# 02 Role Hierarchy

This diagram details the hierarchical structure of user roles within NST-Connect, including their inheritance and authority. Platform Admins hold supreme authority, while Faculty and Club Admins manage their respective domains down to individual students.

```mermaid
graph TD
    PlatformAdmin[Platform Admin] --> FacultyAdmin[Faculty Admin]
    PlatformAdmin --> ClubAdmin[Club Admin]
    
    FacultyAdmin --> FacultyMentor[Faculty Mentor]
    
    FacultyMentor --> ClubAdmin
    
    ClubAdmin --> CoreMember[Core Member]
    CoreMember --> Member[Member]
    Member --> Student[Student]
    
    classDef admin fill:#f9d0c4,stroke:#333,stroke-width:2px;
    classDef faculty fill:#fff2cc,stroke:#333,stroke-width:2px;
    classDef club fill:#d9ead3,stroke:#333,stroke-width:2px;
    classDef student fill:#cfe2f3,stroke:#333,stroke-width:2px;
    
    class PlatformAdmin admin;
    class FacultyAdmin,FacultyMentor faculty;
    class ClubAdmin,CoreMember club;
    class Member,Student student;
```
