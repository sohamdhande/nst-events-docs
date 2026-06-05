# Storage Security

## Buckets
1. **`avatars`**: Public user profile pictures.
2. **`club_banners`**: Public club branding.
3. **`event_media`**: Public event posters and attachments.
4. **`secure_documents`**: Private forms, budget proposals, faculty sign-offs.

## Permissions & RLS
* **Read Policies**: `avatars`, `club_banners`, `event_media` are globally readable. `secure_documents` require `CLUB_ADMIN` or `FACULTY` roles.
* **Upload Restrictions**: Users can only upload to `avatars` with an ID matching their `auth.uid()`. Organizers can upload to `event_media`.

## File Validation
* **Allowed Extensions**: `jpg`, `png`, `webp`, `pdf`.
* **Size Limits**: 5MB for images, 10MB for PDFs.

## Folder Isolation
Folders inside buckets mirror database UUIDs. A user can only upload an avatar into `avatars/<user_id>/`.

## Media Ownership
All media rows must reference the `user_id` that uploaded them to prevent orphaned files and enforce quotas.

## Threats & Mitigations
* **Threat**: Malicious file upload (e.g., malware).
* **Mitigation**: Strict MIME type validation in Storage RLS policies.
