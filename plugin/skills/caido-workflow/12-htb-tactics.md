# 12 — HTB-Specific Tactics

The patterns, tells, and meta-skills unique to the Hack The Box platform. Most of this generalizes to any CTF; some of it is specifically how HTB box authors think.

---

## 1. Platform Workflow

### 1.1 VPN connection
```bash
sudo openvpn machines_us-4.ovpn
# verify:
ip a show tun0
ping -c 2 10.129.0.1
```
- HTB's VPN gives you a `10.10.14.x` (or similar) IP on `tun0`.
- Box IPs are typically `10.10.x.x` (retired) or `10.129.x.x` (active).
- `tun0` IP is your attacker IP for reverse shells. Confirm before generating payloads.

### 1.2 Box lifecycle
- **Active boxes** — not retired, not in academy. No public writeups (allegedly). Worth the most points.
- **Retired boxes** — full IppSec videos and writeups exist; great for learning.
- **Starting Point** — guided intro path; do all of these first if new to HTB.
- **Seasonal** boxes — released periodically; same playbook applies.

### 1.3 Submitting flags
- User flag: `/home/<user>/user.txt` or `C:\Users\<user>\Desktop\user.txt`.
- Root flag: `/root/root.txt` or `C:\Users\Administrator\Desktop\root.txt`.
- Submit on the box's web page. Returns the next prompt or "owned" status.
- Flags rotate — yours is unique to your VPN session.

### 1.4 Reset / spawn
- "Reset" reboots the machine. Use sparingly — costs spawn time, can break others' progress.
- "Cancel/release" lets others spawn. Be courteous on shared boxes.

### 1.5 Pwnbox (HTB's hosted Kali)
- Browser-accessible.
- No reason to use over local Kali except convenience or limited host hardware.

---

## 2. Reading the Box Card

Before any scanning, look at:
- **Name** — almost always a clue. `Granny`, `Grandpa` → old Windows. `Bashed` → Linux web shell theme. `Doctor` → medical/healthcare app. `Forge` → SSRF/forging. `Photobomb` → image-processing related.
- **OS** — Linux/Windows.
- **Difficulty** — informs depth of methodology.
- **User-rated difficulty** — sometimes more accurate than the official rating.
- **Release date** — older boxes use older tech (PHP 5, IE-era headers, etc.).
- **Creator profile** — some authors have signature styles (e.g., heavy AD, or always-LFI).
- **Tags** (if present) — reveal classes hinted.

The box description text is rare on retired boxes but always read it on active/seasonal.

---

## 3. Default Methodology Per Difficulty

### 3.1 Easy boxes (typical pattern)
- One web app or one network service holds the foothold.
- Foothold tends to be: default creds, public CVE, simple LFI/SQLi/cmd-injection, anonymous SMB share.
- User → root via *one* of: clear sudo entry, SUID GTFOBins, writable cron, easy kernel CVE.
- Time budget: 1–2 hours from spawn to root.

If you're past 2 hours on an Easy box without foothold, you missed:
- A vhost / hostname
- A directory not in your wordlist
- A clue in the box name
- A service version that maps to a public exploit

### 3.2 Medium boxes
- Often two foothold steps or chained attacks (e.g., LFI → log poisoning → RCE).
- Frequently require crafting custom payloads (not just running sqlmap).
- Privesc usually involves credential reuse or service misconfig.
- 2–4 hours.

### 3.3 Hard boxes
- Multiple users (often 3+) with lateral pivots.
- May require source code review of a custom app.
- Privesc may require chained ACL abuses (BloodHound essential).
- 4–10+ hours.

### 3.4 Insane boxes
- Custom protocols, kernel exploits, complex AD trust attacks, OffSec-grade source review.
- 10+ hours, sometimes days.
- Treat as a learning project, not a sprint.

---

## 4. Hostname & VHost Discipline

The single most-frequent fail on HTB. Whenever you see a hostname:

1. From `nmap` SSL cert (CN, SAN)
2. From `whatweb` / response page title / footer
3. From a redirect Location header
4. From a service banner (SMTP, SMB, MSSQL)
5. From the box name itself (often `<name>.htb`)

→ **Add to /etc/hosts immediately**:
```bash
echo "<ip> name.htb www.name.htb" | sudo tee -a /etc/hosts
```
Many web apps virtual-host on the IP and serve a 403/blank page until you Host:-header them correctly. Skipping this step costs hours.

After adding, vhost-fuzz against each known hostname:
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
     -u http://name.htb -H "Host: FUZZ.name.htb" \
     -fs <baseline>
