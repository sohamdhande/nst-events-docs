# NST Events — TODO

Last updated: 2026-09-07

---

## 1. Finish core build (blocking everything else)

- [ ] **iOS build & test** — handed off to teammate with a physical iPhone
  - [ ] Pull latest branch, confirm `iosClientId` + `iosUrlScheme` already wired (done — don't recreate Cloud Console client)
  - [ ] `npx expo run:ios --device`, resolve code signing in Xcode (Signing & Capabilities → personal Apple ID team)
  - [ ] Test sign-in with real `@adypu.edu.in` / `@newtonschool.co` account
  - [ ] Test QR scan + geofence (needs a real test session with venue coordinates set near wherever they're testing — coordinate with Soham)
  - [ ] Report any errors as raw logs, not summaries

- [ ] **Sign-out flow** — confirm `GoogleSignin.signOut()` is actually wired to a visible button in the app (mentioned, never confirmed working). Needed regardless of account-deletion stance.

---

## 2. Working tree cleanup (real, unresolved from this session)

- [ ] Review and commit or discard `apps/dashboard` changes (40+ modified/untracked files — student view, club admin UI, hooks) — never independently reviewed/tested this session
- [ ] Review and commit or discard remaining `apps/api` changes (teams, attendance, events, leaderboard routers/services beyond the auth work)
- [ ] Delete scratch/debug files that should never be committed: `scratch.tsx`, `test-api.js`, `packages/database/db-check.ts`, `db-check2.ts`, `db-check3.ts`, `auth_me.js`, `create_event.js/.ts`, `list_users.ts`, `make_admin.ts`, `remove_banner.ts`, `seed_event.ts`
- [ ] Confirm `*.apk` is properly gitignored (added mid-session, verify it stuck)
- [ ] Decide fate of `apps/mobile/eas.json` and other untracked mobile files (disputes, history, settings screens, UI components) — appear to be real feature work, needs review

---

## 3. Security cleanup (before any production deploy)

- [ ] **Remove `ALLOWED_TEST_EMAILS`** entirely from any env that touches production — this was a testing-only bypass and must not exist in prod
- [ ] Rotate `GOOGLE_CLIENT_SECRET` and `JWT_SECRET` if there's any chance of prior exposure (git history, chat, screenshots)
- [ ] Re-verify `.env` files are gitignored in whatever repo state actually gets deployed
- [ ] Confirm no test/debug console.logs remain in shipped code (already removed once — re-check after further mobile work)

---

## 4. Production environment

- [ ] Stand up real production backend hosting (not localhost/LAN IP)
- [ ] Point `EXPO_PUBLIC_API_URL` (mobile) and `NEXT_PUBLIC_API_URL` (dashboard) at the real production HTTPS URL
- [ ] Run production database migrations cleanly on a fresh instance (test this before go-live)
- [ ] Move Google OAuth consent screen out of "Testing" mode (100-user cap otherwise)
- [ ] **Android production SHA-1**: get from `eas credentials` under a production build profile OR from Play Console after first upload (Google Play App Signing re-signs the app — you need THAT certificate's SHA-1, not just your own upload key) — add to the Android OAuth Client ID in Google Cloud Console
- [ ] Confirm who owns/controls the Google Cloud project the OAuth clients live in — transfer or add `clubname@newtonschool.co` as owner if currently under a personal account

---

## 5. Geofence / attendance calibration (real-world data, not code)

- [ ] Set correct venue coordinates for every real campus event location (this caused most of the "bugs" during testing — was actually just wrong test data)
- [ ] Reconsider whether 50m base radius + 100m GPS-accuracy cap is right per-venue (large hall vs. small room vs. outdoor quad may need different radii)
- [ ] Decide on a fallback for venues with poor indoor GPS (manual override by faculty? Wi-Fi-based location? Bigger radius for known-bad-GPS buildings?)

---

## 6. Institutional / account setup

- [ ] Get Apple Developer account access via `clubname@newtonschool.co`
- [ ] Get Google Play Console access via `clubname@newtonschool.co`
- [ ] Enable 2FA on `clubname@newtonschool.co`
- [ ] Set up a real password-manager / handoff process for that account (club leadership changes yearly — don't let this become a single-person dependency)
- [ ] Add team members (you, iOS teammate, future maintainers) to App Store Connect / Play Console with scoped roles rather than sharing the raw login

---

## 7. Privacy policy & store compliance

- [ ] Fill in placeholders in the privacy policy page: `[DATE]`, `[CLUB NAME]`, `[COLLEGE NAME]`, `[CONTACT EMAIL]`
- [ ] Replace `[your-domain]` in mobile app's `Linking.openURL` call with real production URL (currently deferred to publish time — don't forget)
- [ ] Test the live privacy policy URL logged-out (incognito) before submitting, confirm it actually renders with no auth redirect
- [ ] **Check with the college's registrar/IT/academic office** whether "permanent retention, no deletion" is actually aligned with the institution's own student data policy — don't finalize this unilaterally
- [ ] Fill out Apple App Privacy questionnaire (App Store Connect) using `store-data-safety-reference.md`
- [ ] Fill out Google Play Data Safety form using the same reference doc
- [ ] Answer "data deletion supported?" as **No** on both forms, consistent with the actual retention policy — don't overclaim
- [ ] Consider surfacing a short retention notice at first sign-in (not just buried in the linked policy)
- [ ] Refine the location permission usage string to be more explicit for Apple review (current: "Required to verify attendance geofence" — consider: "NST Events uses your location only when you scan an attendance QR code, to confirm you are physically present at the event.")

---

## 8. Store submission mechanics

- [ ] iOS: Distribution certificate + App Store provisioning profile (needs Apple Developer account first)
- [ ] iOS: App Store Connect listing — screenshots, description, privacy policy URL
- [ ] iOS: Budget 1-3+ days for App Review, expect scrutiny on location/auth permission justification
- [ ] Android: Google Play Console developer account setup ($25 one-time, via institutional account)
- [ ] Android: Build a signed release **AAB** (not APK) for Play Store
- [ ] Android: Complete Play Store Data Safety disclosure
- [ ] Set a realistic launch date with review-time buffer — don't plan for day-of approval on either platform

---

## 9. Operational readiness

- [ ] Set up error monitoring (e.g. Sentry) for both mobile and backend — currently debugging via raw console logs only, won't scale to real users
- [ ] Review rate limiting config is sensible for real concurrent load (e.g. everyone scanning QR at once at lecture start)
- [ ] Identify a rollback plan / last-known-good tag before deploying each release
- [ ] Set up a way to get bug reports from real users that includes enough diagnostic info (device, OS version, app version) to actually debug — don't rely on screenshots alone

---

## 10. Post-launch (v1.1+, not before v1 ships)

- [ ] Set up `expo-updates` for OTA delivery of JS-only changes (no store review needed) — configure on BOTH platforms' native builds at the same time to avoid rebuilding iOS twice
- [ ] Build a version-gate / force-update flow for native-code changes that can't ship via OTA:
  - [ ] Backend: `GET /v1/app/min-version` endpoint
  - [ ] Mobile: compare installed version against minimum on launch, show blocking "Please update" screen with store deep-link if below minimum
- [ ] Decide on and implement a real data retention/auto-deletion policy for GPS-precision location data specifically (keep attendance yes/no, purge raw coordinates after N months) — currently explicitly "no deletion," revisit if institution wants otherwise

---

## Notes / context for future reference

- Backend auth flow: `loginWithGoogle` (web, code exchange) and `loginWithIdToken` (mobile, direct id_token) both funnel through the same domain-restriction logic in `auth.service.ts` — keep it that way, don't let them diverge.
- Allowed domains are env-driven (`ALLOWED_EMAIL_DOMAINS`), not hardcoded — extend via env, not code, when adding a new institution.
- `hostedDomain` should NOT be set in `GoogleSignin.configure()` — it only supports one domain and would hide `newtonschool.co` accounts from the picker. Server-side allowlist is the real enforcement.
- Geofence RPCs (`mark_attendance`, `mark_attendance_v5`, `sync_offline_attendance`, `sync_offline_attendance_v9`) all use `geofence_radius + LEAST(gps_accuracy, 100)` — if a 5th variant is ever added, it needs this too.
