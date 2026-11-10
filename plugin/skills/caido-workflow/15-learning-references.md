# 15 — Learning Path & References

How to build skill systematically over years, and where to look when this skill set isn't deep enough.

---

## 1. Skill-Building Cadence

A sustainable weekly rhythm beats burst studying. Pick a level and stay with it.

### 1.1 Beginner (first 6 months)
- 1–2 hours/day, 5 days/week
- Focus: foundations + Easy boxes
- Goal: comfortable solo on Easy HTB, half of Medium

Weekly:
- Mon/Wed/Fri: HTB Easy / TryHackMe rooms
- Tue/Thu: PortSwigger Academy labs (foundations: SQLi, XSS, auth)
- Sat: read 2–3 disclosed bug bounty reports
- Sun: rest or repeat a previous box for speed

### 1.2 Intermediate (6–18 months)
- Same cadence, harder targets
- Focus: Medium boxes consistently, AD basics, custom payloads
- Goal: OSCP-level proficiency, comfortable with most Medium + some Hard

Weekly:
- Mon/Wed: HTB Medium
- Tue/Thu: PortSwigger Practitioner / Apprentice labs + AD-specific HTB Academy modules
- Fri: Proving Grounds Practice box
- Sat: write up a recent box (private blog ok)
- Sun: rest

### 1.3 Advanced (18 months+)
- 5–10 hours/week, narrowing focus
- Focus: depth in 1–2 specialties (web, AD, cloud, mobile)
- Goal: novel finds, real engagements

- Pick a specialty: web app sec, Active Directory, cloud (AWS/GCP/Azure), mobile, hardware/IoT
- Read source code of real apps in the specialty
- Contribute writeups, tools, CTF challenges

---

## 2. Curated Box Lists

### 2.1 OSCP-prep (TJnull's list)
The community-curated list of HTB and Proving Grounds boxes that match OSCP difficulty. Updated yearly.
https://docs.google.com/spreadsheets/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/

Rough sequence for the HTB-only column:
- Lame, Legacy, Devel (intro)
- Beep, Optimum, Bashed, Nibbles
- Sense, Solidstate, Node, Valentine
- Granny, Grandpa, Cronos
- Forest (AD basics)
- Active (AD basics)
- Sauna, Resolute (AD intermediate)

### 2.2 Web-focused
- Bastard, Arctic, Tally (older IIS)
- Bart, Jarvis, Magic (PHP web)
- TheNotebook, Cache (modern Node/PHP)
- Networked, Photobomb (image / file upload)

### 2.3 Active Directory–focused
- Forest, Active, Sauna (entry-level)
- Resolute, Cascade (intermediate)
- Monteverde, Mantis, Sizzle (advanced)
- Multimaster, Reel (full-chain)

### 2.4 Privesc deep-dive
- HTB modules in Academy: Linux Privesc, Windows Privesc
- TryHackMe rooms: "Linux PrivEsc", "Windows PrivEsc", "Linux PrivEsc Arena"

---

## 3. Foundational Knowledge (read these once)

### 3.1 Books
- **The Web Application Hacker's Handbook** (Stuttard, Pinto) — still the canonical web sec text.
- **Real-World Bug Hunting** (Yaworski) — modern web bugs explained.
- **The Hacker Playbook 3** (Kim) — practical pentesting workflow.
- **Red Team Field Manual** (RTFM) — concise command reference.
- **Blue Team Field Manual** (BTFM) — defender perspective.
- **Linux Basics for Hackers** (OccupyTheWeb) — for true beginners.
- **Practical Malware Analysis** (Sikorski, Honig) — for malware/RE pivot.
- **The Tangled Web** (Zalewski) — deep web protocol weirdness.
- **A Guide to Kernel Exploitation** (Perla, Oldani) — kernel-level exploitation.

