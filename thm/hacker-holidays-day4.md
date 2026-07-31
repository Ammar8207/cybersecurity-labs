# THM - Hacker Holidays Day 4: Packed Light
**Category:** Forensics / Network Analysis  
**Difficulty:** Easy

## Tools Used
- Wireshark
- TShark
- Python 3

## Steps
1. Opened the provided `traffic.pcapng` in Wireshark.
2. Filtered traffic on port `8080` using:
   ```text
   tcp.port == 8080
   ```
3. Observed an initial request for `/temp/updates.py`, followed by repeated `GET /` requests every second.
4. Followed the HTTP stream for `GET /temp/updates.py` and analyzed the downloaded Python script.
5. Identified that the malware:
   - Captured keystrokes using `pynput`.
   - Encoded the captured data.
   - Exfiltrated the encoded data through the `hotel_sess_state` Cookie header.
6. Extracted all Cookie values using TShark:
   ```bash
   tshark -r traffic.pcapng \
     -Y "http.request && http.cookie" \
     -T fields \
     -e http.cookie
   ```
7. Verified the extracted values were Base64-encoded:
   ```bash
   python3 -c 'import base64; print(base64.b64decode("HA=="))'
   ```
8. Analyzed the extracted Cookie values based on the encoding method identified in the downloaded Python script.
9. Recovered the flag.

## What I Learned
- Malware can use legitimate HTTP headers (such as `Cookie`) as covert channels for data exfiltration.
- Following HTTP streams can reveal malware or scripts transferred over the network.
- Reading malware source code can quickly explain how network communication and data encoding work.
- TShark can quickly extract specific protocol fields from a PCAP without manually inspecting every packet.
- Base64 is an encoding mechanism, not encryption, and is commonly used to disguise transmitted data.
- Small, periodic HTTP requests can indicate malware beaconing or covert communication.
```
