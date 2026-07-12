# HTB - Crocodile (Tier 1)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- ftp
- gobuster
- Browser

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV <IP>` — found port 21 (FTP) and 
   port 80 (HTTP/Apache) open
3. Connected to FTP as anonymous, ran `ls` — found 
   `allowed.userlist` and `allowed.userlist.passwd`
4. Downloaded both files using `get`
5. Used Gobuster to brute force PHP files:
   `gobuster dir -u http://<IP> -w /usr/share/wordlists/dirb/common.txt -x php`
6. Found `login.php` — navigated to it in browser
7. Used credentials from FTP files to log in as admin
8. Got the flag from the dashboard

## What I Learned
- Always check FTP anonymous login — sensitive files 
  are often left exposed
- Gobuster helps find hidden pages not linked anywhere 
  on the site
- Combining findings from multiple services (FTP + HTTP) 
  is key in real engagements
