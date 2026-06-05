# API Freeze V1

## Status: FROZEN
## Version: 1.0
## Date: 2026-06-05

## Accepted Risks
* RLS policies and RPCs place a heavy computation load on the database.
* PostgREST direct queries mean the database schema is closely tied to the frontend data models.

## Known Limitations
* Rate limiting is currently global per IP/User; advanced behavioral rate limiting is not implemented.

## Future Improvements
* GraphQL integration via pg_graphql.
* Dedicated caching layer (Redis) for public event feeds.

## Not In Scope
* Cryptographic signing of API requests from the mobile client.
