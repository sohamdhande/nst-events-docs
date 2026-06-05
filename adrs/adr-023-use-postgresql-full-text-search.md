# ADR-023: Use PostgreSQL Full Text Search

## Status
Accepted

## Context
Users need to discover events, clubs, and peers easily.

## Decision
Use PostgreSQL Full Text Search.

## Consequences
Positive: Avoids Elasticsearch complexity while delivering robust search functionality.
Negative: Requires setting up search vectors and triggers.

## Alternatives Considered
* **Algolia / Elasticsearch**: Rejected due to cost and infrastructure overhead.

## Future Impact
Keeps the tech stack simple and cost-effective.
