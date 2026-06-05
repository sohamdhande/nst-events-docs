# Threat Model

Utilizes the STRIDE methodology.

## Assets
* User PII (Emails, Names)
* Attendance Integrity (Leaderboard/Grades)
* Event Data

## Attackers
* Malicious Students (Attempting proxy attendance)
* External Actors (Attempting data scraping)

## Threats & Mitigations
* **S**poofing Identity: Mitigated by strict Supabase JWT verification. (Risk: Low)
* **T**ampering with Data: Mitigated by immutable `audit_logs` and RLS locking down `UPDATE` queries. (Risk: Low)
* **R**epudiation: Mitigated by logging `actor_id` on all critical transactions. (Risk: Low)
* **I**nformation Disclosure: Mitigated by strict RLS `SELECT` policies on private events and secure documents. (Risk: Medium)
* **D**enial of Service: Mitigated by rate limiting and `SELECT FOR UPDATE` locks preventing DB deadlocks. (Risk: Medium)
* **E**levation of Privilege: Mitigated by live role resolution. JWTs cannot be modified to forge admin rights. (Risk: Low)
