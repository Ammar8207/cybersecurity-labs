# HTB - Unified (Tier 2)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- Metasploit (ubiquiti_unifi_log4shell)
- Burp Suite
- mongo (MongoDB shell)
- ssh

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV -sC` on target — found ports 22, 6789, 8080, and 8443 open
3. Visited `https://<IP>:8443` in browser — found UniFi Network login page (version 6.4.54)
4. Identified the version as vulnerable to Log4Shell (CVE-2021-44228)
5. Loaded Metasploit and used `exploit/multi/http/ubiquiti_unifi_log4shell`
6. Configured RHOSTS, RPORT, SSL, LHOST, and SRVHOST
7. Ran the exploit and got a reverse shell as the `unifi` user
8. Stabilized the shell using `script /dev/null -c bash`
9. Found the user flag at `/home/michael/user.txt`
10. Connected to the local MongoDB instance on port 27117 using `mongo --port 27117`
11. Switched to the UniFi database with `use ace`
12. Enumerated admin users with `db.admin.find().forEach(printjson)`
13. Found the administrator account and attempted to update the password hash using `db.admin.update()`
14. Queried site settings with `db.setting.find().forEach(printjson)`
15. Found plaintext SSH credentials for root: `NotACrackablePassword4U2022`
16. SSH'd into the machine as root using the discovered password
17. Grabbed the root flag from `/root/root.txt`

## What I Learned
- Log4Shell (CVE-2021-44228) can be exploited through simple input fields like a login form
- JNDI leverages LDAP to trigger the vulnerability
- UniFi stores its data in a MongoDB database named `ace` on port 27117
- Always check local databases for stored credentials — they often contain plaintext passwords
- PowerShell history on Windows and MongoDB settings on Linux are both goldmines for credential hunting
