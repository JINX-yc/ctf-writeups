# Rooted: Nightmare Edition

**Category:** PWN 
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica{REDACTED}`

---

## Overview

We land in a container as a low-privilege user with no sudo password. The path to root involves finding a strangely named file containing a rot8000-encoded password, then escalating privileges.

---

## Solution

### Step 1 — Check sudo permissions

```bash
sudo -l
```

Requires a password we don't have yet.

### Step 2 — Find writable files and directories

```bash
find / -writable -not -path "*/proc/*" -not -path "*/sys/*" 2>/dev/null
```

- `-writable` — lists files and directories the current user can write to
- `-not -path "*/proc/*"` — skips the virtual `/proc` filesystem
- `-not -path "*/sys/*"` — skips the kernel interface `/sys`

This outputs a long list of paths, most being fake flags. One stands out: a file literally named `--flag is here--`.

### Step 3 — Read the suspicious file

```bash
cat "/home/ctf/documents/notes/.drafts/--flag is here--"
```

Output is a cipher using unusual Unicode characters (rot8000 encoding):
```
籲籨籪籶籨籪籨籹类籸籨籮籼籬籪籵籪籽籸类
```

### Step 4 — Decode the cipher

Decode using [rot8000] (a Unicode-aware rotation cipher):

```
REDACTED
```

### Step 5 — Switch to root using the decoded password

```bash
su root
# Password: REDACTED
```

We are now root.

### Step 6 — Find the real flag

List all files inside `/root`:

```bash
find /root -type f 2>/dev/null
```

Three files with "flag" in their name appear. Read each with `cat` until the real flag is found:

```bash
cat /root/<flag-file>
```

---

## Flag

```
softwarica{REDACTED}
```
