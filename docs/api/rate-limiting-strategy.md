# Rate Limiting Strategy

Rate limiting is enforced at the Supabase API Gateway and within specific Edge Functions.

## Defined Limits
* **Authentication**: Supabase default limits (e.g., 30 OTP requests/hour).
* **Attendance (`mark_attendance`)**: 5 requests per minute per user (prevents brute-forcing the QR TOTP).
* **Registrations**: 10 requests per minute per user.
* **Team Operations**: 20 requests per hour.
* **Media Uploads**: 10 uploads per hour per user.
* **Admin Actions**: High limits (100/min) for Dashboard users.
* **Notifications**: Clubs restricted to 2 broadcasts per week.

## Documented Reasoning
The limits protect the database from connection exhaustion and prevent malicious actors from scripting mass-registrations to block legitimate students from attending limited-capacity events.\n