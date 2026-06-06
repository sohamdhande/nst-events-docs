# Color System

## Semantic Color Architecture
Colors are defined by their semantic purpose (e.g., `background-primary`, `text-muted`) rather than their literal hue. This enables immediate support for Dark Mode and Multi-Campus theming.

## Primary Palette
Used for brand identity, primary actions, and active states.
* `primary-base`: The core interactive color.
* `primary-hover`: Slightly darker/lighter for interaction feedback.
* `primary-surface`: A washed-out variant for subtle backgrounds.

## Secondary & Accent Palette
Used for secondary actions, borders, and visual grouping.
* `secondary-base`: Neutral interactive elements.
* `accent-base`: Highlighting specific, non-critical callouts.

## Semantic States
* **Success**: Green scale. Used for approvals, successful registrations.
* **Warning**: Yellow/Orange scale. Used for waitlists, pending states.
* **Error**: Red scale. Used for rejections, destructive actions, locked features.
* **Info**: Blue scale. Used for standard announcements.

## Neutral System
A comprehensive gray scale (`gray-50` to `gray-900`) used for text hierarchy, borders, and surface elevations.

## Light and Dark Mode Behavior
Colors automatically invert their semantic mapping based on the active theme. `bg-surface-primary` maps to `white` in Light Mode and `gray-900` in Dark Mode.
