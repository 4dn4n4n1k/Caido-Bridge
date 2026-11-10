# 01 — Lab Environment Setup

A correctly built lab is the difference between a smooth engagement and 90% of your time fighting your own tooling.

---

## 1. Choosing the Base OS

| Option | When to use |
|--------|-------------|
| **Kali Linux** | Default. Largest tool set, best community support, what walkthroughs assume. |
| **Parrot OS Security** | Lighter, AnonSurf included, prettier desktop. Equivalent for most work. |
| **BlackArch** | Massive tool set on top of Arch. Use only if you know Arch. |
| **Custom Debian/Ubuntu** | Cleaner, smaller. Build your own stack. Best for senior operators. |
| **REMnux** | Malware analysis specifically. Not a daily driver. |

**Recommendation:** Kali for the first year. Switch later if you have reasons.

### Install media
- Always grab from the official site, verify GPG signature.
- Kali: https://www.kali.org/get-kali/
- Parrot: https://parrotsec.org/download/

### Bare metal vs VM vs WSL2

| Form | Pros | Cons |
|------|------|------|
| Bare metal | Best perf, full hardware (Wi-Fi adapters) | Risk to host, harder to snapshot |
| VirtualBox / VMware VM | Snapshots, isolation, easy reset | Lower perf, USB passthrough quirks |
| WSL2 | Lightweight, share host files | No GUI by default, networking limits |

For HTB/CTF: **VM with snapshots**. Take a clean snapshot after every major customization.

---

## 2. Initial Hardening & Setup

```bash
# Update everything
sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

# Set timezone
sudo timedatectl set-timezone UTC

# Generate a fresh SSH key (for GitHub, etc.)
ssh-keygen -t ed25519 -C "lab"

# Increase open file limits (helps fuzzing)
echo "* soft nofile 65535" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65535" | sudo tee -a /etc/security/limits.conf

# Useful sysctls for high-volume scanning
sudo sysctl -w net.ipv4.ip_local_port_range="10000 65535"
sudo sysctl -w net.core.somaxconn=4096
```

---

## 3. Tooling Inventory

### 3.1 Recon
```bash
sudo apt install -y nmap masscan rustscan whatweb dnsutils
# Subdomain enum (modern)
sudo apt install -y amass
# Optional:
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

### 3.2 Web
```bash
sudo apt install -y gobuster ffuf feroxbuster wfuzz nikto wpscan dirb
sudo apt install -y burpsuite zaproxy sqlmap
# Headless browsers
sudo apt install -y chromium
```

### 3.3 Shells / Exploit Frameworks
```bash
sudo apt install -y netcat-traditional ncat socat metasploit-framework
sudo apt install -y exploitdb seclists
pipx install pwncat-cs        # superior reverse shell handler
```

### 3.4 AD / Windows
```bash
sudo apt install -y crackmapexec impacket-scripts evil-winrm bloodhound
sudo apt install -y enum4linux smbclient smbmap responder ldap-utils
sudo apt install -y kerbrute
pipx install bloodhound        # Python collector
pipx install certipy-ad        # ADCS abuse
```

### 3.5 Cracking
```bash
sudo apt install -y hashcat john hydra medusa hashid
# GPU drivers if you have a GPU — hashcat benefits massively
```

### 3.6 Misc utilities
```bash
sudo apt install -y proxychains4 chisel tmux jq xxd ripgrep fd-find
pipx install updog              # quick HTTP file server, nicer than python -m http.server
pipx install ligolo-ng-proxy    # only the client side ships in repos in some distros
```

### 3.7 Editors
```bash
sudo apt install -y neovim
# or:
curl -fsSL https://code.visualstudio.com/sha/download?build=stable -o vscode.deb && sudo dpkg -i vscode.deb
```

---

## 4. Wordlists

`SecLists` is the canonical collection. Confirm:
```bash
ls /usr/share/seclists/
```
Top-of-mind dirs:
- `Discovery/Web-Content/` — directory and file fuzzing (`raft-*`, `directory-list-*`)
- `Passwords/` — `rockyou.txt`, `Common-Credentials/`, `Leaked-Databases/`
- `Usernames/` — `xato-net-10-million-usernames.txt`, names by gender
- `Fuzzing/` — payloads for SQLi, XSS, LFI, command-injection
- `SNMP/`, `Miscellaneous/`, `IOCs/`

Decompress rockyou:
```bash
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz
ls -lh /usr/share/wordlists/rockyou.txt
```

Custom wordlist generation:
```bash
# from a website
cewl https://example.com -w cewl.txt -d 3 -m 5
# from existing list with mutations
hashcat --stdout -r /usr/share/hashcat/rules/best64.rule rockyou.txt | head
# username permutations from real names
python3 username-anarchy.py -i 'first last'   # github.com/urbanadventurer/username-anarchy
```

---

## 5. Workspace Layout

A consistent layout pays back forever.

```
~/htb/
  ├── _templates/              # blank notes.md, scan scripts
  ├── _tools/                  # PEAS, kerbrute, chisel, nc.exe, etc.
  ├── _wordlists/              # custom lists
  └── <box-name>/
        ├── nmap/              # scan output
        ├── web/               # downloaded pages, fuzz outputs
        ├── loot/              # creds, hashes, keys, screenshots
        ├── exploits/          # tweaked PoCs
        ├── shells/            # generated payloads
        └── notes.md
