# Waitlist Confirmation Flow

## Context
Waitlisted registrations require a dedicated, first-class confirmation state to prevent user confusion regarding their actual attendance status.

## Flow Sequence
1. Register
2. RPC Response returns `WAITLISTED`
3. UI routes to **Waitlist Confirmation Screen**

## Display Elements
* **Waitlist Status**: Prominent warning color indicating they do not have a confirmed seat.
* **Queue Position**: Display queue number (e.g., "You are #4 on the waitlist").
* **Waitlist Explanation**: Clear text explaining that they will be notified via Push if a seat opens up (Priority 3 Home State).

## Actions
* **Keep Waitlist Spot**: Acknowledge and return to feed.
* **Cancel Registration**: Remove self from queue.
