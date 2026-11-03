# JWT & Session Strategy

## Overview
NST-Events uses a stateless access JWT paired with database-backed opaque refresh tokens. The Express backend owns the entire authentication lifecycle — Google OAuth, token issuance, rotation, and revocation.

---

## Access Token (JWT)

| Property | Value |
|---|---|
| **Format** | Signed JSON Web Token |
| **Algorithm** | **`HS256`** (Symmetric signing via `JWT_SECRET`, matching active code and `jwt-and-sessions.md`) |
| **Expiry** | 15 minutes (`900s`) |
| **Payload Claims** | `{ sub: user_id, secVer: security_version, iat, exp }` |
| **Signing secret** | `JWT_SECRET` env var (≥256-bit cryptographically secure random string) |
| **Stored client-side** | In memory only (not localStorage, not cookies by default) |

The JWT contains only `user_id` (`sub`) and `secVer` (`users.security_version`). It does not contain roles or club memberships — those are resolved live from the database on every request.

### Instant Session Invalidation via `secVer` (Security Version)
The `users` table maintains an integer column: `security_version INT DEFAULT 1`.
- When an administrator modifies a user's role, deactivates an account, or revokes all sessions (`/v1/admin/users/:userId/revoke-sessions`), `security_version` is atomically incremented in PostgreSQL.
- During Express authentication middleware validation:
  ```typescript
  if (user.securityVersion !== payload.secVer) {
    throw new UnauthorizedError('Session revoked due to role or access change');
  }
  ```
