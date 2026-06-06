# Component Library

## Buttons
* **Purpose**: Primary interactions.
* **Variants**: Primary, Secondary, Outline, Ghost, Destructive.
* **States**: Default, Hover, Active, Disabled, Loading (renders a spinner).

## Inputs
* **Purpose**: Data entry.
* **States**: Default, Focus (distinct ring), Error (red border + message), Disabled.

## Cards
* **Purpose**: Grouping related information. 1px border, no shadow.

## Modals & Drawers
* **Purpose**: Contextual, high-focus tasks. Mobile defaults to Bottom Drawers; Web defaults to centered Modals.

## Tabs
* **Purpose**: Horizontal context switching within views.

## Badges
* **Purpose**: Status indicators (Success, Warning, Error, Neutral).

## Tables
* **Purpose**: Dense data display. Supports sticky headers, row hover states, and pagination controls.

## Command Palette
* **Purpose**: Global search and navigation overlay.

## Notifications (Toasts)
* **Purpose**: Ephemeral system feedback. Auto-dismiss after 4 seconds.

## Accessibility Requirements
Every component MUST possess explicit `.focus-visible` states, proper ARIA labels, and keyboard navigation support.
