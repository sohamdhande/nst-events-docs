# Club Leadership Handover

## Purpose
Facilitates the secure transition of `Club Admin` responsibilities when the current leader graduates or steps down.

## Transfer Process
1. **Initiate**: Current Club Admin initiates the "Leadership Transfer" workflow from the Dashboard.
2. **Select Successor**: Admin selects an eligible `Core Member` from their roster.
3. **Faculty Approval**: The request enters `PENDING_APPROVAL`. The Faculty Mentor must approve the transfer.
4. **Execution**: System securely transfers the `Club Admin` role to the successor and downgrades the initiator to `Core Member` or `Member`.

## Constraints
* Only **one** active Club Admin ownership chain at a time.
* Initiator completely loses admin privileges the moment the transfer executes.

## Platform Override
If the Club Admin unexpectedly leaves, the `Platform Admin` can forcefully execute a `FORCE_TRANSFER_LEADERSHIP` RPC.

## Failure Scenarios
If the Faculty Mentor rejects the transfer, the initiator remains Club Admin.

## Audit Logging
Both standard handovers and Platform Overrides generate a `LEADERSHIP_HANDOVER` audit log.
