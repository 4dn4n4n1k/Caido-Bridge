# 00 — Mindset & Workflow

The mechanics of pentesting can be learned in months. The mindset takes years. This file is what separates the operator who finishes Easy boxes in 90 minutes from the one who finishes in 30 — and from the one who solves Hard boxes others give up on.

---

## 1. Core Principles

### 1.1 Enumeration is 80% of the box
Most failed attempts are failures to enumerate, not failures to exploit. Before you reach for an exploit, you should know:
- Every open port and its service version
- Every directory and file on every web service
- Every vhost and subdomain
- Every parameter on every script
- Every config file readable by the current user
- Every running process, every cron job, every SUID binary

If you've done all of that and you're stuck, **you missed something**. Re-run with bigger wordlists, different protocols, different perspectives. The vuln is there.

### 1.2 Assume nothing, verify everything
- "I scanned all ports" — did you scan UDP? Did you scan TCP 1-65535 not just top 1000?
- "There's no DNS" — did you actually `dig`?
- "The login is rate-limited" — did you check from a different source IP / endpoint?
- "Anonymous FTP is disabled" — did you try with a real username?

The boxes you fail are the ones where you assumed.

### 1.3 One detail is enough
You don't need to find every vulnerability. You need to find one *exploitable* path. A single forgotten backup file, a single weak credential, a single unsanitized parameter ends the box.

### 1.4 Pivot every credential
A credential found in one place must be tried in every other place:
- SSH, FTP, SMB, RDP, WinRM, MSSQL, MySQL, PostgreSQL
- Web logins (admin panel, CMS, mail, monitoring)
- Encrypted archives, KeePass DBs, SSH key passphrases

Reuse is the #1 way users-on-target become root-on-target.

### 1.5 Loop tightly
Recon → Enumerate → Hypothesize → Exploit → Loot → Repeat
After every step, ask: *what new attack surface did this open?*. New creds open new logins. New shells open new file systems. New file systems open new creds. Round and round.

---

## 2. Phases of an Engagement

### 2.1 Pre-engagement (real-world)
- Confirmed scope (IP ranges, domains, hours, allowed tactics)
- Signed Statement of Work / authorization letter
- Emergency contacts
- Reporting format & timelines
- Out-of-scope assets explicitly listed

(For HTB this collapses to: VPN connected, target IP confirmed.)

### 2.2 Reconnaissance
- Passive: OSINT, public DNS, certificate transparency, code repos, employee LinkedIn
- Active: ping sweep, port scan, service version, banner grab

### 2.3 Enumeration
- Per-service deep dive
- Web: vhosts, dirs, params, source code, JS files
- AD: user enum, group enum, ACL enum, BloodHound

### 2.4 Vulnerability identification
- Map services to known CVEs
- Map functionality to vuln classes (file upload → RCE; login → auth bypass; search → injection)

### 2.5 Exploitation
- Always test in lab if PoC is sketchy
- Read the exploit before running it (check for backdoors)
- Tweak the payload — public PoCs often need work

### 2.6 Post-exploitation
- Stable shell
- Loot credentials, hashes, keys, configs
- Identify lateral targets and pivot points

### 2.7 Privilege escalation
- Automated enum (LinPEAS / WinPEAS)
- Manual deep dive
- Kernel exploits as last resort

### 2.8 Reporting
- Executive summary (impact, in business terms)
- Technical detail (steps to reproduce, evidence)
- Risk rating (CVSS or similar)
- Remediation guidance

---

## 3. Time Management on a Box

A useful soft framework for HTB-style targets:

| Elapsed | Should have |
|---------|-------------|
| 0–15 min | Full port scan, service versions, web fingerprint |
| 15–45 min | Directory/vhost enum, identified candidate vuln |
| 45–90 min | Foothold (initial shell or web RCE) |
| 90–150 min | User flag |
| 150–240 min | Root flag |

If you're 60+ minutes in with no foothold, **stop pivoting on hunches and step back**:
1. Re-list every clue you've gathered.
2. Search for the *least* familiar one.
3. Re-run scans with bigger wordlists / different angles (UDP, vhost, larger dir list).
4. Check if a "boring" service you skipped (SNMP, NFS, RPC) has more to give.

If you're 4+ hours total without root, write notes, walk away, sleep, return. The brain reorganizes overnight; the box that was impossible at midnight is often obvious at 8 AM.

---

## 4. Note-Taking Discipline

A notes file is not optional. Use the same structure for every box:

