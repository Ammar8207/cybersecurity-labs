# HTB - Appointment (Tier 1)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- Browser

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran nmap -sV on target IP — found port 80 (HTTP) open
3. Opened browser, navigated to http://<IP>
4. Found a login page
5. Used SQL injection in username field: `admin'#`
6. Bypassed authentication, got the flag

## What I Learned
- SQL injection can bypass login forms entirely
- The `'#` payload comments out the password check 
  in the SQL query
- Never trust user input — always sanitize and 
  validate on the backend
