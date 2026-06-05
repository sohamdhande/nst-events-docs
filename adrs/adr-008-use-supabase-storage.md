# ADR-008: Use Supabase Storage

## Status
Accepted

## Context
Users and clubs need to upload images, banners, and documents.

## Decision
Use Supabase Storage.

## Consequences
Positive: Media should not be stored in relational tables. Integrated deeply with Supabase Auth and RLS.
Negative: Additional cost for large bandwidth.

## Alternatives Considered
* **AWS S3 directly**: Rejected to simplify infrastructure.

## Future Impact
Centralized media management with built-in access control.
