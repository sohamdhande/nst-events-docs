# Mobile Navigation Architecture

## Overview
The NST-Events mobile app employs a fluid, context-aware navigation structure optimized for high-speed engagement and task completion. 

## Tab Structure
The primary navigation relies on a customizable bottom tab bar.
* **Home**: Context-aware dashboard, driven by the Priority System.
* **Campus**: Shared Community Resources and Discovery.
* **Profile**: Personal Records and History.

## Modals vs Stacks
* **Stacks**: Used for deep, hierarchical navigation (e.g., Campus -> Club List -> Club Details).
* **Modals**: Used for ephemeral, context-switching tasks (e.g., Event Registration, QR Check-in) requiring user focus before returning to the previous context.

## State Management
Tabs maintain their own separate navigation stacks. However, custom tab removal triggers state destruction (see Tab Governance).