```markdown
# <Box Name> — <IP>

## Summary
- OS: ?
- Difficulty: ?
- Initial vector: ?
- Privesc: ?
- Time: ?

## Recon
### Nmap
[paste]
### Hostnames
[/etc/hosts entries]

## Web (port 80)
### Tech
- [whatweb output]
### Directories
- /admin (302)
- /uploads (403)
### Findings
- [interesting JS, comments, etc.]

## Foothold
### Vector
[describe]
### Payload
[the working payload, escaped]
### Catcher
[command used]

## User
- User: ?
- Path: ?
- Flag: <hash>

## Lateral / Privesc
### Loot
- [creds, hashes, keys]
### Vector
[describe]

## Root
- Flag: <hash>

## Lessons
- [what would I do faster next time?]
```

The "Lessons" section is the highest-value part. Read it before starting the next box.

---

## 5. Tooling Discipline

- **One tool per job, fully understood.** It's better to know `ffuf` deeply than to have ten fuzzers you barely use.
- **Read the man page.** Most operators never read `man nmap`. Doing so is a force-multiplier.
- **Build a personal cheatsheet.** Steal from this skill set, but keep your own — the muscle memory of *typing* the commands matters.
- **Update tools weekly.** `searchsploit -u`, `apt update && apt upgrade`, repo updates for `nuclei`, `impacket`, etc.
- **Know what tools exist for what.** When you hit Redis, you should already know `redis-cli`. When you hit ADCS, you should already know `certipy`. The "I'll Google it" tax compounds.

---

## 6. Mental Models

### 6.1 Trust boundaries
Every system has *trust boundaries* — places where untrusted input crosses into trusted code. Vulns live at the boundaries:
- Network → web app (port 80)
- HTTP request → application (parameters, cookies, headers)
- Application → database (queries)
- Application → OS (system calls, file ops)
- Application → external API (SSRF, deserialization)
- User → admin (auth, IDOR)

When stuck, ask: *what trust boundary haven't I attacked yet?*

### 6.2 The vulnerability is where the code is most complex
Bugs cluster around:
- Auth, session, password reset (state machines are hard)
- File upload / parsing (formats are hard)
- Templating (mixing data and code is hard)
- Deserialization (reconstructing objects is hard)
- Multi-step transactions (race conditions are hard)
- Edge cases of input validation (filters are hard)

### 6.3 Defense in depth fails one layer at a time
Even hardened boxes have *one* layer that's weak. Find it:
- Strong auth, weak session token
- Strong session, weak CSRF
- Strong CSRF, weak rate limit
- Strong rate limit, weak password reset

### 6.4 Real-world maps to CTF
Almost every CTF technique has a real-world analog. Treat each box as practice for the corresponding real-world skill, not a puzzle for its own sake.

---

## 7. Anti-Patterns to Avoid

- **Tool roulette** — running every scanner without reading the output
- **PoC mashing** — pasting random GitHub PoCs hoping one works
- **Premature privesc** — looking for SUIDs before you've explored the box
- **Walkthrough peeking** — reading the next step instead of solving
- **Single-thread thinking** — fixating on one path when there are five
- **Skipping the "boring" port** — that DNS server has a zone transfer
- **Trusting the easy answer** — the obvious vuln is sometimes a rabbit hole; check, then move on if it's a dead end
- **Not stabilizing the shell** — wasting 30 minutes fighting a broken TTY when 30 seconds fixes it

---

## 8. Building the Habit

Daily practice trumps weekend marathons. 60 focused minutes per day for a year produces an elite operator. Suggested rotation:

- **Mon/Wed/Fri:** active boxes (HTB / TryHackMe / PG)
- **Tue/Thu:** PortSwigger Academy labs (web theory)
- **Sat:** read 2–3 disclosed bug bounty reports, write a 1-paragraph reflection
- **Sun:** rest, or re-do an old box for speed (target: 50% of original time)

Keep a public-or-private blog. Writing about what you learned cements it. The act of explaining a technique exposes the gaps in your understanding.

---

## 9. Ethics & Discipline

- Never test against systems you don't own without explicit written permission.
- Never reuse client tooling artifacts (creds, screenshots, payloads) across engagements.
- Never publish PoCs or details that could harm clients before remediation.
- Disclose responsibly to vendors when you find real vulns in real software.
- Respect the platforms (HTB, THM, PG) — don't share active-machine writeups, don't auto-pwn for points.

The community is small. Reputation compounds, in both directions.

---

*Methodology > tools > tricks. Re-enumerate when stuck. The flag is always one detail away.*
