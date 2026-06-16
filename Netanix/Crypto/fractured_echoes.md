# Fractured Echoes

**Category:** Cryptography — Broken DSA / Hidden Number Problem  
**Platform:** Netanix / NxCTF  
**Flag:** `NxCTF{REDACTED}`

---

## Overview

We are given `chall.py` (a signing scheme claiming to be "post-quantum, entropy hardened, state scrambled, and lattice resistant") and `output.txt` (20 signed messages + a ciphertext). Every security claim is false. Three independent weaknesses in the nonce generation expose the secret key via the Hidden Number Problem (HNP), which is then used to XOR-decrypt the ciphertext.

---

## Solution

### Step 1 — Read the signing scheme

```python
p = 2**127 - 1        # Mersenne prime
q = getPrime(120)     # 120-bit group order

class WeakPRNG:
    def __init__(self):
        self.M = [[5,1,0],[0,5,1],[0,0,5]]
        self.S = [random.randint(1,p-1) for _ in range(3)]
    def step(self):
        ns = [sum(self.M[i][j]*self.S[j] for j in range(3)) % p for i in range(3)]
        self.S = ns
        return ns[0]

def sign(msg):
    st  = rng.step()
    k   = bytes_to_long(sha256(long_to_bytes(st)).digest()) % q
    r   = pow(2, k, p) % q
    h   = bytes_to_long(sha256(msg).digest())
    s   = ((h + x*r) * inverse(k, q)) % q
    leak = (k >> 40) ^ ((k & 0xffff) << 12)
    return (r, s, leak)
```

This is essentially DSA: `s ≡ (h + x·r) · k⁻¹ (mod q)`. Recovering any `k` gives `x` directly.

### Step 2 — Exploit weakness #1: `r` leaks `k mod 127`

Because `p = 2¹²⁷ − 1` is a Mersenne prime, `ord_p(2) = 127`. Therefore:

```
r = (2^k mod p) mod q = 2^(k mod 127) mod q
```

Every `r` value is a small power of 2, giving us **7 free bits** of each nonce. Verify:

```bash
python3 check.py
# i=0 r=2^14   i=1 r=2^16   i=2 r=2^116  ...
```

### Step 3 — Exploit weakness #2: the leak gives 64 more bits

The "entropy hardened" leak `(k >> 40) ^ ((k & 0xffff) << 12)` only XORs two non-overlapping regions:

| Leak bits | What is revealed |
|-----------|-----------------|
| 0 – 11    | `k[40..51]` directly (12 bits) |
| 12 – 27   | `k[52..67] ⊕ k[0..15]` (coupled 16 bits) |
| 28 – 79   | `k[68..119]` directly (52 bits) |

Out of 120 bits, **64 are handed over for free**. Combined with the `mod 127` constraint, the residual unknown shrinks to under `2^61`.

### Step 4 — Solve via the Hidden Number Problem (HNP)

Each signature gives a linear relation mod `q`:

```
k_i ≡ A_i + B_i · x   (mod q)
```

where `A_i = h_i / s_i` and `B_i = r_i / s_i` (mod q).

Using 20 samples, set up a Boneh–Venkatesan / Kannan-embedded lattice (22 dimensions) and run LLL reduction:

```python
from fpylll import IntegerMatrix, LLL

inv127 = pow(127, -1, q)
for i in range(n):
    nu = (e_list[i] - K_known[i]) % 127
    Ap.append(inv127 * ((A_list[i] - K_known[i] - nu) % q) % q)
    Bp.append(inv127 * B_list[i] % q)

SCALE, M_val, center = 2**60, 2**120, 2**68 // 254
dim = n + 2
L = IntegerMatrix(dim, dim)
for i in range(n):   L[i, i] = q * SCALE
for i in range(n):   L[n, i] = Bp[i] * SCALE
L[n, n] = 1
for i in range(n):   L[n+1, i] = (Ap[i] - center) * SCALE % (q * SCALE)
L[n+1, n+1] = M_val

LLL.reduction(L)
# x falls out of the second-to-last column
```

Output:
```
X_SECRET = 478316562975177751731920142610365046
# Verified: all 20 reconstructed k_i match leaked bits and mod-127 constraint
```

### Step 5 — Decrypt the ciphertext

The ciphertext from `output.txt` is 37 bytes (non-multiple of 16 → stream cipher / XOR). Derive the keystream as `sha256(x)`:

```python
import hashlib

x  = 478316562975177751731920142610365046
ct = bytes.fromhex("fb6e478ac03ba60b48a3733752d59cfb83b25c02ed906857d821841ad6619d07c76370b6fb")
key    = hashlib.sha256(x.to_bytes((x.bit_length()+7)//8, 'big')).digest()
stream = (key * 2)[:len(ct)]
print(bytes(c ^ s for c, s in zip(ct, stream)).decode())
```

Output:
```
NxCTF{REDACTED}
```

---

## Key Insights

Three independent vulnerabilities compounded to make the scheme trivially broken:
1. Using `2^k mod p` with a Mersenne prime leaks `k mod 127` via discrete log.
2. The "entropy hardened" XOR-based leak directly exposes 64 of 120 key bits.
3. With 20 HNP samples and ~61 unknown bits per sample, a 22-dimensional LLL lattice solves the rest.

---

## Flag

```
NxCTF{REDACTED}
```
