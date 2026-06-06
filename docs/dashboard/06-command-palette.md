# Command Palette

## Architecture
A global overlay (accessed via Cmd+K / Ctrl+K) providing instant navigation and action execution.

## Discovery & Search
Indexes all accessible routes, active events, and club rosters. Search relies on the backend FTS vectors for rich results.

## Actions
Users can execute macros: "Create Event", "Approve Hackathon", "Switch to Developer Club".

## Command Palette Security (CRITICAL)
**Command Palette ≠ Permission Bypass.**
The palette is strictly a UI navigation layer. 
* **Role Filtering**: The palette only renders actions the current user is authorized to see.
* **Permission Enforcement**: Executing an action from the palette fires the exact same API request as clicking the button. It must pass JWT validation, RPC authorization, and RLS policies.
