# Error Message

**Category:** Cryptography

**Flag format:** `softwarica{}`

The challenge gave us this message:

> 
> 
> 
> Guys i have encrypted something in this softwarica{codwmofgmxefdnvytglmspigdodhirgydom} but i cant get through this what it was so help me find out..
> I also look back to the softwarica college hehehehhe.......
> 

The encrypted flag:

```
softwarica{codwmofgmxefdnvytglmspigdodhirgydom}
```

Along with this hint:

> I also look back to the softwarica college hehehehhe.......
> 

---

## Finding the Key

The words **“look back”** suggested reversing `softwarica`, which gave us `acirawtfos` as the key.

---

## Decryption

The cipher was **Bifid**. We only needed to decrypt the text inside the flag wrapper, using these settings:

- **Ciphertext:** `codwmofgmxefdnvytglmspigdodhirgydom`
- **Key:** `acirawtfos`

Bifid uses a 5x5 letter square and mixes the letters row and column coordinates. Decrypting with these settings gave the message:

```bash
[REDACTED]
```

After wrapping the decoded message with `softwarica{}` , we get the flag:

```bash
softwarica{REDACTED}
```