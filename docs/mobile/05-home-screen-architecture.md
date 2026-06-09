# Home Screen Architecture

## Data Fetching: `get_home_feed()` RPC
The Home tab requires extreme efficiency. It relies entirely on a single aggregated backend request: `get_home_feed()`. 
The Home screen must **not** make multiple independent requests for core data.

**RPC Return Payload:**
* Upcoming Events
* Active Attendance Windows
* Notification Badge Count
* Club Updates
* Pending Actions

## Delta Sync
The RPC supports a `last_fetched_at` parameter.
* **Behavior**: The client sends the last successful refresh timestamp.
* **Response**: The backend returns *only changed data* (New notifications, changed event states, new attendance windows, updated announcements).

### Refresh Strategies
* **Background Refresh**: Silent delta sync on a polling interval.
* **Foreground Refresh**: Triggered by SSE push from the Express backend (e.g., a new attendance window opening, a registration slot becoming available).
* **Pull-to-Refresh**: Forces a hard reset of `last_fetched_at` to guarantee absolute freshness.

## Dynamic Home States
Home is entirely context-aware.
* **Attendance Active**: Attendance Widget dominates layout.
* **Event Starting Soon**: Countdown Widget dominates layout.
* **No Urgent Context**: Standard Campus Dashboard layout.
