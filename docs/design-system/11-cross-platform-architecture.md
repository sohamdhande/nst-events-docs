# Cross-Platform Architecture

## Single Token Source
To maintain exact visual parity between the Next.js Dashboard and the Expo Mobile App, both codebases consume the exact same token definitions.

## Web Implementation
* **Framework**: React / Next.js
* **Styling**: Tailwind CSS
* **Components**: Radix UI primitives wrapped with Tailwind styles.

## Mobile Implementation
* **Framework**: React Native / Expo
* **Styling**: NativeWind (Tailwind for React Native)
* **Components**: Custom built native views mapping closely to the Web variants.

## Visual Parity
Due to the Single Token Architecture, semantic colors and border radii translate 1:1 across platforms. Native elements (like scrollbars or native navigation headers) are suppressed or styled to match the brand identity.
