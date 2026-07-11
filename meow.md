# HTB - Meow (Tier 0)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- telnet

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran nmap on target IP — found port 23 (Telnet) open
3. Connected using `telnet <IP>`
4. Logged in with username `root`, blank password
5. Ran `ls` to list files, then `cat flag.txt` to read the flag

## What I Learned
- Telnet transmits data unencrypted
- Services left with default or empty credentials are 
  a common misconfiguration in real environments
