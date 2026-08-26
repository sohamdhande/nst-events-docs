# Storage RLS

> **NOT IN V1 (Mostly)**: General file uploads are deferred to a later release (V1.1 / V2). NST-Events V1 does not generally require a storage provider and uses default/generated fallback assets. **However**, Club Branding (Banners) is the approved pilot for the future upload architecture. This document is preserved as **future design guidance** for when file uploads are fully implemented, and is applicable to the Club Branding pilot.

## Status
**Deferred (Mostly) — Not implementable for general files until fully prioritized and a storage provider is selected (ADR-008), though Club Branding Banners will serve as the pilot.**

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
