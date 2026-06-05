# Database Schema: [Domain/Module Name]

## Purpose
[Describe what data is managed within this module and its business significance.]

## Tables
### `table_name`
[Description of the table's purpose.]

| Column Name | Data Type | Constraints | Description |
|---|---|---|---|
| `id` | UUID | Primary Key | Unique identifier |
| `created_at` | Timestamp | Not Null | Creation timestamp |

## Relationships
* `table_a.foreign_id` -> `table_b.id` (One-to-Many)

## Row Level Security (RLS) Policies
* **Policy Name**: [Description of who can read/write and under what conditions.]

## Performance Considerations
* **Indexes**: [List important indexes and why they exist]
