# Abyss

**Category:** Reverse Engineering — Android / Custom VM  
**Platform:** Netanix / NxCTF  
**Flag:** `NxCTF{cust0m_vm_4nd_5b0x_h4rd_!}`

---

## Overview

An Android APK presents a single 32-character text input and a "validate" button. Internally it runs a custom register-based VM over XOR-encrypted bytecode. The VM applies a 4-round substitution-permutation network (SPN) per byte of input, comparing the result against a `TARGET` array. Since each byte is processed independently and every operation is bijective, the key can be recovered by brute-forcing all 256 values per position.

---

## Solution

### Step 1 — Initial Recon

```bash
unzip -l abyss.apk
# AndroidManifest.xml, classes.dex (8 KB), assets/ai_notice.txt, resources.arsc

strings classes.dex
# BC_ENC, BC_KEY, S_BOX, ROUND_KEYS, STR_TABLE, TARGET,
# "32 character key", "accepted", "nope. r=", "wrong length (expected 32)"
```

The string table reveals the architecture: encrypted VM bytecode (`BC_ENC`/`BC_KEY`), an S-box, round keys, and a target comparison buffer.

### Step 2 — Understand the Validation Logic

Decompiled `MainActivity.check()`:

```java
String s = et.getText().toString();
if (s.length() == 32) {
    int r = run(s.getBytes());  // VM interpreter
    if (r == 1) → "accepted"
    else        → "nope. r=" + r
} else → "wrong length (expected 32)"
```

The goal is to make `run(input)` return `1`.

### Step 3 — Decrypt the Bytecode

The VM's bytecode `BC` is derived in the static initializer with a repeating-key XOR:

```
BC[i] = BC_ENC[i] ^ BC_KEY[i & 15]
```

### Step 4 — Disassemble the VM

`run()` is a 16-register integer VM. The opcode set:

| Opcode | Mnemonic | Effect |
|--------|----------|--------|
| `0x10` | `LDI r, imm32` | Load immediate |
| `0x12` | `LDinput r, ri` | `r = input[reg[ri]]` |
| `0x13` | `SBOX r, ri` | `r = S_BOX[reg[ri] & 0xff]` |
| `0x14` | `RK r, a, b` | `r = ROUND_KEYS[reg[a]][reg[b]]` |
| `0x15` | `TGT r, ri` | `r = TARGET[reg[ri]]` |
| `0x20–0x22` | `ADD/SUB/MUL` | Arithmetic |
| `0x23–0x25` | `XOR/AND/OR` | Bitwise |
| `0x26–0x27` | `SHL/SHR imm` | Shifts |
| `0x28` | `ANDI imm` | AND with immediate |
| `0x29–0x2A` | `INC/DEC` | ±1 |
| `0x30` | `CMP a, b` | Set eq/lt flags |
| `0x40–0x42` | `JMP/JE/JNE` | Control flow |
| `0xF0/0xF1` | `RET 1/RET 0` | Accept / Reject |

### Step 5 — Understand the Per-Byte Transform

Disassembling reveals that for each of the 32 input positions, the same 4-round SPN is applied:

```
for r in 0..3:
    x = (x + ROUND_KEYS[r][i]) & 0xff   # add round key
    x = S_BOX[x]                         # S-box substitution
    x = ((x << 4) | (x >> 4)) & 0xff    # nibble swap
require x == TARGET[i]
```

Every operation is bijective on `[0, 256)`, so for each position there is **exactly one** input byte that maps to `TARGET[i]`.

### Step 6 — Brute-Force Solver

No cryptanalysis needed — try all 256 values per position:

```python
def forward(x, i):
    for r in range(4):
        x = (x + RK[r][i]) & 0xff
        x = S_BOX[x]
        x = ((x << 4) | (x >> 4)) & 0xff
    return x

flag = bytes(
    next(x for x in range(256) if forward(x, i) == TARGET[i])
    for i in range(32)
)
print(flag.decode())
```

Verified by re-running the full VM emulation in Python — the recovered string returns `1` (accepted).

---

## Key Insight

The SPN operates **one byte at a time**, making each position independent of the others. This eliminates all diffusion between positions and reduces a "custom cipher" to 32 independent lookup problems, each solvable with a 256-iteration brute force.

---

## Flag

```
NxCTF{cust0m_vm_4nd_5b0x_h4rd_!}
```
