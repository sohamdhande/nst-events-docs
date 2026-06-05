# Permission Enforcement

## Frontend Permissions
The UI hides buttons and routes based on the user's role (Optimistic UX). However, this is strictly for UX purposes and is **not** a security boundary.

## Backend Permissions
Edge Functions verify the JWT and validate permissions by querying the database before taking action.

## Database Permissions (Source of Truth)
Why the database is the source of truth: If a user bypasses the UI and sends a raw HTTP request to PostgREST to delete an event, the RLS policy will natively intercept and reject the request. This eliminates complex middleware bugs.
