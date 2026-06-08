# Lab — Offline Password Cracking with John the Ripper

**Tools:** John the Ripper, rockyou.txt
**Context:** Post-exploitation credential harvesting and offline hash cracking
**Vulnerability Type:** Weak password policy / Credential exposure via `/etc/shadow`

---

## Overview

After obtaining root access via a remote exploit, the `/etc/shadow` file was readable. This file contains hashed passwords for all local system accounts. Extracting and cracking these hashes offline demonstrates the downstream risk of a single initial compromise — weak passwords on any account become a pivot point for lateral movement and persistence.

---

## Phase — Hash Extraction

Post-exploitation with root access, the shadow file was retrieved:

```bash
cat /etc/shadow
```

The file contains entries in the format:

```
username:$hash_type$salt$hash:last_changed:min:max:warn:::
```

Hash type identifiers:
- `$1$` = MD5crypt
- `$6$` = SHA-512
- `*` or `!` = account locked, no password set

Accounts with `*` or `!` cannot be cracked — they have no valid password. Focus is on accounts with actual hash values.

Combined `/etc/passwd` and `/etc/shadow` using John's built-in utility to prepare for cracking:

```bash
unshadow /etc/passwd /etc/shadow > combined.txt
```

---

## Phase — Offline Cracking

Ran John the Ripper against the combined file using the rockyou wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt combined.txt
```

John automatically detected the hash types (`sha512crypt`, `crypt(3) $6$`) and began testing candidates from the wordlist.

To display cracked passwords after the session:

```bash
john --show combined.txt
```

---

## Results

Multiple accounts were cracked within the session window, confirming weak password policies across the system. Passwords redacted — the key finding is that common dictionary words with simple substitutions (letters replaced with numbers or symbols) fall quickly against rockyou.txt, which contains over 14 million real-world passwords from historical breaches.

---

## Criticality Assessment

**Rating: High**

Weak passwords compound the impact of any initial compromise:

- **Credential reuse** — users who reuse passwords across systems allow a single cracked hash to unlock multiple accounts or services
- **Lateral movement** — cracked service account credentials (e.g. `msfadmin`, `postgres`) enable pivoting to other systems or databases
- **Persistence** — an attacker with valid credentials can return even after the initial exploit vector is patched
- **Privilege escalation** — a cracked account with sudo rights or group membership elevates access further

---

## Key Takeaways

- Root access to a Linux system means full access to `/etc/shadow` — every account's password is now at risk offline
- `unshadow` combines passwd and shadow into the format John expects — always use it before cracking
- rockyou.txt covers the vast majority of weak passwords used in real environments; if a password falls to rockyou it should be considered compromised immediately
- Hash type matters for cracking speed — MD5crypt (`$1$`) cracks significantly faster than SHA-512 (`$6$`); older systems using MD5 are especially at risk
- Strong password policies (length, complexity, uniqueness) and MFA are the only reliable mitigations — hashed passwords are not protected passwords
