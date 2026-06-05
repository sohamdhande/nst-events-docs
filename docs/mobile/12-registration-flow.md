# Pessimistic Registration Flow

## Philosophy
NST-Events operates on a strict **Pessimistic Registration Flow**. Because event capacity is highly competitive, the UI must never show "Registered" or optimistic success states before backend confirmation.

## Flow Sequence
1. **Tap Register**: User initiates action.
2. **Confirmation Modal**: Explicit user consent.
3. **Confirm**: User confirms.
4. **Loading State**: UI blocks further interaction and displays a spinner.
5. **`register_event` RPC**: Client awaits backend PostgreSQL transaction (which executes `SELECT FOR UPDATE` capacity locks).
6. **Response**: Backend replies.

## Outcomes
* **REGISTERED**: Success UI.
* **WAITLISTED**: Routes to the Waitlist Flow.
* **FAILED**: Error toast (e.g., network failure, banned user).
