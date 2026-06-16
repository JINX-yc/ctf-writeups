# Digital Archeology

**Category:** Forensics
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica{REDACTED}`

---

## Overview

We are given a disk image file (`hahahaha.img`). The real flag is XOR-encrypted inside the image. A decoy base64-encoded fake flag is also present to throw us off.

---

## Solution

### Step 1 — Identify the file type

```bash
file hahahaha.img
# hahahaha.img: DOS/MBR boot sector
```

Inspect the hex header:

```bash
xxd hahahaha.img | head -5
```

Output reveals the magic bytes `HPFS` — this is an **HPFS (High Performance File System)**, originally created for IBM's OS/2 operating system as an improvement over FAT.

### Step 2 — Extract readable strings

```bash
strings hahahaha.img
```

Notable output:
```
HPFS
OS2WARP_VOL
SECRET.TXT
ENCODING=XOR
c29mdHdhcmljYXtmYWtlX2ZsYWdfbm90X3RoaXN9
< );8.=&,.4'|7
8~5{=+
...
```

Two things stand out: a base64 string and some garbled blobs after it labelled `ENCODING=XOR`.

### Step 3 — Decode the base64 string (fake flag)

```bash
echo "c29mdHdhcmljYXtmYWtlX2ZsYWdfbm90X3RoaXN9" | base64 -d
# softwarica{fake_flag_not_this)
```

As expected — a red herring.

### Step 4 — Brute-force the XOR key

The blobs below the base64 are XOR-encrypted with an unknown single-byte key. Since there are only 256 possible keys, brute-force all of them and look for the `softwarica` prefix:

```python
with open('hahahaha.img', 'rb') as f:
    data = f.read()

blob = data[0xc800:0xc840]

for key in range(256):
    result = ""
    for byte in blob:
        result += chr(byte ^ key)
    if "softwarica" in result:
        print(f"Found it!")
        print(f"Key used : {key}")
        print(f"Decrypted: {result}")
        break
```

Output:
```
Found it!
Key used : 79
Decrypted: softwarica{REDACTED}
```

---

## Key Insight

The `ENCODING=XOR` string in the image was a direct hint that the data was XOR-encrypted. Since the key space for a single-byte XOR is only 256, brute-forcing is trivial — just check which key produces a known plaintext prefix (`softwarica`).

---

## Flag

```
softwarica{REDACTED}
```
