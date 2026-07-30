# THM - Hacker Holidays Day 2: Room 404
**Category:** Web / Directory Enumeration
**Difficulty:** Very Easy

## Tools Used
- gobuster
- git-dumper

## Steps
1. Connected to TryHackMe VPN: `sudo openvpn thm.ovpn`
2. Started lab machine on port 8080
3. Ran gobuster to find hidden directories:
   `gobuster dir -u http://<IP>:8080 -w /usr/share/wordlists/dirb/common.txt`
4. Found `.git/HEAD` returning status 200 — exposed Git repository
5. Used git-dumper to download the source code:
   `git-dumper http://<IP>:8080/.git ./dumped`
6. Found three files: `app.js`, `index.html`, `README.md`
7. Read source files — found the flag inside

## What I Learned
- `.git/HEAD` returning 200 means an exposed Git repository
- Gobuster finds hidden directories not linked anywhere on the site
- git-dumper reconstructs full source code from an exposed `.git` folder
- Developers must never leave `.git` accessible on production web servers
