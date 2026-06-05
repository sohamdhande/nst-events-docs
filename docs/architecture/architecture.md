# Architecture Decision Summary

This document summarizes the core architectural pillars of the NST-Events platform.

## High-Level Architecture
* **Frontend Mobile**: Expo React Native (Student-first experience).
* **Frontend Web**: Next.js (Administration, Event Management, Analytics).
* **Backend BaaS**: Supabase.
* **Database**: PostgreSQL with Row Level Security (RLS) enforcing the Permission Matrix.
* **Serverless Logic**: Supabase Edge Functions (Deno).
* **Storage**: Supabase Storage for assets.

## Architectural Decisions Summary
All fundamental architectural patterns have been documented in `/adrs`. 
The vast majority of structural decisions have been **Accepted**.

* **Accepted**: 34 records (including core stack, authorization logic, approval workflows, generic event models, and basic spoofing detection).
* **Rejected**: 5 records (including mobile admin workflows, static QR codes, and device cryptographic signing in V1).
* **Under Review**: 0 records. All major V1 architectural roadblocks have been resolved.

Please refer to `adrs/README.md` for a complete index of all decisions.
