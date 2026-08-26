# Club Branding & Banners

## 1. Overview
Club Branding allows student organizations to establish a visual identity on the platform. The primary branding element in V1 is the Club Banner.

## 2. Current Architecture (V1)
V1 Club Branding uses an externally hosted image URL. The platform stores only the URL reference in the `banner_url` field.

> **FILE UPLOADS REMAIN DEFERRED.** 
> Club Banner is NOT a file-upload pilot in V1. No platform-managed media storage infrastructure (e.g., S3, R2, signed uploads) is implemented. A future version may introduce platform-managed media storage.

### 2.1 External URL Storage
The platform does not store image files. It stores a string reference to an external HTTP/HTTPS URL. The backend validates only the scheme (rejecting `javascript:`, `data:`, etc.), not the remote image contents or dimensions.

### 2.2 Form Specifications
Web Dashboard management provides a URL input field with a live preview.

**Constraints (UX Recommendations only):**
- **Recommended Size:** 1600 × 400 pixels
- **Minimum Size:** 1200 × 300 pixels
- **Aspect Ratio:** 4:1 (Web UI enforces layout using `object-fit: cover`)

### 2.3 Missing/Broken Banner Handling
- If `banner_url` is null, the frontend renders a semantic "No Banner" token-based fallback.
- If the image URL is broken or fails to load, the frontend catches the `onError` event and gracefully downgrades to the "No Banner" fallback without breaking layout.

## 3. Operations

### 3.1 Create Club
Club creation optionally accepts `banner_url`.

```json
{
  "name": "Robotics Club",
  "description": "Building the future.",
  "initial_admin_id": "uuid",
  "banner_url": "https://example.com/banner.jpg"
}
```

### 3.2 Post-Creation Management
Admins (Platform or Club) can modify the banner using the standard Club update endpoint:

`PATCH /v1/clubs/:id`

- **Update/Add:** Send the new URL string.
- **Remove:** Send `banner_url: null`.
- **Unchanged:** Omit the field.

## 4. Security
- The backend strictly enforces the `http://` or `https://` scheme.
- The frontend enforces valid URLs before submission.
- Unsafe schemes (e.g., `javascript:`) are rejected by the backend.
- The backend does NOT fetch the remote image URL, preventing Server-Side Request Forgery (SSRF) vulnerabilities.

## 5. Acceptance Criteria
See `docs/frontend/shared/ACCEPTANCE_CRITERIA.md` (AC-CLUB-BANNER-URL-01 through 11).