### 3.2 Long-form online
- **Hacker101** (HackerOne) — free web app sec course
- **PortSwigger Web Security Academy** — gold standard, free
- **OWASP Testing Guide** — methodology bible
- **OWASP Top 10** — yearly snapshot of the most common bugs
- **OWASP Cheat Sheet Series** — defender-side, useful for offense too
- **HackTricks** — exhaustive offensive cheatsheets
- **PayloadsAllTheThings** — payload encyclopedia
- **The Hacker Recipes** (https://www.thehacker.recipes/) — clean, modern AD focus
- **Active Directory Security** (https://adsecurity.org/) — Sean Metcalf's blog
- **HarmJ0y** (https://blog.harmj0y.net/) — AD attack research
- **SpecterOps blog** (https://posts.specterops.io/) — AD, ADCS, BloodHound

### 3.3 Tool docs (read fully, not skim)
- `man nmap`
- Burp Suite documentation
- Metasploit Unleashed (free course)
- impacket README + per-tool source
- BloodHound documentation

---

## 4. Platforms

### 4.1 Hands-on
- **Hack The Box** (https://www.hackthebox.com/) — primary
- **TryHackMe** (https://tryhackme.com/) — guided rooms, gentler entry
- **Proving Grounds** (https://www.offsec.com/labs/) — OSCP-styled lab
- **PortSwigger Academy** (https://portswigger.net/web-security) — web-only, free
- **Vulnhub** (https://vulnhub.com/) — downloadable VMs
- **PentesterLab** (https://pentesterlab.com/) — small focused exercises
- **Root-Me** (https://www.root-me.org/) — wide variety, multilingual
- **HackingHub** — modern web challenges

### 4.2 CTF
- **CTFtime** (https://ctftime.org/) — calendar of all CTFs
- **picoCTF** — beginner-friendly, US-based
- **Cyberdefenders** — DFIR-leaning
- **HTB Battlegrounds** — PvP

### 4.3 Bug bounty (after CTF feels stale)
- HackerOne (https://hackerone.com/)
- Bugcrowd (https://bugcrowd.com/)
- Intigriti (https://intigriti.com/)
- YesWeHack (https://yeswehack.com/)
- Open Bug Bounty (https://openbugbounty.org/)

---

## 5. People to Follow (Video / Stream)

- **IppSec** (https://www.youtube.com/@ippsec) — retired-HTB walkthroughs, methodology gold
- **John Hammond** — broad pentesting & malware
- **LiveOverflow** — binary exploitation deep-dives
- **HackerSploit** — beginner-to-intermediate
- **0xdf** (blog: https://0xdf.gitlab.io/) — clean HTB writeups
- **The Cyber Mentor (TCM)** — practical pentesting courses
- **Conda** (Twitch) — live CTF
- **Stök** — bug bounty interviews & reflection
- **NahamSec** — bug bounty practical content

---

## 6. Certifications (When Useful)

| Cert | When to take it |
|------|-----------------|
| **eJPT** (eLearnSecurity Junior PT) | Entry-level practical, self-paced |
| **OSCP** | Industry standard for entry pentest roles |
| **OSEP** | Offensive Security Experienced Pen Tester (post-OSCP, more evasion / AD) |
| **OSWE** | Web app deep-dive, source-review heavy |
| **OSED** | Exploit development |
| **CRTO** (RastaMouse) | Modern red team / C2 / AD |
| **CRTP / CRTE** (Altered Security) | AD-specific, hands-on |
| **PNPT** (TCM) | Practical, AD-heavy, less expensive |
| **OSCE3** | Combination of OSEP/OSWE/OSED |
| **GPEN, GXPN, GWAPT** | SANS — expensive but well-respected |
| **CEH** | Skip unless required by HR |

Start: eJPT → OSCP → specialty (OSEP for red team, OSWE for web).

---

## 7. Communities

- **HTB Forums** (https://forum.hackthebox.com/) — official, hint-only
- **Discord servers** — most platforms have one
- **r/hacking, r/netsec, r/AskNetsec** — moderated, broad
- **Local DEFCON groups** — meet humans
- **Twitter / Mastodon /infosec.exchange** — current research
- **infosec.exchange** Mastodon instance — many security pros migrated here

---

## 8. Reading Discipline

### 8.1 Disclosed reports
HackerOne and Bugcrowd publish disclosed reports. Read 2–3 per week.
- https://hackerone.com/hacktivity?type=public
- https://bugcrowd.com/disclosures
For each: write a one-line lesson. After 100 reports, you'll have intuition for what looks vulnerable.

### 8.2 Vendor advisories
- Microsoft Patch Tuesday summaries
- Cisco / Fortinet / Palo Alto / Citrix advisories
- Atlassian (Confluence, Jira)
- Apple / Android security bulletins

### 8.3 Project Zero / GitHub Security Lab
- https://googleprojectzero.blogspot.com/
- https://securitylab.github.com/
Top-tier research; you won't always understand everything, but exposure helps.

### 8.4 Conference talks
- DEF CON, Black Hat (USA + EU + Asia)
- BSides (regional)
- OffensiveCon
- x33fcon, Insomnihack, NorthSec
YouTube channels of each have free talks.

---

## 9. Lab Exercises Beyond CTF

- **Build a vulnerable lab**: `vulnnoob`, `dvwa`, `juice-shop`, `webgoat`, `bwapp`. Pop them.
- **Build a homelab AD**: 1 DC + 2 workstations on VirtualBox. Misconfigure deliberately. Practice attacks against it.
- **Source-dive a CMS quarterly**: WordPress plugin, Joomla extension, custom PHP app from GitHub. Read for vulns.
- **Write a small web app then break it**: builds intuition no checklist gives.
- **Reverse a small CTF binary**: even one a quarter teaches debugging.

---

## 10. Habits That Compound

### 10.1 Note-taking
A persistent notes system pays back forever. Pick one:
- **Obsidian** — markdown, plugin-rich
- **Notion** — cloud, collaborative
- **CherryTree** — local, hierarchical
- **Joplin** — cross-platform markdown
- **vimwiki / org-mode** — text-only, terminal

Structure:
```
notes/
  ├── boxes/<name>.md
  ├── techniques/<class>.md
  ├── tools/<tool>.md
  ├── creds/   (encrypted)
  └── _index.md
```

### 10.2 One-line lessons
After every box: one sentence in `_lessons.md`. Re-read monthly.

### 10.3 Time tracking
Per box: foothold time, root time, total time. Trends matter more than averages.

### 10.4 Spaced re-do
Re-do a box you struggled with 30 days later. Target half the time.

---

## 11. Specialty Tracks

### 11.1 Web Application Sec
- PortSwigger Academy → all labs
- "The Web Application Hacker's Handbook"
- HackerOne disclosed reports
- Bug bounty practice
- Source-dive top-100 GitHub PHP/Node apps
- Tools: Burp Pro, ZAP, sqlmap, ffuf, custom scripts

### 11.2 Active Directory / Red Team
- HTB Academy AD path
- Altered Security CRTP / CRTE / CRTM
- Specter Ops "Adversary Tactics: Detection" (defender lens, useful)
- Tools: BloodHound, Rubeus, Mimikatz, Impacket, certipy, NetExec
- Read: ADSecurity blog, HarmJ0y blog, "An ACE Up the Sleeve"

### 11.3 Cloud Sec
- AWS / GCP / Azure security docs (start with the official "best practices")
- "Hands-On AWS Pentesting"
- flAWS / flAWS2 challenges
- Tools: Pacu, ScoutSuite, Prowler, kube-hunter, CloudGoat

### 11.4 Mobile
- "The Mobile Application Hacker's Handbook"
- OWASP MASTG (Mobile Security Testing Guide)
- Tools: Frida, Objection, MobSF, jadx, ghidra
- Practice: Damn Vulnerable iOS App, InsecureBank

### 11.5 Binary Exploitation / Reverse Engineering
- Nightmare CTF (https://guyinatuxedo.github.io/)
- pwn.college (https://pwn.college/)
- "Hacking: The Art of Exploitation"
- Tools: pwntools, gdb-peda/pwndbg, ghidra, radare2

### 11.6 Hardware / IoT / Embedded
- Read: "The Car Hacker's Handbook", "Practical IoT Hacking"
- Tools: bus pirate, JTAGulator, logic analyzer
- Platforms: Hack The Box has occasional hardware boxes; Pwn2Own embedded

---

## 12. Quick Reference Hub (just bookmarks)

```
GTFOBins:                 https://gtfobins.github.io/
LOLBAS:                   https://lolbas-project.github.io/
HackTricks:               https://book.hacktricks.xyz/
PayloadsAllTheThings:     https://github.com/swisskyrepo/PayloadsAllTheThings
The Hacker Recipes:       https://www.thehacker.recipes/
RevShells generator:      https://revshells.com/
PortSwigger Academy:      https://portswigger.net/web-security
HTB Academy:              https://academy.hackthebox.com/
IppSec videos search:     https://ippsec.rocks/
0xdf writeups:            https://0xdf.gitlab.io/
ADSecurity:               https://adsecurity.org/
HarmJ0y:                  https://blog.harmj0y.net/
SpecterOps:               https://posts.specterops.io/
CTFtime:                  https://ctftime.org/
HackerOne hacktivity:     https://hackerone.com/hacktivity?type=public
Project Zero:             https://googleprojectzero.blogspot.com/
SecLists repo:            https://github.com/danielmiessler/SecLists
Impacket:                 https://github.com/fortra/impacket
NetExec (CME successor):  https://github.com/Pennyw0rth/NetExec
Kerbrute:                 https://github.com/ropnop/kerbrute
PEASS-ng:                 https://github.com/peass-ng/PEASS-ng
```

---

## 13. The Long Game

Skill compounds, but only if you put in the reps. Two truths:

1. **Quantity, then quality.** First year = volume. 50 boxes, 100 PortSwigger labs. Don't be a perfectionist about your first 50 writeups; just do them.
2. **One technique mastered beats ten skimmed.** Pick a specialty by year two. Go deeper than anyone you know on at least one topic. That's how you get hired and how you find novel bugs.

The community has many career paths off this skill set:
- Pentester (consulting)
- Red team (in-house)
- Bug bounty hunter (independent)
- Application security engineer (defender side, often more lucrative)
- Detection engineer
- Vulnerability research
- Exploit developer
- Security architect

Some of the best pentesters I've seen were ex-developers, ex-sysadmins, and ex-defenders. Cross-discipline experience is an unfair advantage.

---

## 14. Minimum Bookshelf (3 books)

If you can only read three:
1. **The Web Application Hacker's Handbook** (web)
2. **The Hacker Playbook 3** (workflow)
3. **Linux Basics for Hackers** (foundations) — replaceable with `man bash` if you're already comfortable.

Add one specialty book per year after.

---

## 15. Last Word

Pentesting is not a job that rewards memorization or talent alone. It rewards methodical curiosity over years. The operators who become elite did the boring reps — recon, enumeration, note-taking — every single time, even when the box was easy.

Build the habit. The skill follows.

---

*The flag is always one detail away. The career is many years of finding those details.*
