# Component Contract

See [Component Freeze](../design-system/15-component-freeze-v1.md).

All components must adhere strictly to documented variants. Inventing variants is FORBIDDEN.

## Web Shell Components

### TopBar
- **Purpose**: Global horizontal navigation bar acting as the top bounds of the AppShell. Hosts the `NotificationDrawer` toggle, Sign Out action (if implemented here), and mobile branding.
- **Placement**: Fixed at the top of the AppShell, spanning the remaining width next to the Sidebar (or full width on mobile).
- **Responsive behavior**: Visible on all breakpoints. On mobile (<768px), it often hosts the Hamburger menu and primary branding.
- **Accessibility**: Must use `<header>` semantic tag. Interactive items must be keyboard accessible with `.focus-visible`.
- **Interaction limits**: Does not perform routing independently; simply houses action triggers.

### ContextSwitcher
- **Purpose**: Displays the active campus or club context (presentation-only in V1).
- **Placement**: Inside the SidebarNavigation.
- **Variants**: Standard dropdown visual.
- **Responsive behavior**: Hidden on mobile if Sidebar is hidden, unless explicitly exposed in the mobile menu.
- **Accessibility**: Must use proper `aria-expanded` and `aria-haspopup` labels.
- **Interaction limits**: V1 is PRESENTATION-ONLY. Must not invoke backend mutations, persist state, or rewrite URLs.

### BreadcrumbTrail
- **Purpose**: A flat routing trace to provide context of the user's location.
- **Placement**: Top of the `MainContent` area, typically within a `PageHeader` or just above the primary `h1`.
- **Inputs/Props**: Array of `{ label: string, href?: string }`.
- **Accessibility**: Use `<nav aria-label="Breadcrumb">` and standard list semantics (`<ol>`, `<li>`).
- **Interaction limits**: Static links to parent routes; no dynamic client-side filtering.
