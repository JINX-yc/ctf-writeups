# Time Machine

**Category:** Misc  
**Platform:** Netanix / NxCTF  
**Flag:** `softwarica{REDACTED}`

---

## Overview

We are given a Docker image to explore. The flag is hidden inside a shell script that is only accessible to a specific user baked into the image.

---

## Solution

### Step 1 — Pull the Docker image

```bash
docker pull keeedhacker/timemachine
```

### Step 2 — Inspect the image build history

```bash
docker history keeedhacker/timemachine
```

Key observations from the history:
- `COPY flag.sh /opt/flag.sh` — the flag may be stored in this script
- `chmod 700 /opt/flag.sh` — only the owning user can execute it
- `echo "keed:error." | chpasswd` — user `keed` exists with password `error.`

### Step 3 — Run the container and switch to user `keed`

```bash
docker run -it keeedhacker/timemachine /bin/bash
su keed
# Password: error.
```

### Step 4 — Read the flag script

Now that we are `keed`, we have permission to read `/opt/flag.sh`:

```bash
cat /opt/flag.sh
```

Output:
```bash
echo "softwarica{REDACTED}" > /tmp/flag.txt
```

The flag is embedded directly in the script.

---

## Flag

```
softwarica{REDACTED}
```
