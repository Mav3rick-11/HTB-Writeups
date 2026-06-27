# HTB Writeups

> ⚠️ **For educational purposes only.**
>
> These writeups cover retired HackTheBox machines. All flags have been redacted. Techniques documented here are intended for authorized security research and CTF practice only.

## Completed Machines

| Machine | OS | CVE / Vulnerability | Writeup |
|---------|-----|---------------------|---------|
| Legacy | Windows | CVE-2008-4250 (MS08-067) | [legacy.md](legacy/legacy.md) |
| Lame | Linux | CVE-2004-2687 (distcc) | [lame.md](lame/lame.md) |
| Jerry | Windows | Default Tomcat credentials + WAR upload | [jerry.md](jerry/jerry.md) |
| Cap | Linux | IDOR + Linux capabilities abuse (cap_setuid) | [cap.md](cap/cap.md) |
| Nibbles | Linux | CVE-2015-6967 Nibbleblog file upload + sudo misconfiguration | [nibbles.md](nibbles/nibbles.md) |
| Bashed | Linux | Web shell exposure + sudo lateral move + cron job abuse | [bashed.md](bashed/bashed.md) |

## Skills Demonstrated

- Network enumeration with Nmap (port scanning, service detection, vuln scripts)
- Metasploit module configuration and exploitation
- Shell upgrading (dumb shell → full TTY)
- Linux post-exploitation enumeration (SUID binaries, capabilities, GTFOBins)
- Privilege escalation via SUID nmap interactive mode and Linux capabilities abuse
- Windows SMB exploitation (MS08-067 / EternalBlue era)
- Apache Tomcat credential brute-force and malicious WAR file deployment
- IDOR vulnerability identification and exploitation
- PCAP analysis and plaintext credential harvesting

## Lab Writeups

| Lab | Vulnerability | CVE | Writeup |
|-----|--------------|-----|---------|
| vsftpd Backdoor | Unauthenticated RCE via FTP backdoor | CVE-2011-2523 | [vsftpd_234_backdoor.md](labs/vsftpd_234_backdoor.md) |
| Apache Path Traversal | Double URL encoding bypass | CVE-2021-42013 | [apache_path_traversal_cve2021_42013.md](labs/apache_path_traversal_cve2021_42013.md) |
| Password Cracking | Offline hash cracking with John + rockyou | — | [password_cracking_john.md](labs/password_cracking_john.md) |

## Tools Used

`nmap` · `searchsploit` · `metasploit` · `msfvenom` · `python`