- This immediately invalidates all active 15-minute access JWTs for that user across all devices without waiting for expiration.

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
  ip_address  TEXT                          -- [TECH DEBT] Implemented as TEXT instead of INET (loses subnet-query capability). See tech debt tracker.
);
```

| Property | Value |
|---|---|
| **Expiry** | 30 days |
| **Storage** | HttpOnly, Secure in production; relaxed only for local HTTP development (`NODE_ENV !== 'production'`), SameSite=Strict cookie |
| **Hashing** | Raw token is SHA-256 hashed before DB insert |
| **Rotation** | On every use — old token revoked, new token issued with the same `family_id` |
| **Family Revocation**| If a revoked token is used, all tokens with the same `family_id` are revoked |

---

## OAuth CSRF State Cookie

Used exclusively during the Google OAuth login flow to prevent CSRF attacks.

| Property | Value |
|---|---|
| **Format** | Signed random string |
| **Expiry** | Short-lived (10 minutes) |
| **Storage** | HttpOnly, Secure in production, SameSite=Lax cookie |

*Note*: `SameSite=Lax` is strictly required (instead of `Strict`) because this cookie must survive the top-level cross-site redirect back from `accounts.google.com`. This cookie is completely separate from the refresh token cookie.

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
1. Client calls `POST /auth/refresh` sending mandatory custom header `X-Client-Platform: web|mobile`.
2. Credentials presented:
   - **Web**: HttpOnly `refreshToken` cookie sent automatically (`credentials: 'include'`).
   - **Mobile**: `{ refresh_token }` in JSON request body with `Content-Type: application/json` and `X-Client-Platform: mobile`.
3. Express reads the refresh token from cookie or request body, computes its SHA-256 hash, and looks up the row in `refresh_tokens`.
4. **Theft Detection**: If `revoked_at IS NOT NULL`:
   - If $\Delta t \le 5$ seconds, treated as benign concurrent tab race (returns 401, does not revoke family).
   - If $\Delta t > 5$ seconds, treated as token theft: Express revokes all tokens with the same `family_id` via `revoke_refresh_token_family` and returns 401.
5. If valid and unexpired, Express rotates the token: marks the old token `revoked_at = NOW()`, issues a new refresh token with the same `family_id`, issues a new 15-minute access JWT with the user's current `secVer`, and returns the tokens (setting the new cookie for web).

### Refresh Rotation: Concurrent Request Handling
When a token with `revoked_at` already set is presented again, the system computes the delta between `now()` and `revoked_at`. If `delta <= 5` seconds, it is treated as a benign concurrent-request race (returns 401, does not revoke the family). If `delta > 5` seconds, it is treated as token theft (revokes the entire family, per existing policy). This is implemented via `SELECT ... FOR UPDATE` inside a transaction to serialize concurrent refresh calls on the same token.

### Logout
1. Client calls `POST /auth/logout` sending mandatory custom header `X-Client-Platform: web|mobile`.
2. Credentials presented:
   - **Web**: HttpOnly `refreshToken` cookie sent with `credentials: 'include'`.
   - **Mobile**: `{ refresh_token }` in JSON request body with `Content-Type: application/json` and `X-Client-Platform: mobile` (optional `Authorization: Bearer <access_jwt>`).
3. Express reads the refresh token from cookie or request body.
4. **Token Family Revocation on Logout**: Express calls `authService.logout(rawRefreshToken)`, which revokes the token's **entire family** (`family_id`) via `revoke_refresh_token_family`. This guarantees that an in-flight refresh request racing against logout cannot rotate the token and survive sign-out.
5. **No `security_version` Increment (Accepted Design Trade-Off)**:
   - Regular user logout revokes the entire refresh token family in PostgreSQL, preventing any future token refreshes.
   - It intentionally does **not** increment `users.security_version`.
   - Consequently, previously issued 15-minute access JWTs remain cryptographically valid until their expiration timestamp (`exp`). This is an accepted engineering design trade-off:
     - It avoids an unnecessary database write to the `users` table on every single-device logout.
     - It prevents logging out on one device (e.g. mobile) from terminating active sessions on the user's other devices (e.g. web desktop).
     - The client immediately purges the access token from in-memory store and wipes query caches upon logout. Once the 15-minute access window expires, no further access token can be acquired because the refresh family is revoked.
     - For security-critical instant session termination across all devices (e.g., account compromise or role demotion), platform administrators trigger `POST /v1/admin/users/:userId/revoke-sessions`, which atomically increments `security_version` and destroys all active access JWTs instantly.
6. Express clears both `refreshToken` and `user_role` cookies with identical security attributes (`Path=/auth`, `HttpOnly`, `SameSite=Strict`, `Secure`).
7. **Idempotent Response**: `/auth/logout` does not require a valid access token (`authenticate` middleware is omitted). It responds `204 No Content` and clears cookies regardless of whether the refresh token is present, missing, expired, or already revoked (idempotent with no information leakage).
8. Client discards in-memory access token, clears session cookies/storage, and flushes cache.

---

## Refresh & Logout Credential Rules

| Requirement | `POST /auth/refresh` | `POST /auth/logout` |
|---|---|---|
| **Mandatory Header** | `X-Client-Platform: web \| mobile` | `X-Client-Platform: web \| mobile` |
| **Missing/Invalid Header** | Rejected with `400 Bad Request` | Rejected with `400 Bad Request` |
| **CSRF Defense** | Custom non-simple header blocks cross-site forms and simple requests | Custom non-simple header blocks cross-site forms and simple requests |
| **CORS Preflight** | `OPTIONS` preflight verifies `Origin` against `ALLOWED_ORIGINS`; `Access-Control-Allow-Headers` must include `X-Client-Platform` with `credentials: true` | `OPTIONS` preflight verifies `Origin` against `ALLOWED_ORIGINS`; `Access-Control-Allow-Headers` must include `X-Client-Platform` with `credentials: true` |
| **Web Credential** | `refreshToken` cookie (`HttpOnly`, `Path=/auth`, `SameSite=Strict`, `Secure`), sent with `credentials: 'include'` | `refreshToken` cookie (`HttpOnly`, `Path=/auth`, `SameSite=Strict`, `Secure`), sent with `credentials: 'include'` |
| **Mobile Credential** | Request body: `{ refresh_token: string }`, `Content-Type: application/json`, `X-Client-Platform: mobile` | Request body: `{ refresh_token: string }`, `Content-Type: application/json`, `X-Client-Platform: mobile` (optional `Authorization: Bearer <token>`) |
| **Access Token Requirement**| None | None (`authenticate` omitted; expired access tokens can log out) |
| **Response** | `200 OK` `{ access_token, expires_in, refresh_token, user }` + Set-Cookie (web) | `204 No Content` + Clear-Cookie (`refreshToken`, `user_role`) |
| **Revocation Scope** | Rotates single token (or revokes family if theft detected) | Revokes entire `family_id` (token family) |
| **`security_version` Bump** | Unchanged | Unchanged (accepted trade-off; access JWT valid up to 15 min) |

---

## Revocation Policy

| Trigger | Action |
|---|---|
| **Regular User Logout** | Revokes entire `family_id` in `refresh_tokens`. Does **not** increment `users.security_version` (access JWT valid up to 15 min, accepted trade-off). |
| **Club Role Change** | **Club role revocations take effect immediately** — roles are resolved live from `club_memberships` on every request, not cached in the JWT. The 15-min JWT TTL applies only to authentication session validity. |
| **Admin Session Revocation / Compromise** | Platform Admin calls `POST /v1/admin/users/:userId/revoke-sessions`, which revokes all `refresh_tokens` rows and **atomically increments `security_version`**, instantly invalidating all active access JWTs across all devices. |
| **User Deletion** | `ON DELETE CASCADE` on `refresh_tokens.user_id` — all sessions destroyed automatically in PostgreSQL. |

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
- **CSRF Defense-in-Depth**: State-changing cookie-authenticated endpoints (`/auth/refresh`, `/auth/logout`) mandate `X-Client-Platform: web|mobile`, ensuring browser cross-site forms and simple requests cannot trigger state transitions.

---

## Hostile JWT Defenses
As of Phase 18 and subsequent security hardening, rigorous defenses against hostile JWT manipulation are in place:
- **Algorithm Enforcement**: The JWT verification strategy explicitly enforces symmetric **`HS256`** signed with `JWT_SECRET` (matching active code and `jwt-and-sessions.md`), strictly rejecting 'none' or mismatched asymmetric algorithm headers.
- **Signature Manipulation**: Malformed, truncated, or tampered signatures are caught immediately, resulting in 401 Unauthorized without further processing.
- **Expired Tokens**: Standard JWT `exp` checks are strictly enforced. Requests with expired tokens trigger an automated client-side refresh workflow.
- **Security Version Mismatch**: Mismatches between `payload.secVer` and `users.security_version` instantly reject requests with 401 Unauthorized, enabling immediate administrative session termination.
