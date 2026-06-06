# Accessibility Standards

## WCAG AA Minimum
All components are strictly audited against WCAG 2.1 AA.

## Keyboard & Focus
* Every interactive component supports `Tab`, `Enter`, and `Space`.
* Focus management traps focus within Modals and Drawers.
* Visible focus rings (`focus-visible`) are mandatory.

## Screen Reader Support
* `aria-labels` and `aria-describedby` map to helper text.
* `aria-live="polite"` regions announce dynamic data (like LiveMetricHUD capacity updates).

## Visuals & Touch
* Contrast minimum 4.5:1 for text.
* Touch targets minimum 44x44px.
* Motion Reduction (`prefers-reduced-motion`) disables spring animations and replaces them with instant opacity fades.
