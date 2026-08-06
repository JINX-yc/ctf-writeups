# Day - 6

---

**Challenge name:** Overheard at Breakfast

**Category:** OSINT

**Difficulty:** Easy

**Date completed:** 2nd of August 2026

---

## Summary

A screenshot of a Discord conversation between two Byte Lotus Hotel guests contained enough publicly available information to identify a forgotten online profile. The key clue was an email address and a reference to a profile-linking service beginning with the letter "G". Recognizing this as Gravatar, I generated the email's MD5 hash, located the hidden profile, and decoded the Base64-encoded flag displayed on the page.

---

### Step 1: Analyze the conversation

The provided Discord screenshot looked like an ordinary conversation, but several details stood out. Lambo mentioned:

> "I used to use this free tool that let me upload my profile and link other media accounts... Started with a G if I remember correctly."
> 

He also shared his email address:

```
lambobytelotushotel@gmail.com
```

The room description and Mia's hint ("y'all need to actually READ what they said, not just skim it") suggested that the important clues were hidden in the conversation itself rather than elsewhere.

---

### Step 2: Identify the service

The phrase "...upload my profile and link other media accounts..." combined with "Started with a G..." strongly pointed toward **Gravatar**. Another giveaway was the room's category list: OSINT, Social Media, Hashing. Gravatar identifies user profiles using the MD5 hash of an email address.

---

### Step 3: Generate the MD5 hash

Using the email from the conversation:

```bash
echo -n "lambobytelotushotel@gmail.com" | md5sum
```

Output:

```
[REDACTED]
```

Visiting:

```
https://gravatar.com/[REDACTED]
```

revealed Lambo's forgotten Gravatar profile.

---

### Step 4: Decode the prize

The Gravatar profile contained the following string:

```
[REDACTED]
```

This looked like Base64. Decoding it:

```bash
echo "[REDACTED]" | base64 -d
```

returned:

```
THM{[REDACTED]}
```

---

## Flag

```
THM{[REDACTED]}
```

---

## Why this worked

Gravatar generates user profiles based on the MD5 hash of an email address. Since the conversation leaked the email address and hinted at a service beginning with "G", it was possible to recreate the profile URL simply by hashing the email. The hidden profile then revealed a Base64-encoded string, which decoded directly into the flag.

---

## Lessons Learned

- Small pieces of publicly shared information (such as an email address) can be enough to discover forgotten online accounts.
- Always pay attention to challenge hints - in this case, the mention of Hashing and Mia's advice to "READ what they said" pointed directly toward the solution.
- Gravatar uses the MD5 hash of an email address as a public identifier, making it a common OSINT pivot when an email address is known.
- Base64 encoding is not encryption; it is merely an encoding scheme and should always be considered when investigating suspicious-looking strings.
- OSINT challenges often rely more on recognizing patterns and services than on complex technical exploitation.
