# ADR-002: Use PostgreSQL as the primary database

## Status
Accepted

## Context
We need a robust, scalable data store capable of handling complex relationships between students, clubs, events, and attendance records.

## Decision
Use PostgreSQL as the primary database.

## Consequences
Positive: Strong relational modeling, RLS support, mature ecosystem, and future portability.

## Alternatives Considered
* **NoSQL (MongoDB)**: Rejected because event and user data is highly relational.

## Future Impact
Forms the foundation for all data persistence and security (RLS).
