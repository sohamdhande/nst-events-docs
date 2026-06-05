# Leaderboard Data Model

## Model
* **`leaderboard_scores`**: An append-only ledger of points. 
* **`seasons`**: Represents academic terms (e.g., "Spring 2026").

## Strategy
* **No points for registration**: To prevent abuse, points are strictly tied to confirmed physical attendance records or manual Faculty awards.
* **Seasons**: We calculate scores dynamically summing over a specific date range.

## Individual vs Club Leaderboard
* Individual Leaderboard: Points grouped by `user_id`.
* Club Leaderboard: Points are aggregated based on the participation rates of the club's events and the achievements of its Core Members.
