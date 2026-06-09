# Storage RLS

> **NOT IN V1**: File uploads are deferred to a later release (V1.1 / V2). NST-Events V1 does not require a storage provider and uses default/generated fallback assets only. This document is preserved as **future design guidance** for when file uploads are implemented.

## Status
**Deferred — Not implementable until file uploads are prioritized and a storage provider is selected (ADR-008).**

## Future Access Control Model
In the Express-based architecture, file access control is **not** implemented via PostgreSQL RLS on a `storage.objects` table (that was Supabase-specific). Instead, all storage authorization flows through the Express backend:

- **Uploads**: Client requests a pre-signed upload URL from Express. Express RBAC middleware validates the user's role and bucket path before issuing the URL.
- **Private reads**: Client requests a signed read URL from Express. Express validates `FACULTY` or `PLATFORM_ADMIN` role before generating the URL.
- **Public reads**: `avatars`, `event_media`, and `club_banners` are intended to be publicly readable by CDN URL without requiring a signed URL.

## Future Bucket Structure
* **`avatars`**: Public. Upload gated by Express: `avatars/<user_id>/` enforced at URL-generation time.
* **`event_media`**: Public. Upload requires `CLUB_ADMIN` or `CORE_MEMBER` role via Express RBAC.
* **`club_banners`**: Public. Upload requires `CLUB_ADMIN` role.
* **`secure_documents`**: Private. Read requires `FACULTY` or `PLATFORM_ADMIN` role verified by Express before issuing signed URL.

## What Changed From the Old Architecture
The old architecture used Supabase Storage RLS policies on the `storage.objects` table (e.g., `bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]`). These policies no longer apply. All access control is now Express-mediated.
