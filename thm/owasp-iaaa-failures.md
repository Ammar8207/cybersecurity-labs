# THM - OWASP Top 10 2025: IAAA Failures
**Category:** Web Security
**Difficulty:** Easy

## Overview
IAAA stands for **Identity, Authentication, Authorization, and Accountability** — a framework for how users and their actions are verified in applications. This room covers three OWASP Top 10 2025 categories related to IAAA failures.

## Topics Covered

### A01: Broken Access Control (IDOR)
Changed the `accountID` parameter in the URL to access another user's account — found a user with over $1 million and read their private notes. This is horizontal privilege escalation — same role, but accessing another user's data.

### A07: Authentication Failures
Registered an account with username `aDmiN` — the app didn't normalize usernames before comparing, so it treated it as the admin account. Logged in and accessed the admin dashboard.

### A09: Logging & Alerting Failures
Analyzed application logs to identify a brute-force attack — found the attacker's IP, the compromised account, and what endpoint they accessed after gaining access.

## What I Learned
- IDOR allows accessing other users' data by changing ID parameters
- Authentication systems must normalize usernames before comparison
- Good logging is essential for detecting and investigating attacks
- Without proper logs, it's impossible to know what an attacker did
