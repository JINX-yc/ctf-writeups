# Depths of Reverse

**Category:** Reverse Engineering  
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica_ctf{REDACTED}`

---

## Overview

We are given an ELF binary. Running it with the wrong input prints a fake flag. The real flag is XOR-encoded in the binary and only printed when the correct numeric input is supplied. The key is found by analyzing the assembly.

---

## Solution

### Step 1 — Run `strings` on the binary

```bash
strings <binary>
```

This reveals:
- A fake flag (red herring)
- Several short, garbled strings that appear to be XOR-encoded (e.g. `)5<.-;(39;`)

### Step 2 — Identify the XOR key

The encoded strings appear to start with `softwarica`. We can recover the XOR key by XORing the first byte of a known plaintext against the first byte of the ciphertext:

```python
target = b")5<.-;(39;"
known  = b"softwarica"

key = target[0] ^ known[0]  # 0x29 ^ 0x73 = 0x5a
print(hex(key))  # 0x5a
```

The key is `0x5A`.

### Step 3 — Decode all XOR-encoded strings

```python
key = 0x5a
strings = [b")5<.-;(39;", b"9.<!(i,i;6?>", b">?*.2)", b"(i,i()i", b"i4=34ii(k4='"]

for s in strings:
    print(bytes([b ^ key for b in s]).decode())
```

This reveals the components of the flag but the binary still needs the correct input to print it.

### Step 4 — Disassemble the binary to find the input check

```bash
objdump -d <binary>
```

In the `main` function, look for a comparison instruction:

```asm
cmp    $0x6042, %edi
```

In x86-64 calling convention, the first integer argument is passed in `%edi`. This line compares the user's input against the hardcoded constant `0x6042`.

Convert `0x6042` to decimal:

```
0x6042 = 24642
```

### Step 5 — Run the binary with the correct input

```bash
chmod +x <binary>
./<binary>
# Enter: 24642
```

---

## Program Flow

```
your input
│
▼
cmp against 0x6042 (24642)
│
├─ WRONG ──► print fake flag ──► exit
│
└─ RIGHT ──► XOR decode loop (key=0x5a)
                              │
                              ▼
                     flag built in stack memory
                              │
                              ▼
                     printf("FLAG: %s", flag)
```

---

## Flag

```
softwarica_ctf{REDACTED}
```
