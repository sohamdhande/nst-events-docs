# Motion System

## Philosophy
Motion must be purposeful, rapid, and physically accurate.

## Web Animations
* **Durations**: Extremely fast. Standard UI transitions (hover, active) use `150ms`. Layout shifts use `200ms`.
* **Easing**: Use `ease-out` for entering elements and `ease-in` for exiting elements.

## Mobile Animations
* **Spring Animations**: Native elements (Modals, Bottom Sheets) use spring physics for natural, interruptible gestures.

## Interaction Feedback
* All interactive elements (buttons, rows) provide instant visual feedback on touch/hover via background color shifting or subtle opacity changes.

## Swipe To Check-In
The primary mechanism for recording attendance on mobile is a **Swipe To Check-In** gesture.
* **Behavior**: Provides heavy mechanical resistance, requiring deliberate physical intent to prevent accidental check-ins. Provides haptic feedback upon success.
