# Simple??

**Category:** Cryptography
**Platform:** Netanix / NxCTF  

---

## Overview

We are given an image file containing symbols from an obscure cipher alphabet. The image also hides metadata with a Vigenère-encoded ciphertext. Identifying the symbol alphabet gives us the key, which we then use to decode the ciphertext.

---

## Solution

### Step 1 — Identify the Symbol Cipher

The challenge image contains unusual symbols. Reverse image searching (e.g., on Google Images) leads to [dcode.fr](https://www.dcode.fr), which lists dozens of symbol cipher alphabets.

Cross-referencing the shapes against the dcode.fr cipher list identifies them as the **Mourier Alphabet**.

Reference: [https://www.dcode.fr/mourier-alphabet](https://www.dcode.fr/mourier-alphabet)

### Step 2 — Inspect the Image Metadata

Run `exiftool` on the challenge image to check for hidden metadata:

```bash
exiftool <challenge_image>
```

Key finding in the output:
```
Image Description : Deck_Q1Gp3cd_4Rd_Kv1fqm(perhaps need a key?)
```

This is a Vigenère-encoded ciphertext, and the hint confirms a key is needed.

### Step 3 — Decode the Mourier Alphabet Symbols

Using the dcode.fr Mourier Alphabet decoder, translate the symbols shown in the challenge image. The decoded output is:

```
REDGORILLAZ
```

This is the Vigenère key.

### Step 4 — Decode the Vigenère Ciphertext

Apply the key `REDGORILLAZ` to the ciphertext from the image metadata using a Vigenère decoder (e.g., [dcode.fr/vigenere-cipher](https://www.dcode.fr/vigenere-cipher)):

```
Ciphertext : Deck_Q1Gp3cd_4Rd_Kv1fqm
Key        : [REDACTED]
```

The decoded output is the flag.

---

## Attack Chain

```
Challenge image
    │
    ├─ Symbols → Mourier Alphabet decoder → Key: [REDACTED]
    │
    └─ exiftool → Image Description → Vigenère ciphertext
                                              │
                                    Vigenère decrypt (key=REDACTED)
                                              │
                                           FLAG
```

---

## Key Insight

This challenge layers two encoding steps: an obscure symbol alphabet to conceal the key, and Vigenère encryption on the actual ciphertext hidden in metadata. Neither step is strong on its own — the challenge relies on the obscurity of the Mourier Alphabet to hide the key.

---

## Flag

> Obtained by Vigenère-decoding the `Image Description` metadata field using `[REDACTED]` as the key.
