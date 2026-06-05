# Dynamic QR Attendance System

## Overview
The attendance system is critical for institutional reporting and leaderboard scoring. It prioritizes fraud mitigation.

## Fraud Mitigation Mechanisms
1. **Dynamic QR Codes**: The projected QR code changes every few seconds based on a cryptographic seed to prevent students from sharing screenshots.
2. **Geofencing**: The mobile app forces GPS validation against the event's location boundary.
3. **Basic GPS Spoof Detection**: The mobile app utilizes native OS APIs to detect if a "Mock Location" provider is active and blocks the check-in attempt. (No advanced third-party anti-spoofing SDKs or Device Cryptographic Signing are used in V1).

## Constraints & Rules
* **Faculty Cannot Modify**: Faculty Mentors and Faculty Admins can view attendance reports, but they **cannot** mark attendance for themselves, generate QR codes, or manually modify student attendance records. Only Platform Admins have override capabilities to preserve audit integrity.
* **Audit Logs**: Every successful and failed scan is logged for dispute resolution.

## Scoring
Attendance is the primary driver for the Leaderboard. Points are awarded exclusively for meaningful, verified physical check-ins. No points are granted for mere registration.