```

Templated init script (`~/htb/_templates/new-box.sh`):
```bash
#!/usr/bin/env bash
set -e
NAME="$1"; IP="$2"
[ -z "$NAME" ] || [ -z "$IP" ] && echo "usage: $0 <name> <ip>" && exit 1
DIR="$HOME/htb/$NAME"
mkdir -p "$DIR"/{nmap,web,loot,exploits,shells}
cat > "$DIR/notes.md" <<EOF
# $NAME — $IP
## Summary
## Recon
## Web
## Foothold
## User
## Privesc
## Root
## Lessons
EOF
echo "$IP $NAME.htb" | sudo tee -a /etc/hosts
echo "Workspace: $DIR"
```

Make it executable: `chmod +x new-box.sh`. Run as `new-box.sh nibbles 10.10.10.75`.

---

## 6. Pre-staged Tools Directory

Keep a `~/htb/_tools/` directory with frequently-uploaded binaries ready to serve:

```
_tools/
  linpeas.sh
  winPEASx64.exe
  winPEASany.exe
  pspy64
  pspy32
  nc.exe
  socat
  socat.aarch64    # for ARM targets
  chisel
  chisel.exe
  ligolo-proxy
  ligolo-agent.exe
  PowerView.ps1
  PowerUp.ps1
  Sherlock.ps1
  Watson.exe
  Seatbelt.exe
  SharpHound.exe
  SharpHound.ps1
  Rubeus.exe
  Mimikatz.exe       # AV will scream; prefer DInvoke if needed
  GodPotato.exe
  PrintSpoofer.exe
  JuicyPotatoNG.exe
```

Refresh quarterly:
```bash
# LinPEAS / WinPEAS
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/winPEASx64.exe
# pspy
wget https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64
# chisel
wget https://github.com/jpillora/chisel/releases/latest/download/chisel_*_linux_amd64.gz
```

A one-liner to serve everything:
```bash
cd ~/htb/_tools && python3 -m http.server 8000
```

---

## 7. VPN Configuration

### Connect to HTB
```bash
sudo openvpn /path/to/machines_us-4.ovpn
# Run in background or in a dedicated tmux pane.
# Confirm:
ip addr show tun0
ping -c 2 10.129.0.1
```

### Common VPN issues
- "TLS handshake failed" — try a different region's pack.
- "Tun/tap device not found" — `sudo modprobe tun`.
- Slow / dropped packets — switch UDP↔TCP variant, try a different region.

### Start-on-boot (optional, lab only)
Use systemd:
```ini
# /etc/systemd/system/htb-vpn.service
[Unit]
Description=HTB VPN
After=network.target
[Service]
Type=simple
ExecStart=/usr/sbin/openvpn --config /etc/openvpn/htb.ovpn
Restart=on-failure
[Install]
WantedBy=multi-user.target
```
`sudo systemctl enable --now htb-vpn`.

---

## 8. Browser Setup

A dedicated browser for testing avoids cross-contaminating cookies and history.

**Firefox developer edition** profile with:
- **FoxyProxy** — flip between Burp / no proxy
- **Wappalyzer** — tech fingerprinting
- **Cookie Editor** — quick cookie tampering
- **HackTools** extension — payloads & cheatsheets in-page
- **User-Agent Switcher**
- Disable HTTPS-only mode for lab (lets Burp self-signed certs through cleanly)

Burp CA cert:
- Burp → Proxy → Options → Import / Export CA Certificate → DER
- Firefox → about:preferences#privacy → Certificates → Import

---

## 9. Snapshot Strategy (VMs)

- **Snapshot 1: Clean install** — OS done, updates done, no tools
- **Snapshot 2: Tools installed** — full default kit, no engagements
- **Snapshot 3: Per-engagement** — before each pentest / box, before each risky tool

Restore policy: weekly back to Snapshot 2 to avoid drift; per-engagement back to 3 when done.

---

## 10. Resource Hygiene

- **Disk**: keep wordlists on a separate volume if you can. SecLists + rockyou alone is ~3 GB.
- **RAM**: Burp wants 4–8 GB for big sites. Bloodhound + Neo4j want 4 GB.
- **CPU**: hashcat will saturate. Keep cracking on a separate machine when possible.
- **Network**: a wired connection beats Wi-Fi for fuzzing (lower jitter, fewer dropped packets).

---

## 11. Verifying the Lab

After setup, run a self-check:

```bash
# Tools present
for t in nmap ffuf gobuster sqlmap nikto wpscan hydra john hashcat \
         smbclient enum4linux crackmapexec evil-winrm impacket-secretsdump; do
  command -v "$t" >/dev/null && echo "[OK] $t" || echo "[MISS] $t"
done

# Wordlist sanity
[ -f /usr/share/wordlists/rockyou.txt ] && echo "[OK] rockyou" || echo "[MISS] rockyou"
[ -d /usr/share/seclists ] && echo "[OK] seclists" || echo "[MISS] seclists"

# VPN connectivity (after `sudo openvpn ...`)
ip addr show tun0 && echo "[OK] tun0 up"
```

---

## 12. Maintenance Routine

Weekly:
- `sudo apt update && sudo apt full-upgrade`
- `searchsploit -u`
- `nuclei -update-templates`
- Refresh tools in `~/htb/_tools/`
- Snapshot

Monthly:
- Review installed packages, prune what you didn't use
- Update GitHub-installed Go/Python tools
- Rotate any keys/tokens used in tools

Quarterly:
- Full re-snapshot from clean baseline (drift control)
- Review your notes folder, archive old boxes

---

*A 30-minute lab tune-up saves 30 hours over a year.*
