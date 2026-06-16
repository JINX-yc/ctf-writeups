# Eew What Is This

**Category:** Reverse Engineering  
**Platform:** Netanix / NxCTF  
**Flag:** `flag{REDACTED}`

---

## Overview

We are given a heavily obfuscated C source file. The flag is hidden by shuffling character checks across the code using array indexing. Preprocessing the file expands all macros to reveal the real logic, and re-ordering the index-to-character mapping gives us the flag directly.

---

## Solution

### Step 1 — Preprocess the obfuscated source

Run the C preprocessor to expand all macros and reveal the underlying logic:

```bash
gcc -E obfuscated.c > expanded.c
```

This strips away all the macro obfuscation and shows the actual C code.

### Step 2 — Locate the core logic

Search for assignment patterns to find the main function:

```bash
grep -n "=" expanded.c | head
```

The decompiled output reveals a `main` function that checks each position of an array `e[]` independently against hardcoded character values.

### Step 3 — Reconstruct the flag

Each check in the code follows the pattern `e[index] == 'char'`. By collecting all index-to-character mappings and sorting by index (0–19), the flag assembles directly:

| Index | Char |
|-------|------|
| 0  | f  |
| 1  | l  |
| 2  | a  |
| 3  | g  |
| 4  | {  |
| 5  | b  |
| 6  | 8  |
| 7  | 8  |
| 8  | 0  |
| 9  | c  |
| 10 | 7  |
| 11 | 9  |
| 12 | 7  |
| 13 | 2  |
| 14 | 9  |
| 15 | 4  |
| 16 | 9  |
| 17 | 5  |
| 18 | 2  |
| 19 | }  |

---

## Key Insight

The obfuscation relied entirely on macros to hide the structure of the program. `gcc -E` is a simple but powerful tool that collapses all macro layers, exposing the raw logic without needing a decompiler. Once visible, the flag was just a matter of reading the character comparisons in index order.

---

## Flag

```
flag{REDACTED}
```
