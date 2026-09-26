# Mobile Implementation Contract

## Authoritative Specification
For the complete, authoritative V1 mobile requirements, see [17-final-student-app-specification.md](../../mobile/17-final-student-app-specification.md).

## Expo Router hierarchy
Uses `(auth)` and `(app)` boundaries. See [Final Student App Specification](../../mobile/17-final-student-app-specification.md).

## Session lifecycle
Managed via SecureStore and React Query.

## Cache clearing on logout
Logout MUST clear React Query private data.

## Tab Customization Status
RESOLVED: Bottom navigation is frozen to a strict 3-tab shell (`ATTENDANCE/HOME`, `HISTORY`, `PROFILE`). Customization is rejected to guarantee predictable high-speed usage.

