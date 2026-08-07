# THM - Hacker Holidays Day 8: Towel on the Sunbed
**Category:** Web
**Difficulty:** Medium

## Tools Used
- nmap
- Browser
- Burp Suite

## Steps

1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Ran `nmap -sC -sV <IP>` — found port 22 (SSH) and port 3000 (HTTP/Node.js Express)
3. Opened `http://<IP>:3000` in browser — found Ponzi Portfolio login page
4. Registered a new account — landed on dashboard showing 0 PONZI balance
5. Found "Claim" button for staking rewards — 50 PONZI per claim, once every 24 hours
6. Noted the Whale Vault requires 150 PONZI to unlock
7. Clicked Claim and intercepted the request in Burp Suite:
   `POST /claim` with session cookie, no CSRF token or request body
8. Sent the request to Burp Repeater, duplicated it 4 more times (5 tabs total)
9. Grouped all 5 tabs and used **Send group in parallel (single-packet attack)**
10. Race condition succeeded — all 5 claims processed before the server recorded the first — balance jumped to 250 PONZI, status upgraded to Whale
11. Opened the Whale Vault — retrieved the flag

## What I Learned
- Race conditions occur when a server reads state, makes a decision, then writes state — parallel requests can slip through before the write happens
- The `/claim` endpoint had no atomic check — it read the last-claim timestamp and wrote it in separate operations, creating a exploitable window
- Burp Repeater's "Send group in parallel" replicates a single-packet attack, sending multiple requests simultaneously to maximize timing overlap
- No CSRF token or request uniqueness check meant the same request could be replayed identically
- Business logic flaws don't require code injection — abusing the intended flow at the right moment is enough
