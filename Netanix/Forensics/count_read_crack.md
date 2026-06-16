# Count-Read-Crack

**Category:** Cryptography
**Platform:** Netanix / NxCTF  
**Flag:** `Neta{REDACTED}`

---

## Overview

We are given three files, each with 31 lines mapping a 32-bit message to an 8-bit CRC. The trick is that the **CRC-8 generator polynomial itself is the flag character** for each line. By finding the polynomial that produces the given CRC for each message across all three files, we recover the flag.

---

## The Trick

The challenge hint says: *"each character of the flag is encoded with a custom 8-bit CRC trick"* — one character per line.

For any given line, only a handful of the 256 possible polynomials will produce the observed CRC for the given message. Since all three files encode the **same flag**, we intersect the candidate sets for each line position across all three files. A unique printable ASCII character falls out per position.

---

## Solution

### Step 1 — Implement the CRC-8 function

```python
def crc8(msg, poly):
    reg  = msg << 8
    full = (1 << 8) | poly
    for bit in range(39, 7, -1):
        if (reg >> bit) & 1:
            reg ^= full << (bit - 8)
    return reg & 0xff
```

### Step 2 — Find candidate polynomials for each file

For each line in each file, collect every polynomial (0–255) that produces the observed CRC:

```python
def candidates(samples):
    return [
        [p for p in range(256) if crc8(m, p) == c]
        for m, c in samples
    ]

c1 = candidates(f1)
c2 = candidates(f2)
c3 = candidates(f3)
```

### Step 3 — Intersect candidates across all three files and filter for printable ASCII

```python
flag = ""
for i in range(31):
    inter = set(c1[i]) & set(c2[i]) & set(c3[i])
    printable = [p for p in inter if 32 <= p <= 126]
    flag += chr(printable[0])

print(flag)
```

Output:
```
Neta{REDACTED}
```

> **Note:** One position had two printable candidates (`H` and `3`). Leetspeak context (`F1x3d`) made `3` the obvious correct choice.

---

## Key Insight

The challenge disguises the flag as CRC generator polynomials. Since CRC-8 operates on a single byte and the polynomial space is only 256, brute-forcing all possible polynomials per line is cheap. Using three independent files eliminates ambiguity by narrowing the intersection to a single printable character per position.

---

## Flag

```
Neta{REDACTED}
```
