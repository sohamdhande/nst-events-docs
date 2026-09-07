# Frontend Implementation Contract

## A. Purpose
To serve as the definitive entry point for AI code generation.

## B. Scope
Web (Next.js) and Mobile (Expo) frontends.

## C. Authority hierarchy
1. `docs/frontend/IMPLEMENTATION_CONTRACT.md`
2. Screen specifications
3. Shared component specifications
4. `docs/api/02-api-routing-matrix.md`

## D. Web Architecture
See [Web Implementation Contract](./web/IMPLEMENTATION_CONTRACT.md).

### Web App Shell
The authorized global layout structure (`AppShell`) consists of:
- **Sidebar Navigation** (Left column)
- **TopBar** (Top row; hosts mobile title, sign-out, and NotificationDrawer access)
- **Main Content Area** (Scrollable view)

### Shared Shell Components
- **BreadcrumbTrail**: A flat routing trace displayed at the top of the Main Content Area on applicable screens.
- **ContextSwitcher**: A presentation-only V1 component residing in the Sidebar. Does not perform backend mutation.

## E. Mobile Architecture
See [Mobile Implementation Contract](./mobile/IMPLEMENTATION_CONTRACT.md).

## H. API Integration Rules
Only use endpoints listed in `docs/api/02-api-routing-matrix.md`. Do NOT invent APIs.

## M. Security Rules
Frontend authorization is UX-only. The backend is the security authority.
Primary CLUB_ADMIN cannot participate in their own club's event.
