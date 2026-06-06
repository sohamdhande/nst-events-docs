# 20 Codebase Architecture

This diagram breaks down the development architecture, detailing the specific technologies used across the monorepo for the frontend clients, backend, and shared components.

```mermaid
graph TD
    subgraph Monorepo
        subgraph Apps
            Expo["Expo React Native App"]
            NextJS["Web Dashboard (Next.js)"]
        end
        
        subgraph Packages
            Types["Shared Types & Interfaces"]
            UI["Shared Design System & Components"]
        end
    end
    
    subgraph Supabase
        Auth["Supabase Auth"]
        DB[("PostgreSQL Database")]
        Functions["Edge Functions"]
        Storage["Supabase Storage"]
    end
    
    Expo --> Types
    NextJS --> Types
    
    Expo --> UI
    NextJS --> UI
    
    Expo --> Auth
    NextJS --> Auth
    
    Expo --> DB
    NextJS --> DB
    
    Expo --> Functions
    NextJS --> Functions
    
    Expo --> Storage
    NextJS --> Storage
    
    classDef app fill:#e0f2fe,stroke:#0284c7,stroke-width:2px,color:#0c4a6e;
    classDef pkg fill:#fef08a,stroke:#ca8a04,stroke-width:2px,color:#713f12;
    classDef sba fill:#dcfce7,stroke:#16a34a,stroke-width:2px,color:#14532d;
    
    class Expo,NextJS app;
    class Types,UI pkg;
    class Auth,DB,Functions,Storage sba;
```
