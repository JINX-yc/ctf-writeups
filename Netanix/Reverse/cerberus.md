# Cerberus

**Category:** Reverse Engineering 
**Platform:** Netanix / NxCTF  
**Flag:** `NxCTF{REDACTED}`

---

## Overview

A statically-linked x86-64 ELF with three layers of defense ("three heads"): anti-debug checks, multiple decoy flags, and a SIGSEGV-based control flow trick that hides the real validator. The flag is stored XOR-encrypted in a custom ELF section and is decrypted using a key derived from the binary's own bytes at runtime. The solution is to patch a single branch in the binary to bypass the input comparison gate, letting the real decryption path always execute.

---

## Solution

### Step 1 — Initial Recon

```bash
file cerberus.elf
# ELF 64-bit, x86-64, statically linked, not stripped

strings cerberus.elf
```

Notable findings:
- **Four decoy flags** (plaintext bait):
  - `NxCTF{n1c3_try_n0t_th3_r34l_path}`
  - `NxCTF{bo_w4s_b41t_th3_r34l_p4th_15_d1ff3r3nt}`
  - `NxCTF{d3bug_funct10n_d04snt_g1v3_fl4g}`
  - `NxCTF{vm_w4s_w4y_t00_3z_h3r3s_th3_d3c0y}`
- **Custom ELF sections** holding the real encrypted flag:
  - `.cerb_flaglen` = `0x2a` (42 bytes)
  - `.cerb_flag` = 42 encrypted bytes
  - `.cerb_target` = 32 bytes (expected transformed input)
  - `.cerb_bytecode` = 256 bytes (XOR-encrypted VM blob)
  - `.text_hash_anchor` @ `0x494f80` (1 page, zeroed in file — anti-tamper)

### Step 2 — The Three Heads

**Head 1 — Anti-debug ("bites"):** `0x401f50` runs four ptrace/`/proc`-based debugger detection routines OR'd together. If a debugger is detected, execution diverts to a decoy/nope path.

**Head 2 — Decoys ("lies"):** Functions `admin_panel` and `__cerberus_debug_dump`, plus a `0x7f` first-byte special case, print bait banners and decoy flags.

**Head 3 — SIGSEGV trick ("barks"):** For normal input, `main` calls `0x401f70`, which stashes the input pointer, then deliberately writes to the unmapped address `0xdead0000`, raising `SIGSEGV`. A signal handler installed at startup catches the fault and dispatches into the **real validator** at `0x402680`. This hides the validation logic from casual static analysis.

### Step 3 — The Real Validator at `0x402680`

```
key32  = KDF(.text_hash_anchor @ 0x494f80, 0x1000)   ; 32-byte key derived from binary's own bytes
BC     = .cerb_bytecode[i] ^ key32[i & 31]            ; decrypt VM bytecode
t      = cipher_transform(input[0..31])                ; run the VM on input
if t == .cerb_target[0..31]:                           ; SIMD 32-byte compare
    flag[i] = .cerb_flag[i] ^ key32[i & 31]           ; decrypt real flag
    write(1, "ok: ", flag)                             ; print it
else:
    silently return                                     ; wrong input = silence
```

Key observation: **the flag's XOR key is the same `key32` regardless of user input**. The input only controls the gate. If we can skip the comparison, the binary will always decrypt and print the flag.

### Step 4 — Patch the Gate Branch

The conditional jump `je 0x402825` at file offset `0x2797` is the gate. Patch it to NOPs:

```python
data = bytearray(open('cerberus.elf', 'rb').read())

off = 0x2797   # file offset = vaddr - 0x400000 (non-PIE binary)
assert data[off:off+6] == bytes.fromhex('0f8488000000')  # JE rel32
data[off:off+6] = b'\x90' * 6                            # NOP sled

open('patched.elf', 'wb').write(data)
```

### Step 5 — Run the Patched Binary

```bash
chmod +x patched.elf
printf 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\n' | ./patched.elf
```

Output:
```
cerberus // ready
> ok: NxCTF{REDACTED}
```

The SIGSEGV handler still fires → signal dispatches to `0x402680` → the KDF computes `key32` from the unmodified binary → `cipher_transform` runs but the comparison no longer blocks → flag is decrypted and printed.

---

## Why This Works

The anti-tamper KDF reads `.text_hash_anchor` — but since we only patched the `.text` section (not that anchor region), the derived `key32` is identical to the legitimate runtime value. The binary effectively decrypts its own flag for us.

---

## Flag

```
NxCTF{REDACTED}
```