```

---

## 5. The "I'm Stuck" Reset Procedure

When 60+ minutes into an Easy / Medium and no foothold:

1. **Re-list every clue** in your notes. Read aloud.
2. **Pick the least-explored clue** — go deep on it.
3. **Re-run** scans with bigger wordlists / different perspectives:
   - `directory-list-2.3-medium.txt` instead of `common.txt`
   - Different extensions matching the *actual* stack
   - UDP scan if you skipped it
   - Vhost fuzz if you didn't
4. **Look up the framework / version** on Exploit-DB and GitHub. Not just CVEs — known footguns.
5. **Try anonymous / guest / default** on every authenticated service. Always.
6. **Inspect every JS file** for endpoints / secrets.
7. **Inspect cookies/tokens** — base64? JWT? signed?
8. **Walk through every form / workflow** as a hostile user. What if I skip a step? What if I tamper with a hidden input?
9. **Box description / tags / forum** — read for hints (without spoilers if active box).
10. **Step away** for 15 minutes. Come back fresh.

If still stuck, the meta-clue is "I'm not enumerating something I should be." Re-enumerate from scratch. The signal is in the noise.

---

## 6. Common HTB-Box Themes & What They Imply

| Pattern in name / theme | Likely vector |
|-------------------------|---------------|
| "Grandpa", "Granny" | Old Windows IIS, MS exploits, classic SMB |
| "Bank", "Vault" | Authentication-heavy, JWT, session tampering |
| "Bash*", "Shell*" | Bash injection / web shell |
| "Forge" | SSRF / forging requests |
| "Photo*", "Snap*" | Image processing → ImageMagick, EXIF, file upload |
| "*Doctor", *medical | App with custom routes, SSTI/Email |
| "Time" | Time-based vulnerabilities, race conditions, Cron |
| "Ldap*", "Active*" | AD-focused |
| "*Ping", "Net*" | Command injection on a network tool |
| "Office", "Doc*" | Office macro / template injection |
| Numbers (Lame, Devel) | Easy intro boxes; classic tech |

---

## 7. Automating Recon Setup

A `box-init.sh` saves 5 minutes per box:
```bash
#!/usr/bin/env bash
set -euo pipefail
NAME="${1:?usage: $0 name ip}"
IP="${2:?usage: $0 name ip}"
DIR=~/htb/$NAME
mkdir -p "$DIR"/{nmap,web,loot,exploits,shells}
cd "$DIR"
sudo bash -c "echo '$IP $NAME.htb' >> /etc/hosts"
echo "[*] Quick TCP all-port scan..."
sudo nmap -p- --min-rate 5000 -T4 -Pn -oN nmap/all.txt "$IP" &
PID=$!
echo "[*] Quick top-1000 with -sCV..."
sudo nmap -sCV --min-rate 1000 -T4 -Pn -oN nmap/quick.txt "$IP"
wait $PID
PORTS=$(grep -oP '\d+/open' nmap/all.txt | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')
[ -n "$PORTS" ] && sudo nmap -sCV -p"$PORTS" -oN nmap/services.txt "$IP"
sudo nmap -sU --top-ports 100 -oN nmap/udp.txt "$IP" &
echo "[*] Done initial recon. Workspace: $DIR"
```

Tmux template (so panes are pre-laid for the box):
```bash
tmux new-session -d -s "$NAME" -n main 'cd ~/htb/'$NAME
tmux split-window -h -t "$NAME":main
tmux split-window -v -t "$NAME":main.1
tmux split-window -v -t "$NAME":main.0
tmux attach -t "$NAME"
```

---

## 8. Common Foothold Patterns by Service

### 8.1 Port 80 only (no SMB, no AD)
- Fingerprint stack → known CMS? → use `wpscan`/`joomscan`/`droopescan`.
- Default creds on admin panels (`admin:admin`, `admin:password`).
- Directory fuzz with the right extensions.
- Look for upload endpoints → upload PHP/JSP/ASPX shell.

### 8.2 Port 80 + SMB
- Try anonymous on SMB → grab files (often a key, doc with creds, source code).
- If anonymous reads a `.bak` or source file → review for creds / vulns.

### 8.3 Port 21 (FTP)
- Anonymous? Look for `id_rsa`, source code, web roots.
- Writable directory + FTP serves the web root → upload a webshell.

### 8.4 Port 25 (SMTP)
- VRFY/EXPN for user enum.
- If accompanied by a web mail interface (Roundcube, etc.), try discovered users.

### 8.5 Port 161 (SNMP)
- `public` or `private` community → snmpwalk → leaks usernames, software, processes (with cmdline arguments — sometimes including credentials).

### 8.6 AD ports (88, 389, 445)
- Kerbrute user enum → AS-REP roast → crack → user.
- Then BloodHound the path to admin.

### 8.7 Database ports (1433, 3306, 5432, 6379, 27017)
- Default creds, anonymous binds, then RCE primitives (`xp_cmdshell`, `COPY ... PROGRAM`, Redis SSH-key).

---

## 9. Common Privesc Patterns by OS

### 9.1 Linux Easy
- `sudo -l` → GTFOBins → root
- SUID binary → GTFOBins → root
- Writable cron script → reverse shell on next tick → root
- pwnkit (`CVE-2021-4034`) on un-patched older boxes
- Credential reuse from web app config (`wp-config.php`, `config.php`, `.env`)

### 9.2 Windows Easy
- AlwaysInstallElevated → MSI → SYSTEM
- Service binary writable → replace + restart → SYSTEM
- SeImpersonate via service account → PrintSpoofer/GodPotato → SYSTEM
- Credentials in `Unattend.xml` / GPP `Groups.xml`

### 9.3 AD Easy/Medium
- AS-REP roast → user crack → second user
- Kerberoast → svc account crack → admin
- ESC1/ESC8 (ADCS) → cert → DA
- Generic ACL edge in BloodHound → write/owner abuse → DA

---

## 10. Pwnbox vs Local — Tradeoffs

| Aspect | Pwnbox | Local Kali |
|--------|--------|-----------|
| Setup | None | Hours |
| Customization | Limited | Total |
| GPU cracking | No | Yes |
| Persistence | Limited (cleared between sessions) | Yes |
| Performance | Variable | You control |
| Tools updated | HTB does it | You do it |

For learning: Pwnbox is fine. For long-haul work and real engagements: local.

---

## 11. Forum & Discord Etiquette

- HTB forum: hints only, never full writeups for active machines.
- Discord: same.
- IppSec videos: only for retired boxes; otherwise spoiler.
- Reddit / external writeups: same — retired only is the norm.
- If you ask for help, share what you've *done* and what you're *seeing* — don't just say "stuck on box X."

---

## 12. Tracking Progress

Keep a per-box log AND an aggregate stats file:
```
~/htb/_stats.md
- 2024-01-15  Lame      Easy   Linux    foothold:14m   root:32m
- 2024-01-16  Legacy    Easy   Win      foothold:08m   root:18m
- 2024-01-18  Devel     Easy   Win      foothold:25m   root:1h05m
```

Trend graphs: time-to-root over time. The number should drop. If it doesn't, you're hitting a methodology gap — go deeper on the recurring stuck-points.

---

## 13. After the Box

Habitual post-box ritual:

1. **Submit** flags.
2. **Read the official writeup** (if retired).
3. **Watch IppSec's video** (if available) — even if you rooted it. He'll show you 3 things you didn't know.
4. **Update your skill notes** (`skill.md`) — add any new technique.
5. **Diff your time** against the writeup's expected time. Faster → consolidate technique. Slower → identify the bottleneck.
6. **Note the lesson** in `~/htb/_lessons.md`. One sentence per box. Re-read monthly.

---

## 14. HTB Academy Path

If you're new, work the Pentester path:
1. Penetration Testing Process
2. Network Enumeration with Nmap
3. Footprinting
4. Information Gathering — Web Edition
5. Vulnerability Assessment
6. File Transfers
7. Shells & Payloads
8. Using Web Proxies
9. Attacking Common Applications
10. Login Brute Forcing
11. SQL Injection Fundamentals
12. SQLMap Essentials
13. Cross-Site Scripting (XSS)
14. File Inclusion / Directory Traversal
15. File Upload Attacks
16. Command Injections
17. Web Attacks
18. Attacking Common Services
19. Linux Privilege Escalation
20. Windows Privilege Escalation
21. Active Directory Enumeration & Attacks
22. Documentation & Reporting

Each module ends with cubes (boxes). They're cheap practice with structured learning.

---

## 15. Anti-Patterns Specific to HTB

- Spawning a new box every time you get stuck — guarantees you never learn the hard lessons.
- Treating boxes as isolated puzzles. They're a corpus; cross-box pattern recognition is the skill.
- Reading writeups before genuine attempts. You can't develop intuition by reading answers.
- Ignoring retired boxes. The OSCP-prep list is *the* curriculum.
- Skipping "boring" boxes (Lame, Legacy, Devel). They teach foundational tech you can't skip.
- Not noting your shortcuts. The next box you do will reuse half this one's techniques.

---

*HTB is a flight simulator for pentesting. Treat it that way: methodical practice over years, not a leaderboard sprint.*
