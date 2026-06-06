# Accessibility

## Minimum Standard
NST-Events enforces WCAG 2.1 AA as the absolute minimum standard across all interfaces.

## Focus States
Every interactive element must define a visible focus ring (`focus-visible:ring-2 focus-visible:ring-primary-base focus-visible:ring-offset-2`).

## Keyboard Navigation
The entire Dashboard must be operable via keyboard (Tab, Space, Enter, Escape).

## Screen Readers
Dynamic data (like live attendance updates or Command Palette results) must utilize `aria-live` regions.

## Touch Targets
Mobile UI enforces a minimum touch target size of `44x44` points for critical actions.

## Attendance Accessibility Fallback
The `Swipe To Check-In` gesture is difficult for users with limited motor control. 
* **Fallback**: The UI provides a **Tap To Confirm** accessibility mode. When enabled, the swipe gesture is replaced with a standard, highly visible "Confirm Attendance" button requiring a single tap.
