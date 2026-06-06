# 20 Codebase Architecture

This diagram breaks down the development architecture, detailing the specific technologies used across the monorepo for the frontend clients, backend, and shared components.

```mermaid
graph TD
    subgraph Monorepo
        subgraph Apps
            Expo[Expo React Native App]
            NextJS[Next.js Web Dashboard]
        end
        
        subgraph Packages
            Types[Shared Types & Interfaces]
            UI[Shared Design System & Components]
        end
    end
    
    subgraph Supabase
        Auth[Supabase Auth]
        DB[(PostgreSQL Database)]
        Functions[Edge Functions]
        Storage[Supabase Storage]
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
```
