# Dashboard Freeze V1

## Status: FROZEN
## Version: 1.0
## Date: 2026-06-06

## Accepted Tradeoffs
* Operations Mode requires the user to keep the browser tab active to cast the TOTP QR code.
* URL State Rule creates long, complex URLs for deeply filtered data tables.

## Known Limitations
* Command Palette macro creation is static for V1. Custom user macros are deferred.

## Future Enhancements & Out Of Scope
* Real-time collaborative event drafting (Google Docs style) is out of scope.
* Automated Certificate issuance is out of scope for V1.

## Resolved Issues (Phase 1-4)
* Operations Mode lockdown is now backed by `events.is_locked` + `lock_event()` / `unlock_event()`.
