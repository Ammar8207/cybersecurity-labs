# HTB - Archetype (Tier 2)
**Difficulty:** Very Easy  
**OS:** Windows

## Tools Used
- nmap
- smbclient
- impacket-mssqlclient
- python3 http.server
- nc
- impacket-psexec
- PowerShell

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV <IP>` on target — found SMB, MSSQL, and WinRM open
3. Enumerated SMB shares using `smbclient -L //<IP> -N`
4. Found a non-administrative SMB share named `backups`
5. Connected to the share using `smbclient //<IP>/backups -N`
6. Listed files with `dir` and found `prod.dtsConfig`
7. Downloaded the file using `get prod.dtsConfig`
8. Read the file with `cat prod.dtsConfig`
9. Found MSSQL credentials inside the configuration file
10. Connected to MSSQL using `impacket-mssqlclient`
11. Checked SQL privileges and confirmed the user was `sysadmin`
12. Enabled `xp_cmdshell` to execute Windows commands
13. Verified command execution with `EXEC xp_cmdshell 'whoami';`
14. Created a PowerShell reverse shell script on Kali
15. Started a Python web server using `python3 -m http.server 80`
16. Started a listener using `nc -lvnp 443`
17. Used `xp_cmdshell` to download and execute the PowerShell reverse shell
18. Got a shell as `archetype\sql_svc`
19. Read the user flag from `C:\Users\sql_svc\Desktop\user.txt`
20. Checked PowerShell history at `ConsoleHost_history.txt`
21. Found Administrator credentials in the PowerShell history file
22. Used `impacket-psexec` with Administrator credentials to get a SYSTEM shell
23. Read the root flag from `C:\Users\Administrator\Desktop\root.txt`

## What I Learned
- SMB anonymous access can expose sensitive backup files
- Configuration files may contain hardcoded credentials
- MSSQL credentials can be used with `impacket-mssqlclient`
- `xp_cmdshell` allows command execution on Windows through MSSQL
- PowerShell history can leak sensitive commands and passwords
- `psexec` can be used to gain an administrative shell when valid credentials are available
