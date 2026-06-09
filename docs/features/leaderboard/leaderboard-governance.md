# Leaderboard Governance

## Abuse Prevention
Points must never be directly editable. Values are calculated dynamically from verified database records (`event_registrations` and `event_results`).
Students are cryptographically and operationally blocked from assigning roles or submitting competition results.

## Audit Requirements
Any manipulation of the base records triggering point derivation is strictly audited:
* `ASSIGN_PARTICIPATION_ROLE`
* `REMOVE_PARTICIPATION_ROLE`
* `SUBMIT_COMPETITION_RESULT`
* `MODIFY_COMPETITION_RESULT`
* `ADJUST_POINTS` (Restricted entirely to Platform Admin).

## Future Expansion
In V2, the point model may expand to include "Bounties" for open-source contributions to campus IT systems, automatically integrating with GitHub via a webhook handler in the Express backend.
