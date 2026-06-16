# Automata

**Category:** Reverse Engineering  
**Platform:** Netanix / NxCTF  

---

## Overview

We are given a binary that implements a custom virtual machine (automata). Each flag character is built from a sequence of VM opcodes. We analyze it in Ghidra, then write a Ghidra script to emulate the VM and extract the flag directly.

---

## Solution

### Step 1 — Import into Ghidra and auto-analyze

1. Open Ghidra and create a new project
2. Drag the `automata` binary into the project
3. Let Ghidra auto-analyze the binary and wait for it to complete

### Step 2 — Locate `main()` via the Strings window

1. Go to **Window → Defined Strings**
2. Filter for the string `feed the machine`
3. Double-click the result to jump to it in the Listing view
4. Right-click → **References → Show References to Address**
5. This leads us to `main()`

### Step 3 — Open the Decompiler

Go to **Window → Decompiler**.

Reading the decompiled output reveals that the binary iterates over a block of bytecode (`0x102160` to `0x102436`), each 3-byte instruction consisting of: `opcode`, `dst register`, `immediate value`. The program uses these instructions to build each flag character and compare it against expected values.

### Step 4 — Write a Ghidra script to emulate the VM

Open **Window → Script Manager → New Script → Jython** and paste the following:

```python
PROG_START = 0x102160
PROG_END   = 0x102436

def ror8(v, n):
    n &= 7
    return ((v >> n) | (v << (8 - n))) & 0xFF

addr = toAddr(PROG_START)
end  = toAddr(PROG_END)
flag = []
ops  = []
collecting = False

while addr.compareTo(end) < 0:
    op  = getByte(addr) & 0xFF
    dst = getByte(addr.add(1)) & 3
    imm = getByte(addr.add(2)) & 0xFF
    addr = addr.add(3)

    if op == 0x10:
        ops = []
        collecting = True
    elif op == 0x70 and collecting:
        val = imm
        for o, i in reversed(ops):
            if   o == 0x41: val = (val - i) & 0xFF
            elif o == 0x51: val = (val + i) & 0xFF
            elif o == 0x31: val = (val ^ i) & 0xFF
            elif o == 0x60: val = ror8(val, i)
        flag.append(chr(val))
        collecting = False
    elif collecting:
        ops.append((op, imm))

print "Flag:", "".join(flag)
```

### Step 5 — Run the script

Execute the script inside Ghidra. The script emulates the VM's opcode loop in reverse, undoing each transformation to recover each flag character.

The output prints the full flag.

---

## VM Opcode Reference

| Opcode | Operation       |
|--------|----------------|
| `0x10` | Start new char  |
| `0x41` | Subtract `imm`  |
| `0x51` | Add `imm`       |
| `0x31` | XOR `imm`       |
| `0x60` | Rotate right by `imm` bits |
| `0x70` | Finalize / emit char |

---

## Flag

> Printed by the Ghidra script after emulating the VM bytecode.
