# Rooted: Advanced Edition

**Category:** PWN 
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica{REDACTED}`

---

## Overview

We are dropped into a Docker container where the filesystem is littered with fake flags. The real flag is hidden inside the Docker image's build history — specifically embedded in a C source file compiled during the build process.

---

## Solution

### Step 1 — Search the container for flags (red herring)

Inside the container:

```bash
grep -r "softwarica{" / 2>/dev/null
```

This returns only fake flags:
```
/home/ctfuser/.secret/secret.txt        fake_y0u_w1sh_th1s_w4s_1t
/tmp/.cache/flag.txt                    fake_keep_hunt1ng_buddy
/tmp/flags/flag.txt                     fake_n1c3_try_wr0ng_pl4c3
/tmp/hidden/data.txt                    fake_cl0s3_but_n0_c1g4r
```

### Step 2 — Inspect the Docker image history from the host

Exit the container and run this on the host machine:

```bash
docker history --no-trunc netanix/wherectf:latest
```

The `--no-trunc` flag prevents Docker from cutting off long commands, revealing the full build instructions.

**Key finding:** The build history shows a `RUN` command that includes the contents of `flag_daemon.c`, which has the real flag hardcoded inside the C source code.

### Step 3 — Extract the flag

The flag is visible directly in the image history output, embedded within the C program source:

```
softwarica{REDACTED}
```

---

## Key Insight

Docker image layers are immutable and store every command run during the build. Using `--no-trunc` exposes the full content of `RUN` commands, including hardcoded secrets that were removed from the final image but still exist in the layer history.

---

## Flag

```
softwarica{REDACTED}
```
