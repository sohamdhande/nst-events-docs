# Layout & Spacing

## Grid System
* **Web Grid**: 12-column fluid grid with maximum container width constraints (e.g., `max-w-7xl` for standard views).
* **Mobile Grid**: 4-column responsive grid with standard 16px horizontal margins.

## Spacing Scale
We utilize a strict 4pt linear scale mapped directly to Tailwind utilities:
* `space-1`: 4px (tight grouping)
* `space-2`: 8px (element spacing)
* `space-4`: 16px (component padding)
* `space-6`: 24px (section padding)
* `space-8`: 32px (layout margins)

## Radius Scale
Border radii are kept minimal and sharp, inspired by Vercel/Linear:
* `rounded-sm`: 2px (checkboxes, tags)
* `rounded-md`: 4px (buttons, inputs)
* `rounded-lg`: 8px (cards, modals)
* `rounded-full`: 9999px (avatars only)
