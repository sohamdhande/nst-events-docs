# ADR-024: Store media in object storage

## Status
Accepted

## Context
We need to store images (banners, avatars) associated with events and users.

## Decision
Store media in object storage.

## Consequences
Positive: Scalability and performance.
Negative: Must sync database records with storage objects.

## Alternatives Considered
* **Database BLOBs**: Rejected due to severe performance degradation.

## Future Impact
Ensures fast API response times.
