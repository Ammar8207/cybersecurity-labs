# THM - Hacker Holidays Day 14: Management Wants a Word
**Category:** Forensics  
**Difficulty:** Hard

## Tools Used
- impacket-secretsdump
- impacket-dpapi
- john
- python3 (pycryptodome)
- cryptsetup
- tesseract-ocr
- poppler-utils

## Steps

1. Extracted the zip and identified a KAPE triage collection for user `vera`:
   - `NTUSER.DAT` — user registry hive
   - `AppData/Local/Google/Chrome For Testing` — Chrome browser artifacts
   - `AppData/Roaming/Microsoft/Protect` — DPAPI master key
   - `Documents/backup` — unknown 100MB binary file (no extension)

2. Dumped local SAM hashes using the SAM and SYSTEM hives:

```bash
impacket-secretsdump -sam KAPE/C/Windows/System32/config/SAM \
  -system KAPE/C/Windows/System32/config/SYSTEM LOCAL
```

Retrieved vera's NT hash for her local account.

3. Cracked the NT hash with john:

```bash
echo "<nt_hash>" > vera.hash
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt vera.hash
```

Recovered vera's Windows login password.

4. Located vera's DPAPI master key blob:
KAPE/C/Users/vera/AppData/Roaming/Microsoft/Protect/
<SID>/
<master_key_guid>

5. Decrypted the DPAPI master key using vera's SID and cracked password:

```bash
impacket-dpapi masterkey \
  -file "...Protect/<SID>/<master_key_guid>" \
  -sid "<SID>" \
  -password <vera_password>
```

6. Used the master key to decrypt Chrome's AES key from `Local State`, then decrypted the saved password from `Login Data` using Python (pycryptodome + impacket DPAPI):
URL: http://bytelotus.thm:8080/
User: VeraSecretVault
Pass: <decrypted_chrome_password>

7. Identified `Documents/backup` as a VeraCrypt container (hint: version `1.26.29` = VeraCrypt). Opened and mounted it using the recovered Chrome password as the passphrase:

```bash
echo "<chrome_password>" | sudo cryptsetup open --type tcrypt \
  --veracrypt KAPE/C/Users/vera/Documents/backup vera_vol
sudo mount /dev/mapper/vera_vol /mnt/vera
```

8. Found two files inside the container — `transactions_q3.csv` (no flag) and `important_invoice_byte_lotus.pdf` (scanned image, no extractable text)

9. Used OCR to extract text from the scanned PDF:

```bash
pdftoppm important_invoice_byte_lotus.pdf /tmp/vera_pdf
tesseract /tmp/vera_pdf-1.ppm /tmp/vera_out
cat /tmp/vera_out.txt
```

Flag found embedded in the invoice description field.

## What I Learned
- Windows forensics works as a chain — cracking the NT hash unlocked DPAPI, which unlocked Chrome secrets, which opened a VeraCrypt container
- Chrome stores saved passwords encrypted with AES-256-GCM; the AES key is wrapped by DPAPI in `Local State` — both files are needed together to decrypt
- DPAPI master keys are encrypted with the user's Windows password — a weak password makes the entire chain collapsible offline
- VeraCrypt containers have no recognizable file extension or magic bytes — they look like random data; context clues like version numbers in hints are needed to identify them
- `cryptsetup` with `--type tcrypt --veracrypt` opens VeraCrypt volumes on Linux without installing VeraCrypt
- Scanned PDFs contain no extractable text — `pdftotext` returns nothing; OCR via `tesseract` is required
- Never save sensitive passwords in a browser on any machine that could be collected during a triage — Chrome Login Data, Local State, and DPAPI material together fully expose all saved credentials
