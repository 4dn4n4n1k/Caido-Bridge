# 13 — OPSEC & Operational Hygiene

The discipline of leaving no trace where you shouldn't, and a clean record where you should. Even in CTF/lab, build the habits — they're load-bearing in real engagements.

---

## 1. Why OPSEC Matters in CTF

You're practicing for engagements where:
- Detection is failure (red team)
- Forensic evidence is the deliverable (incident response cross-training)
- Compliance dictates audit trails (real pentests)

The CTF doesn't care if you `echo password >> /etc/passwd`. The next engagement will. Build the muscle memory now.

---

## 2. Authorization & Scope

The non-negotiable foundation:
- **Statement of Work** (SoW) signed before any technical work.
- **In-scope assets** explicitly listed (IPs, domains, applications, time windows).
- **Out-of-scope** explicitly listed.
- **Allowed tactics** — DoS? Social engineering? Phishing? Default-deny.
- **Emergency contacts** — who to call when something breaks (because something will).
- **Reporting cadence** — daily updates, immediate critical disclosures.

For HTB / lab: the platform's terms of service is your authorization. Stay on the platform's network.

---

## 3. Targeting Discipline

- Never test against a system you don't own without written permission.
- Never scan IPs adjacent to scope without permission ("oops" doesn't fix a CFAA charge).
- Never reuse client credentials, payloads, or screenshots across engagements.
- Never publish exploit details for client-discovered bugs before remediation.

---

## 4. Network Hygiene

### 4.1 Source IP discipline
- Use the engagement-specific VPN / jump host, not your home IP.
- For HTB, use the HTB VPN — the 10.10.x.x address is yours and HTB-attributable.
- Don't run scans from a machine that's also browsing personal accounts.

### 4.2 Egress traffic
- All testing tools should be configured to use only the engagement's network/VPN.
- Verify with `ip route` before scanning.

### 4.3 Disconnect when done
```bash
sudo killall openvpn
ip a show tun0 2>/dev/null && echo "still connected!" || echo "VPN down"
```

---

## 5. Logging & Evidence

### 5.1 Self-logging (your record)
Per-engagement directory with:
- Timestamped command logs
- All scan outputs
- Screenshots of every finding
- Captured packets (PCAP) for relevant interactions

```bash
# Capture everything in your shell to a log:
script -aq engagement.log
# or via tmux:
tmux pipe-pane -o 'cat >> ~/htb/<box>/term.log'
```

### 5.2 Tool logs
- Burp project file per engagement, saved at end of session.
- Metasploit DB exports (`db_export`).
- BloodHound zips per AD environment.

### 5.3 Evidence integrity
For real engagements:
- Hash artifacts (`sha256sum`) and record hashes.
- Optionally sign reports with your GPG key.
- Use a write-once medium for forensic-grade work.

---

## 6. On-Target Hygiene

### 6.1 Stage in tmp / writable dirs you'll own
```bash
mkdir -p /tmp/.eng-<id>
cd /tmp/.eng-<id>
```
Pick names that don't collide with system paths and that you'll remember to clean.

### 6.2 Avoid widely-readable configs
Don't drop binaries to `/var/www/html` if a user might browse them. Don't drop to `/tmp` with mode 644 if you don't want them readable.

### 6.3 Don't break stuff
- Avoid restarting services blindly. They might be production.
- Avoid tools that lock accounts (`hydra` against AD without rate awareness).
- Avoid memory-corruption exploits without testing in a lab first — they crash apps.

### 6.4 Cleanup checklist
After every engagement step where you persisted anything:
```
[ ] Removed uploaded binaries
[ ] Removed user accounts you created
[ ] Removed SSH keys you added
[ ] Removed cron / scheduled tasks
[ ] Removed services you created
[ ] Restored files you replaced
[ ] Removed reverse-shell processes
[ ] Removed temporary directories
[ ] Killed any background tunnels (chisel, ligolo)
```

Sample Linux cleanup script (lab template):
```bash
#!/usr/bin/env bash
# remove our cron entries
crontab -l 2>/dev/null | grep -v 'pwn-eng' | crontab -
# kill our shells
pkill -f '/tmp/.eng-' || true
# remove our staging dir
rm -rf /tmp/.eng-* 2>/dev/null
# remove our user (if any)
userdel -r pwntest 2>/dev/null
# remove our SSH key entries (assumes a known marker)
sed -i '/PWN-ENG-MARKER/d' /home/*/.ssh/authorized_keys 2>/dev/null
sed -i '/PWN-ENG-MARKER/d' /root/.ssh/authorized_keys 2>/dev/null
echo "[+] Cleanup complete"
```

---

## 7. Credentials Handling

- Never paste real client creds into a chat / Slack / Discord / public note.
- Store engagement creds in a per-engagement KeePass DB, not in plaintext notes.
- Never commit creds to a shared git repo, even a private one.
- Treat captured hashes and tickets as classified — encrypt at rest, transmit only over secure channels.
- Real-world: dispose of all client-derived material at engagement end (per contract).

---

## 8. Tool Choice & Backdoor Awareness

- Only run PoCs you've **read** or that come from trusted sources.
- Beware GitHub PoCs with obfuscated payloads, base64 blobs, or "binary only" releases.
- Snapshot lab VMs before running unverified code.
- Even well-known tools have version-specific bugs — pin tool versions.

Trusted sources:
- impacket (Fortra/CoreLabs)
- mimikatz (Benjamin Delpy / official repo)
- BloodHound (SpecterOps)
- nmap, hashcat, John, Burp Suite (vendors)
- HackTricks / PayloadsAllTheThings — read references, not blind paste

---

## 9. Avoiding Detection (Red Team Context)

For training purposes — not a recipe for evasion against unauthorized targets.

### 9.1 Indicator hygiene
- Avoid known-bad filenames (`mimikatz.exe`, `nc.exe`).
- Avoid scripted indicators baked into PoCs (canary URLs, hardcoded IPs).
- Recompile or re-pack tools when in scope.
- Use in-memory loaders rather than disk drops.

### 9.2 Timing
- Don't burst-scan at 3 AM if the SOC monitors that hour heavily.
- Stretch operations across business hours when fitting business behaviour.
- Pause between attempts — most modern detections fire on rate, not single events.

### 9.3 Process / service indicators
- Avoid spawning suspicious parent-child trees (e.g., `winword.exe → cmd.exe`).
- Live off the land (LOLBAS / GTFOBins) when possible.
- Clean up logs you generated where in scope (carefully).

### 9.4 Network indicators
- Beacon over 443 with TLS, not 4444 plaintext.
- Realistic User-Agent strings in any HTTP-based C2.
- Avoid known C2 infrastructure (cheap VPS that's already on threat lists).

---

## 10. Reporting Discipline

A real pentest is judged by the report, not the shells.

### 10.1 Structure
- Executive summary (business impact, in business language)
- Engagement scope & approach
- Findings (each with: title, severity, description, reproduction, evidence, remediation, references)
- Strategic recommendations
- Appendices (raw data, tools used)

### 10.2 Severity
- CVSS 3.1 base + environmental.
- Don't inflate. Calibrate against industry norms.
- Always express in **risk** terms (impact × likelihood), not just "RCE!"

### 10.3 Reproducibility
Every finding needs steps that someone else can re-run:
- Exact URL / endpoint / command
- Exact payload (escaped)
- Expected response / observed behavior
- Screenshots with timestamps

---

## 11. Personal OPSEC

- Don't post screenshots of active boxes or client data.
- Don't tweet about active engagements.
- Don't show off creds even after rotation — they reveal client identity.
- Use a separate handle for CTF / public security work vs. real-name professional work.

---

## 12. Legal Awareness

A short, non-legal-advice list. Educate yourself on:
- **CFAA** (US) — Computer Fraud and Abuse Act. Authorization is everything.
- **GDPR** (EU) — handling personal data found during engagement.
- **CCPA / state laws** (US) — same.
- **Local equivalents** — many countries have their own.
- **DMCA / EUCD** — exploit research can collide with anti-circumvention rules.

When in doubt, don't, and check with engagement counsel.

---

## 13. Anti-Patterns

- "It's just a lab" → habits formed in lab transfer to real work.
- Running an exploit you can't read → backdoors itself, you find out later.
- Logging into client systems with personal accounts.
- Storing client data in personal cloud (Dropbox, iCloud).
- Mixing CTF and engagement workspaces.
- Letting a tunnel / persistence stay up after the engagement.

---

## 14. The Good-Habits Checklist

Run through this after every box / engagement step:
```
[ ] Did I clean up everything I planted?
[ ] Did I save evidence for findings I care about?
[ ] Did I log my own activity?
[ ] Did I disconnect my VPN / tunnels?
[ ] Did I rotate any creds I exposed?
[ ] Did I update notes and a one-line lesson?
```

---

*Discipline doesn't slow you down — it's the only way to scale your skill.*
