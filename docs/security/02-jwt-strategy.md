# JWT & Session Strategy

## Overview
NST-Events uses a stateless access JWT paired with database-backed opaque refresh tokens. The Express backend owns the entire authentication lifecycle — Google OAuth, token issuance, rotation, and revocation.

---

## Access Token (JWT)

| Property | Value |
|---|---|
| **Format** | Signed JWT (HS256 or RS256) |
| **Expiry** | 15 minutes |
| **Payload** | `{ sub: user_id, iat, exp }` |
| **Signing secret** | `JWT_SECRET` env var (≥256-bit random string) |
| **Stored client-side** | In memory only (not localStorage, not cookies by default) |

The JWT contains only `user_id`. It does not contain roles or club memberships — those are resolved live from the database on every request.

---

## Refresh Token

Refresh tokens are **opaque random strings** stored server-side (in PostgreSQL). The client receives only the token value. The server stores only the SHA-256 hash.

### `refresh_tokens` Table

```sql
CREATE TABLE refresh_tokens (
  id          UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  token_hash  TEXT        NOT NULL UNIQUE,  -- SHA-256 of the raw token
  family_id   UUID        NOT NULL,         -- Links rotated tokens together
  expires_at  TIMESTAMPTZ NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  revoked_at  TIMESTAMPTZ,                  -- set on logout or compromise
  user_agent  TEXT,                         -- for session management UI
  ip_address  INET                          -- for anomaly detection
);
```

| **Expiry** | 30 days |
| **Storage** | HttpOnly, Secure, SameSite=Strict cookie |
| **Hashing** | Raw token is SHA-256 hashed before DB insert |
| **Rotation** | On every use — old token revoked, new token issued with the same `family_id` |
| **Family Revocation**| If a revoked token is used, all tokens with the same `family_id` are revoked |

---

## Authentication Flow

### Initial Login
1. Client opens Google OAuth URL provided by Express (`GET /auth/google`).
2. Google redirects to Express callback (`GET /auth/google/callback`).
3. Express verifies the OAuth code, extracts `id_token`, validates `aud` and `hd` (hosted domain), checks that `email` matches `@adypu.edu.in` or `@newtonschool.co`.
4. Express upserts the user in the `users` table using `google_sub` as the stable identity.
5. Express generates a short-lived access JWT and a random opaque refresh token. Also generates a UUID for `family_id`.
6. Refresh token is SHA-256 hashed and inserted into `refresh_tokens` along with `family_id`.
7. Raw refresh token is set as an HttpOnly cookie. Access JWT is returned in the response body.

### Authenticated Request
1. Client sends `Authorization: Bearer <access_jwt>` on every API request.
2. Express middleware verifies JWT signature and expiry.
3. If valid, Express extracts `user_id` and proceeds.
4. Before any DB query, `withUserContext(user_id, ...)` wraps the query in a transaction with `SET LOCAL app.user_id`.

### Token Refresh
1. Client calls `POST /auth/refresh` with the HttpOnly refresh cookie (automatic).
2. Express reads the cookie, SHA-256 hashes it, looks up the `refresh_tokens` row.
3. Validates: row exists, `expires_at > NOW()`.
4. **Theft Detection**: If `revoked_at IS NOT NULL`, a stolen token is being reused. Express marks all `refresh_tokens` rows with the same `family_id` as `revoked_at = NOW()` and returns 401.
5. If valid, marks the old row `revoked_at = NOW()`.
6. Generates a new random refresh token, inserts new row with the **same** `family_id`.
7. Issues a new access JWT.
8. Sets the new refresh token cookie.

### Logout
1. Client calls `POST /auth/logout`.
2. Express marks the current refresh token row `revoked_at = NOW()`.
3. Express clears the HttpOnly cookie.
4. Client discards the in-memory access JWT.

---

## Revocation Policy

| Trigger | Action |
|---|---|
| Logout | Set `revoked_at` on current token |
| Role change | **Club role revocations take effect immediately** — roles are resolved live from `club_memberships` on every request, not cached in the JWT. The 15-min JWT TTL applies only to authentication session validity. |
| Account compromise | Platform Admin deletes all `refresh_tokens` rows for the `user_id`. All active sessions are invalidated on next refresh |
| User deletion | `ON DELETE CASCADE` on `refresh_tokens.user_id` — all sessions destroyed automatically |

---

## Authorization Timing
Because roles are resolved live from `club_memberships` on every API request (not embedded in the JWT), **club role revocations take effect immediately** on the very next API call. The 15-minute JWT TTL determines only how long an *authentication session* (proof of identity) remains valid — it does **not** create a stale authorization window. If a user's club role is revoked in the database, their next API request will be rejected by Express RBAC middleware, even if their JWT has not yet expired.

---

## Security Properties
- **No role data in JWT**: Eliminates stale permission window from JWT claims.
- **HttpOnly refresh cookie**: Inaccessible to JavaScript — prevents XSS token theft.
- **SHA-256 hash storage**: Database breach does not expose raw refresh tokens.
- **Token rotation with Family Revocation**: Replay attacks using a stolen refresh token are detected. If the attacker uses a stolen token, the next refresh will trigger a family-wide revocation, immediately killing the session for both the attacker and the victim.
- **Short access token TTL (15 min)**: Limits damage window from JWT theft.
