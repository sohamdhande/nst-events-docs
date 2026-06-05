# Storage RLS

## Storage Policies
Supabase Storage utilizes Postgres RLS natively on the `storage.objects` table.

## Matrices
* **Read Policies**: `avatars` (Public), `event_media` (Public), `secure_docs` (Private).
* **Upload Policies**: 
  * `avatars`: `bucket_id = 'avatars' AND auth.uid()::text = (storage.foldername(name))[1]`
  * `event_media`: Requires `CLUB_ADMIN` lookup.
* **Delete Policies**: Same as upload.

## Folder Isolation
Folders inside buckets mirror database UUIDs to strictly enforce ownership logic.

## Signed URL Strategy
For `secure_docs`, the client must request a signed URL. The generation of this URL is gated by an RPC that verifies the user's `FACULTY` or `PLATFORM_ADMIN` status.
