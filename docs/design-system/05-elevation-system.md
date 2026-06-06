# Elevation System

## Core Rule: Borders Over Shadows
We explicitly reject heavy, Material Design-style drop shadows. Elevation and hierarchy are established using **surface contrast** and **subtle borders**.

## Surface Layers
1. **Base Background**: `bg-gray-50` (Light) / `bg-black` (Dark).
2. **Surface Layer**: `bg-white` (Light) / `bg-gray-900` (Dark) with a `border-gray-200` (Light) / `border-gray-800` (Dark) outline.

## Permitted Shadows
Shadows are strictly reserved for elements that "float" above the page context:
* **Modals / Dialogs**: Subtle, highly diffused shadow (`shadow-xl` mapped to a low opacity).
* **Dropdowns / Command Palette**: Medium shadow (`shadow-lg`).

## Prohibited Shadows
* **Cards**: Cards must NEVER use box-shadows. They use a 1px border.
* **Buttons**: Standard buttons do not cast shadows.

## Glass Surfaces (Overlays)
Backdrops for modals use a dense blur (`backdrop-blur-sm`) combined with a semi-transparent dark overlay to focus user attention securely.
