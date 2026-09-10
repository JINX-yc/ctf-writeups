# Defense Nepal

**Category:** Cryptography

**Flag format:** `softwarica{}`

The challenge gave us this message:

> Our Nation is In trouble so Show your Power to defense your nation from back ........ We have got the message from Our Honorable PM Balen Shah:
> 

```
3HHNRD5UE3UHR0LUYH0VS4H3YTUYY1CS033UC34F5VH
```

---

## **Finding the Cipher and Key**

The word “**defense**” pointed toward the **Redefence cipher**, and `NEPAL` looked like a reasonable key to try based on the title.

Putting the ciphertext into [dCode’s Redefence decoder](https://www.dcode.fr/redefence-cipher) with `NEPAL` as the key didn’t give anything readable, though. There was one more clue in the description: **“from back.”**

---

## Reversing the Message

That hint suggested reversing the ciphertext backward:

```
HV5F43CU330SC1YYUTY3H4SV0HYUL0RHU3EU5DRNHH3
```

---

## Decryption

Then it was back to dCode:

- Replace the original ciphertext with the reversed version above.
- **Use the cipherkey** : `NEPAL`.

This time, the output made sense:

```
[REDACTED]
```

Adding the flag wrapper gives:

```
softwarica{REDACTED}
```

So the missing piece wasn’t a different key. It was reversing the ciphertext **before** decrypting it.