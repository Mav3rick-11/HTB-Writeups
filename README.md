# HTB Writeups

> ⚠️ **For educational purposes only.**
>
> These writeups cover retired HackTheBox machines. All flags have been redacted. Techniques documented here are intended for authorized security research and CTF practice only.

## Completed Machines

| Machine | OS | Difficulty | CVE / Vulnerability | Writeup |
|---------|-----|------------|---------------------|---------|
| Legacy | Windows | Easy | CVE-2008-4250 (MS08-067) | [legacy.md](legacy/legacy.md) |
| Lame | Linux | Easy | CVE-2004-2687 (distcc) | [lame.md](lame/lame.md) |
| Jerry | Windows | Easy | Default Tomcat credentials + WAR upload | [jerry.md](jerry/jerry.md) |

## Skills Demonstrated

- Network enumeration with Nmap (port scanning, service detection, vuln scripts)
- Metasploit module configuration and exploitation
- Shell upgrading (dumb shell → full TTY)
- Linux post-exploitation enumeration (SUID binaries, GTFOBins)
- Privilege escalation via SUID nmap interactive mode
- Windows SMB exploitation (MS08-067 / EternalBlue era)
- Apache Tomcat credential brute-force and malicious WAR file deployment

## Tools Used

`nmap` · `searchsploit` · `metasploit` · `msfvenom` · `python`
