# HTB - Abducted (Medium)
**Difficulty:** Medium
**OS:** Linux

## Tools Used
- nmap
- smbclient
- rpcclient
- python3-samba
- rclone
- ssh-keygen
- ssh

## Steps
1. Ran `nmap -sV` — found SSH (22) and Samba (139/445); server string "Hartley Group Document Services"
2. Listed shares anonymously: `smbclient -L //<IP> -N` — found `HP-Reception` (printer), `projects`, `transfer`, `IPC$`
3. `projects` and `transfer` both required auth; connected anonymously to `HP-Reception` (empty)
4. Enumerated users via null session: `rpcclient -U "" -N <IP> -c "enumdomusers"` — found single user `scott`
5. Checked password policy: `rpcclient -U "" -N <IP> -c "getdompwinfo"`
6. Identified the attack surface as **CVE-2026-4480** — a Samba print-subsystem command injection where the client-controlled print job name (`%J` substitution) is passed unsanitized to the configured print command's shell execution
7. Installed `python3-samba`, wrote an exploit script that connects to the `spoolss` RPC pipe, opens the `HP-Reception` printer handle, and submits a print job with job name `|sh`, writing a reverse shell one-liner as the print data
8. Started a listener (`nc -lvnp 4444`) and ran the exploit — got an unauthenticated reverse shell as `nobody`
9. Searched for non-standard config files: `find / -type f -name "*.conf" 2>/dev/null | grep -Ev "^/usr/|^/etc/"` — found `/opt/offsite-backup/rclone.conf`
10. Read the rclone config — contained an obfuscated password for `svc-backup`
11. Decoded it on Kali with `rclone reveal <obfuscated-string>` — recovered plaintext password
12. Tried password reuse via SSH as `scott` — succeeded; read user flag
13. Checked `sudo -l` — no privileges
14. Read `/etc/samba/smb.conf` and `/etc/samba/shares.conf` — found the `transfer` share configured with `force user = marcus` and `wide links = yes`, meaning any file written through it (as `scott`) is created on disk as `marcus`, and symlinks can escape the share's root directory
15. Created a symlink from the share into marcus's home: `ln -s /home/marcus /srv/transfer/marcus`
16. Generated an SSH keypair on Kali (`ssh-keygen -t ed25519`)
17. Connected to the `transfer` share over SMB as `scott`, navigated through the symlink into `marcus`'s home, created `.ssh`, and uploaded the public key as `authorized_keys`
18. Used `smbclient`'s `setmode` to strip world-readable permissions on the uploaded key (SSH requires strict permissions)
19. SSH'd in as `marcus` using the private key
20. Checked group membership — `marcus` belonged to the `operators` group
21. Found what that group could write to: `find / -group operators 2>/dev/null` — `/etc/systemd/system/smbd.service.d`
22. Wrote a systemd drop-in override adding `ExecStartPre=/bin/bash -c 'chmod +s /bin/bash'`
23. Ran `systemctl daemon-reload` and `systemctl restart smbd` — a polkit rule allowed `operators` group members to restart the service without a password, triggering the `ExecStartPre` command as root
24. Confirmed the SUID bit was set on `/bin/bash`, then ran `bash -p` to get a privileged shell
25. Read the root flag from `/root/root.txt`

## What I Learned
- Printer/print-spooler services are a legitimate RCE surface — Samba's `%J` job-name substitution being passed unescaped into a shell `print command` is a classic unsanitized-input injection, just in an unusual subsystem
- `rclone`'s stored credentials are *obfuscated*, not encrypted — the `rclone reveal` command uses a publicly known static key to reverse it instantly, so any leaked rclone config should be treated as a plaintext credential leak
- Credential reuse across service accounts and human accounts (`svc-backup` → `scott`) is still one of the most reliable lateral movement techniques
- Samba share options `force user` + `wide links = yes` together are dangerous: `force user` means anything written through the share lands on disk as a different, often more privileged user, and `wide links` lets symlinks escape the share's root, enabling writes to arbitrary locations (like another user's home directory) as that forced user
- `setmode` in `smbclient` lets you manipulate Unix permission bits over SMB — useful for satisfying SSH's strict permission requirements on `authorized_keys` when writing files through a share
- Group-writable systemd drop-in directories (`*.service.d/`) are a direct root path if the group can also reload/restart the service (via polkit or passwordless sudo) — `ExecStartPre`/`ExecStartPost` directives run with the service's privileges, typically root
