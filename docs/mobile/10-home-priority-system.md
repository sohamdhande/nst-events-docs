# Home Priority System

## The Priority Ladder
The Home tab evaluates context based on a strict 5-tier priority hierarchy. Higher priorities entirely suppress lower priorities for the dominant top-screen widget.

1. **Priority 1: Active Attendance Window**
   * An event the user is registered for is currently accepting check-ins.
2. **Priority 2: Event Starting Within 15 Minutes**
   * Countdown timer to an upcoming commitment.
3. **Priority 3: Waitlist Promotion**
   * Action required to confirm a newly available seat.
4. **Priority 4: Pending RSVP Confirmation**
   * Pending actions requiring user input.
5. **Priority 5: Campus Dashboard**
   * Default state. Standard feed and onboarding checklist.

## Conflict Resolution
If multiple states trigger simultaneously, the highest priority wins.
* **Example**: An Attendance Window is open (Priority 1), AND the user receives a Waitlist Promotion for tomorrow's event (Priority 3).
* **Result**: Priority 1 wins. The Waitlist Promotion is demoted to a standard notification or secondary card below the fold.
