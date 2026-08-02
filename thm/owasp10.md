# THM - OWASP Top 10 2025: Insecure Design
**Category:** Web Security / API
**Difficulty:** Easy

## Tools Used
- curl
- gobuster

## Steps
1. Visited `http://<IP>:5005` — static landing page, no login
2. Viewed page source — no useful information
3. Tried mobile User-Agent spoofing — no change
4. Ran gobuster — found `/console` returning 400
5. Guessed common API endpoints manually
6. Found `/api/users` — returned all users with no authentication
7. Found `/api/messages/admin` — returned admin's private messages with no authentication
8. Retrieved flag from message content

## What I Learned
- Mobile apps are just clients — the real target is the backend API
- APIs must have authentication regardless of who is "supposed" to call them
- Never assume only mobile devices will access backend endpoints
- Insecure Design means the security flaw is in the architecture, not just implementation
- Always enumerate API endpoints during recon — `/api/users`, `/api/messages` etc.
