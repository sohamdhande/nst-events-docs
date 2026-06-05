# Soft Delete Strategy

## Mechanism
We implement soft deletes via a `deleted_at TIMESTAMPTZ` column.

## Why Soft Delete Exists
In an academic setting, records (like event registrations or clubs) are often requested to be restored after accidental deletion. Additionally, deleting a club should not orphan its historical attendance data, which is required for annual reporting.

## Implementation Rules
1. **Views**: We use `security_invoker = true` views (e.g., `active_events`) that automatically append `WHERE deleted_at IS NULL`. Client applications query the view, not the underlying table.
2. **INSTEAD OF DELETE triggers**: Attached to views to intercept `DELETE` statements and convert them to `UPDATE ... SET deleted_at = NOW()`.
3. **Partial Unique Indexes**: Prevent duplicate key errors for re-created entities by indexing only where `deleted_at IS NULL`.

## Cascade Behavior
Soft deletes must logically cascade. An application-layer trigger function handles soft-deleting `event_registrations` when an `event` is soft-deleted.\n