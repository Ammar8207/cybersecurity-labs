# HTB - SpookyPass (Reversing)
**Difficulty:** Very Easy

## Tools Used
- `file`
- `strings`

## Steps
1. Unzipped the challenge archive using `hackthebox` as the password
2. Ran `file pass` — identified it as a 64-bit ELF executable, not stripped
3. Ran `./pass` — binary prompted for a password, rejected wrong input
4. Ran `strings pass` — dumped readable text from the binary
5. Found hardcoded password in plaintext: `s3cr3t_p455_f0r_gh05t5_4nd_gh0ul5`
6. Entered the password — got the flag

## What I Learned
- `file` identifies binary type and whether symbols are stripped
- Not stripped = symbol names intact, easier to reverse
- `strings` extracts all readable plaintext from a binary
- Hardcoded secrets in unobfuscated binaries are trivially recoverable
- Real-world binaries use obfuscation/encryption to prevent this
