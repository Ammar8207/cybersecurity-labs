# HTB - Paperwork (Easy)
**Difficulty:** Easy
**OS:** Linux

## Tools Used
- nmap
- curl
- unzip
- python3 (socket scripting)
- ssh-keygen
- ssh

## Steps
1. Ran `nmap -sV -sC -p22,80 paperwork.htb` — found SSH and HTTP only; full port sweep (`nmap -p- --min-rate=1000 -T4`) revealed an additional port 1515 (`ifor-protocol`)
2. Added vhost entry: `echo "<IP> paperwork.htb" | sudo tee -a /etc/hosts`
3. Visited the web page — it referenced RFC 1179 (LPD printing protocol), target queue `archive_intake`, and a downloadable "Internal Processor" archive
4. Downloaded and extracted the archive via `curl -s http://paperwork.htb/download/archive -o archive.zip && unzip archive.zip` — found `server.py`, the source of the custom LPD service on port 1515
5. Reviewed `server.py` and found the LPD control file's job name (`J` field) gets interpolated unsanitized into a shell command via `subprocess.Popen(..., shell=True)`
6. Wrote a Python script to speak raw LPD protocol, injecting a reverse shell payload into the job name field
7. Started a listener (`nc -lvnp 4444`) and ran the exploit against port 1515 — got a shell as `lp`
8. Enumerated internal listening ports with `ss -tlnp` — found a PJL printer service on `127.0.0.1:9100` and another service on `127.0.0.1:1337`
9. Queried the PJL service and pulled its source (`jetdirect.py`) via `FSUPLOAD`, showing it ran as `archivist` and translated paths unsafely, allowing `../` traversal outside its sandboxed root
10. Generated an SSH keypair on Kali with `ssh-keygen -t ed25519`
11. Used the PJL `FSDOWNLOAD` command with a `../.ssh/authorized_keys` traversal path to write the public key into `archivist`'s `authorized_keys`
12. SSH'd in as `archivist` using the private key
13. Read the user flag from `~/user.txt`
14. Reviewed `/usr/bin/paperwork-daemon` (running as root) — found it watches `commands.log` for PJL commands like `FSQUERY`/`FSUPLOAD`/`FSDOWNLOAD`, and when triggered, passes two live file descriptors (the log file and `/etc/paperwork/admin_pins.conf`) over a Unix socket at `/run/paperwork/mgmt.sock` via `SCM_RIGHTS`
15. Triggered the lockdown by appending `FSQUERY trigger` to `commands.log`
16. Connected to the Unix socket and received the passed file descriptors using `recvmsg`, then read their contents directly with `os.pread`
17. Extracted the root password from the leaked `admin_pins.conf` content
18. Ran `su -` with the leaked password to become root
19. Read the root flag from `/root/root.txt`

## What I Learned
- Legacy/custom network protocols (like a hand-rolled LPD service) are worth fingerprinting carefully — service info in a banner or webpage can point straight to the exploit class (RFC 1179 → printing → command injection patterns)
- `subprocess.Popen(..., shell=True)` with any attacker-controlled field is a direct command injection vector, even when the field looks like harmless metadata (a print job name)
- PJL (`@PJL FSUPLOAD`/`FSDOWNLOAD`) filesystem commands on printer-emulation services are a known path traversal surface — path "sandboxing" via simple string replacement + `normpath` doesn't actually enforce a root boundary
- Reading a service's own source (when leaked or downloadable) is often faster than guessing payloads
- Unix domain sockets with group-writable permissions can leak privileged file descriptors across privilege boundaries via `SCM_RIGHTS` — once an FD is passed, standard file permission checks no longer matter, since the receiving process already holds an open handle
- Always check for internal-only services bound to `127.0.0.1` after gaining an initial foothold — they're often the real privilege escalation path
