# Search Architecture

## Implementation
Utilizes native PostgreSQL Full Text Search (FTS).

## Generated Search Vectors
We use generated columns to maintain the `tsvector` automatically without complex triggers via GIN Indexes.
```sql
search_vector tsvector GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(description, '')), 'B')
) STORED
```

## What Can Be Searched
* **Events**: Title, description, location_name.
* **Clubs**: Name, description.
* **Users**: `full_name`, `email` (Admin dashboards only).\n