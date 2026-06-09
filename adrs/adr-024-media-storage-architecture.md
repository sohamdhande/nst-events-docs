# ADR-024: Media Storage Architecture — Deferred to Post-V1

## Status
Deferred (Depends on ADR-008)

> **DEFERRED**: This ADR defines the media storage access pattern for when file uploads are implemented. NST-Events V1 does not include file uploads. V1 uses default or generated fallback assets only. This architecture applies to a future release (V1.1 / V2) and is **not required for V1 implementation**.

---

## Context
User-generated media (avatars, event banners, club banners) and internal documents (faculty sign-offs, budget proposals) will eventually need to be stored outside the relational database. When implemented, access control must be enforced by the Express backend — clients must never receive raw storage credentials.

---

## V1 Behavior
- **No file uploads** in V1.
- All media URL fields in the database (`avatar_url`, `banner_url`, etc.) are **nullable** and will store `NULL`.
- The frontend will render default or generated fallback assets (e.g., initials-based avatars, placeholder banners) when these fields are `NULL`.
- No pre-signed URL endpoints (`POST /upload/presign`, `GET /upload/signed-read`) are implemented in V1.

---

## Future Architecture (Provider-Agnostic — Applies When Implemented)
Regardless of the storage provider chosen in ADR-008, the following architecture will apply:

### Bucket Structure
| Bucket | Visibility | Upload Role |
|---|---|---|
| `avatars` | Public (CDN) | Self — avatar upload for own `user_id` path |
| `event_media` | Public (CDN) | `CLUB_ADMIN`, `CORE_MEMBER` |
| `club_banners` | Public (CDN) | `CLUB_ADMIN` |
| `secure_documents` | Private (signed URL only) | `FACULTY_MENTOR`, `FACULTY_ADMIN`, `PLATFORM_ADMIN` |

### Express Upload Flow
1. Client calls `POST /upload/presign` with `{ bucket, filename, mimeType }`.
2. Express RBAC middleware validates the user's role for the requested bucket.
3. Express generates a pre-signed upload URL scoped to the user's correct path (e.g., `avatars/<user_id>/avatar.webp`).
4. Client uploads directly to storage using the pre-signed URL (within its short TTL).
5. On success, client calls the relevant API to update the DB column (e.g., `PATCH /users/me` with `{ avatar_url }`).

### Express Private Read Flow
1. Client calls `GET /upload/signed-read?bucket=secure_documents&key=<path>`.
2. Express validates `FACULTY` or `PLATFORM_ADMIN` role.
3. Express generates a short-lived signed read URL (e.g., 5-minute TTL).
4. Client opens the URL directly.

### File Validation (Before Pre-sign)
- Allowed MIME types: `image/jpeg`, `image/png`, `image/webp`, `application/pdf`
- Max size: 5MB for images, 10MB for PDFs
- Path enforcement: Express constructs the storage path server-side — clients cannot write to arbitrary paths

### URL Validation on DB Updates
Express must validate the domain and path of any client-provided media URL before saving it to the database. See `docs/api/08-storage-security.md` for the full security requirement.

---

## What Changed From the Old Architecture
The old architecture used Supabase Storage with `storage.objects` RLS policies for access control. That model is removed. In the future architecture, Express is the access control gatekeeper — storage objects have no per-row access policy; all authorization happens before the URL is issued.

---

## Relationship to ADR-008
ADR-008 decides **which provider** implements the bucket. This ADR decides **how the Express API controls access** to it. Both are deferred for V1 and must be resolved together for file uploads to be implemented.
