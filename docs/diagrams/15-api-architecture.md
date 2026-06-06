# 15 API / RPC Architecture

This diagram shows the flow of requests from client applications through the Supabase Backend and RPC layers to the underlying database, utilizing Row Level Security (RLS).

```mermaid
graph TD
    Mobile["Mobile App (Expo)"]
    Dashboard["Dashboard (Next.js)"]
    
    Mobile -->|GraphQL/REST API| API["Supabase PostgREST API"]
    Dashboard -->|GraphQL/REST API| API
    
    Mobile -->|RPC Calls| RPC["Supabase RPC / Edge Functions"]
    Dashboard -->|RPC Calls| RPC
    
    API --> RLS["PostgreSQL Row Level Security (RLS)"]
    RPC --> RLS
    
    RLS --> Database[("PostgreSQL Database")]
```
