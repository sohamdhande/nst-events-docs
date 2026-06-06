# Design Tokens

The following represents the engineering specification for Tailwind/NativeWind configuration.

## Colors
* `color-primary-base`: `var(--color-primary-base)`
* `color-surface-base`: `var(--color-surface-base)`
* `color-text-primary`: `var(--color-text-primary)`
* `color-text-muted`: `var(--color-text-muted)`
* `color-border-subtle`: `var(--color-border-subtle)`

## Typography
* `font-sans`: `'Geist Sans', sans-serif`
* `font-mono`: `'Geist Mono', monospace`

## Spacing & Radius
* `radius-md`: `4px`
* `radius-lg`: `8px`

## Implementation
These tokens act as the absolute source of truth. They must be perfectly synchronized across the Figma variables, `tailwind.config.js` (Web), and NativeWind configurations (Mobile).
