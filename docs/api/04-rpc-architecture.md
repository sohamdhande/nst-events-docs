# RPC Architecture

## Why RPCs Exist
Remote Procedure Calls (RPCs) encapsulate complex business logic natively within PostgreSQL functions. They are used when multiple tables must be modified in a single atomic transaction.

**Invocation path**: Clients call the Express API → Express RBAC middleware validates → Express calls the PostgreSQL RPC via Prisma's `$queryRaw` or `$executeRaw`. Clients never call RPCs directly.

## Transaction Requirements
All RPCs must guarantee atomicity. If any part of an RPC fails (e.g., an audit log fails to write), the entire transaction rolls back.

## Concurrency Requirements (`SELECT FOR UPDATE`)
Any RPC modifying scarce resources (event capacity, team sizes) must utilize row-level locks.
```sql
SELECT * FROM events WHERE id = p_event_id FOR UPDATE;
```
This strategy ensures that concurrent requests from 500 students at exactly the same millisecond do not oversell a 50-seat event.

## Live Permission Checks
RPCs execute with `SECURITY INVOKER` privileges by default to respect RLS. However, some RPCs (like assigning attendance) require `SECURITY DEFINER` to bypass read-only restrictions. In `SECURITY DEFINER` functions, permissions MUST be explicitly resolved and checked live inside the function block before execution.

## PostGIS Usage
Attendance RPCs use PostGIS to calculate distances dynamically against the event's `location_geofence` before committing the record.
