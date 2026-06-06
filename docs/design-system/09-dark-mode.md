# Dark Mode

## Theme Architecture
NST-Events provides complete Light and Dark themes.

## Default Rule
By default, the application **Follows System Preference**. We do not force Dark Mode on users.

## Manual Override
Users can manually override the theme to strictly Light or strictly Dark via their Profile Settings.

## Token Remapping Strategy
Dark mode is implemented via CSS variables and Tailwind's `dark:` modifier.
* Example: `--bg-surface` maps to `#FFFFFF` in Light and `#111827` in Dark.
* Borders map to `#E5E7EB` in Light and `#374151` in Dark.
