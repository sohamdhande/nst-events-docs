# Toolbar Customization Rules

## Customization
Users may be permitted to customize their bottom tab bar.

## State Destruction
If a user removes a tab from their active toolbar:
* The navigation state and memory associated with that tab is **destroyed**.
* If the tab is later re-added, a **fresh state is created**.

## Rationale
No state preservation exists for removed tabs. This strict rule prevents massive background memory leaks and ensures predictable delta-sync behavior when the app is foregrounded.
