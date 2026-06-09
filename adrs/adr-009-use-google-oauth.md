# ADR-009: Use Google OAuth via Express

## Status
Accepted

## Context
Users need a frictionless way to sign up and log in without remembering passwords. NST requires login to be gated to institutional Google Workspace accounts (@adypu.edu.in for students, @newtonschool.co for faculty and management).

## Decision
Use **Google OAuth 2.0** implemented directly in the Express backend. After a successful OAuth handshake, the Express backend issues a signed JWT to the client. This JWT is sent on every subsequent request and verified by Express middleware before any protected route handler executes.

Authentication is not delegated to a third-party auth service — the Express backend owns the OAuth flow end-to-end.

## Consequences
Positive: Institutional authentication requirement is easily met. Google Workspace accounts already enforce 2FA for NST staff. No third-party auth dependency to manage or pay for.
Negative: The team must implement token issuance, refresh token rotation, and session invalidation logic. Dependent on Google's OAuth uptime.

## Alternatives Considered
* **Email/Password**: Rejected due to security risks and poor UX for an institution already using Google Workspace.
* **Supabase Auth**: Rejected along with the Supabase platform.
* **Auth0 / Clerk**: Rejected to avoid external service dependencies and keep the full stack deployable on the NST Cluster.

## Future Impact
Simplifies identity management by anchoring identity to institutional Google accounts. Domain restriction enforced in Express middleware during OAuth callback.

