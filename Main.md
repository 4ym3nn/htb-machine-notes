# HTB machine ideas

Search this file by technique or machine name. Use the short idea first, then open the linked official PDF when you need the complete steps.

## 1- Markup

**Keywords:** `source-code review`, `XXE`, `SSH`, `private-key discovery`, `reverse shell`, `cron/scheduled-job abuse`, `writable script/file abuse`

**User idea:** source-code review → XXE → SSH → private-key discovery

**Root idea:** reverse shell → cron/scheduled-job abuse → writable script/file abuse

[Open hint sheet](./htb-writeups/Markup.md) · [Open official PDF](./htb-writeups/Markup.pdf)

## 2- Busqueda

**Keywords:** `command injection`, `reverse shell`, `Git history/config`, `configuration-file secrets`, `subdomain enumeration`, `SSH`, `source-code review`, `sudo rule abuse`, `Docker/container escape`, `database enumeration`

**User idea:** command injection → reverse shell

**Root idea:** Git history/config → configuration-file secrets → subdomain enumeration → SSH → source-code review → sudo rule abuse → Docker/container escape → database enumeration → reverse shell

[Open hint sheet](./htb-writeups/Busqueda.md) · [Open official PDF](./htb-writeups/Busqueda.pdf)

## 3- Sau

**Keywords:** `known-vulnerability research`, `reverse shell`, `service abuse`, `sudo rule abuse`, `CVE-2023-26604`

**User idea:** known-vulnerability research → reverse shell

**Root idea:** service abuse → sudo rule abuse → CVE-2023-26604

[Open hint sheet](./htb-writeups/Sau.md) · [Open official PDF](./htb-writeups/Sau.pdf)

## 4- Help

**Keywords:** `HelpDeskZ enumeration`, `SQL injection`, `file upload abuse`, `timestamp prediction`, `reverse shell`, `kernel version enumeration`, `kernel exploit`, `root shell`

**User idea:** HelpDeskZ enumeration → SQL injection → file upload abuse → timestamp prediction → reverse shell

**Root idea:** kernel version enumeration → kernel exploit → root shell

[Open hint sheet](./htb-writeups/Help.md) · [Open official PDF](./htb-writeups/Help.pdf)

## 5- Broker

**Keywords:** `CVE-2023-46606/CVE-2023-46604`, `web shell/RCE`, `reverse shell`, `unsafe deserialization`, `sudo rule abuse`, `configuration-file secrets`, `DLL/library hijacking`, `SSH`, `private-key discovery`

**User idea:** CVE-2023-46606/CVE-2023-46604 → web shell/RCE → reverse shell → unsafe deserialization

**Root idea:** sudo rule abuse → configuration-file secrets → DLL/library hijacking → SSH → private-key discovery

[Open hint sheet](./htb-writeups/Broker.md) · [Open official PDF](./htb-writeups/Broker.pdf)

## 6- Soccer

**Keywords:** `subdomain enumeration`, `group-membership abuse`, `source-code review`, `SQL injection`, `SSH`, `SUID abuse`, `configuration-file secrets`

**User idea:** subdomain enumeration → group-membership abuse → source-code review → SQL injection → SSH

**Root idea:** SUID abuse → configuration-file secrets

[Open hint sheet](./htb-writeups/Soccer.md) · [Open official PDF](./htb-writeups/Soccer.pdf)

## 7- Keeper

**Keywords:** `default credentials`, `ticket/helpdesk disclosure`, `credential reuse`, `SSH`, `KeePass database recovery`, `CVE-2023-32784`, `PuTTY private key conversion`, `SSH as root`

**User idea:** default credentials → ticket/helpdesk disclosure → credential reuse → SSH

**Root idea:** KeePass database recovery → CVE-2023-32784 → PuTTY private key conversion → SSH as root

[Open hint sheet](./htb-writeups/Keeper.md) · [Open official PDF](./htb-writeups/Keeper.pdf)

## 8- Networked

**Keywords:** `file upload abuse`, `cron/scheduled-job abuse`, `source-code review`, `command injection`, `SSH`, `directory/file discovery`, `reverse shell`, `PATH hijacking`, `sudo rule abuse`

**User idea:** file upload abuse → cron/scheduled-job abuse → source-code review → command injection → SSH → directory/file discovery → reverse shell → PATH hijacking

**Root idea:** sudo rule abuse → command injection

[Open hint sheet](./htb-writeups/Networked.md) · [Open official PDF](./htb-writeups/Networked.pdf)

## 9- BoardLight

**Keywords:** `group-membership abuse`, `reverse shell`, `database enumeration`, `SSH`, `SUID abuse`, `CVE-2022-37706`, `Git history/config`

**User idea:** group-membership abuse → reverse shell → database enumeration → SSH

**Root idea:** SUID abuse → CVE-2022-37706 → Git history/config

[Open hint sheet](./htb-writeups/BoardLight.md) · [Open official PDF](./htb-writeups/BoardLight.pdf)

## 10- CozyHosting

**Keywords:** `SSH`, `private-key discovery`, `command injection`, `web shell/RCE`, `reverse shell`, `configuration-file secrets`, `database enumeration`, `password cracking`, `sudo rule abuse`

**User idea:** SSH → private-key discovery → command injection → web shell/RCE → reverse shell → configuration-file secrets → database enumeration → password cracking

**Root idea:** SSH → sudo rule abuse

[Open hint sheet](./htb-writeups/CozyHosting.md) · [Open official PDF](./htb-writeups/CozyHosting.pdf)

## 11- Editorial

**Keywords:** `SSH`, `Git history/config`, `group-membership abuse`, `sudo rule abuse`, `CVE-2022-24439`, `reverse shell`

**User idea:** SSH → Git history/config → group-membership abuse → sudo rule abuse

**Root idea:** Git history/config → CVE-2022-24439 → reverse shell

[Open hint sheet](./htb-writeups/Editorial.md) · [Open official PDF](./htb-writeups/Editorial.pdf)

## 12- Pandora

**Keywords:** `SSH`, `SNMP enumeration`, `SQL injection`, `web shell/RCE`, `SUID abuse`, `virtual-host enumeration`, `configuration-file secrets`, `authentication bypass`, `file upload abuse`, `unsafe deserialization`, `PATH hijacking`, `reverse shell`

**User idea:** SSH → SNMP enumeration → SQL injection → web shell/RCE → SUID abuse → virtual-host enumeration → configuration-file secrets → authentication bypass → file upload abuse → unsafe deserialization

**Root idea:** SUID abuse → PATH hijacking → reverse shell

[Open hint sheet](./htb-writeups/Pandora.md) · [Open official PDF](./htb-writeups/Pandora.pdf)

## 13- LinkVortex

**Keywords:** `LFI/local file read`, `CVE-2023-40028`, `symlink/race-condition abuse`, `writable script/file abuse`, `configuration-file secrets`, `SSH`, `sudo rule abuse`, `private-key discovery`

**User idea:** LFI/local file read → CVE-2023-40028 → symlink/race-condition abuse → writable script/file abuse → configuration-file secrets → SSH

**Root idea:** sudo rule abuse → symlink/race-condition abuse → private-key discovery → SSH

[Open hint sheet](./htb-writeups/LinkVortex.md) · [Open official PDF](./htb-writeups/LinkVortex.pdf)

## 14- Dog

**Keywords:** `known-vulnerability research`, `web shell/RCE`, `reverse shell`, `password cracking`, `database enumeration`, `SSH`, `sudo rule abuse`, `SUID abuse`

**User idea:** known-vulnerability research → web shell/RCE → reverse shell → password cracking → database enumeration → SSH

**Root idea:** sudo rule abuse → password cracking → SUID abuse

[Open hint sheet](./htb-writeups/Dog.md) · [Open official PDF](./htb-writeups/Dog.pdf)

## 15- Usage

**Keywords:** `CVE-2023-24249`, `file upload abuse`, `web shell/RCE`, `reverse shell`, `database enumeration`, `SUID abuse`, `SSH`, `sudo rule abuse`, `symlink/race-condition abuse`, `LFI/local file read`, `private-key discovery`

**User idea:** CVE-2023-24249 → file upload abuse → web shell/RCE → reverse shell → database enumeration → SUID abuse → SSH

**Root idea:** sudo rule abuse → database enumeration → symlink/race-condition abuse → LFI/local file read → private-key discovery → SSH

[Open hint sheet](./htb-writeups/Usage.md) · [Open official PDF](./htb-writeups/Usage.pdf)

## 16- Titanic

**Keywords:** `Docker/container escape`, `Git history/config`, `configuration-file secrets`, `SSH`, `password cracking`, `database enumeration`, `CVE-2024-41817`, `SUID abuse`

**User idea:** Docker/container escape → Git history/config → configuration-file secrets → SSH → password cracking

**Root idea:** database enumeration → Docker/container escape → CVE-2024-41817 → SUID abuse

[Open hint sheet](./htb-writeups/Titanic.md) · [Open official PDF](./htb-writeups/Titanic.pdf)

## 17- Outbound

**Keywords:** `web shell/RCE`, `unsafe deserialization`, `Git history/config`, `CVE-2025-49113`, `configuration-file secrets`, `database enumeration`, `SSH`, `sudo rule abuse`, `source-code review`, `symlink/race-condition abuse`

**User idea:** web shell/RCE → unsafe deserialization → Git history/config → CVE-2025-49113 → configuration-file secrets → database enumeration → SSH

**Root idea:** sudo rule abuse → source-code review → symlink/race-condition abuse

[Open hint sheet](./htb-writeups/Outbound.md) · [Open official PDF](./htb-writeups/Outbound.pdf)

## 18- Editor

**Keywords:** `CVE-2025-24893`, `web shell/RCE`, `LXD/LXC abuse`, `database enumeration`, `reverse shell`, `SSH`, `SUID abuse`, `PATH hijacking`

**User idea:** CVE-2025-24893 → web shell/RCE → LXD/LXC abuse → database enumeration → reverse shell → SSH

**Root idea:** SUID abuse → PATH hijacking → reverse shell

[Open hint sheet](./htb-writeups/Editor.md) · [Open official PDF](./htb-writeups/Editor.pdf)

## 19- Expressway

**Keywords:** `network-device config secrets`, `password cracking`, `SSH`, `sudo rule abuse`, `CVE-2025-32462/CVE-2025-32463`

**User idea:** network-device config secrets → password cracking → SSH

**Root idea:** sudo rule abuse → CVE-2025-32462/CVE-2025-32463

[Open hint sheet](./htb-writeups/Expressway.md) · [Open official PDF](./htb-writeups/Expressway.pdf)

## 20- ServMon

**Keywords:** `known-vulnerability research`, `LFI/local file read`, `path traversal`, `FTP`, `SSH`, `password spraying`, `group-membership abuse`, `service abuse`, `ACL abuse`, `web shell/RCE`

**User idea:** known-vulnerability research → LFI/local file read → path traversal → FTP → SSH → password spraying → group-membership abuse

**Root idea:** service abuse → ACL abuse → SSH → web shell/RCE

[Open hint sheet](./htb-writeups/ServMon.md) · [Open official PDF](./htb-writeups/ServMon.pdf)

## 21- Support

**Keywords:** `LDAP enumeration`, `group-membership abuse`, `WinRM`, `BloodHound path analysis`, `ACL abuse`, `delegation/RBCD`, `Kerberos ticket abuse`, `WMI/PowerShell remoting`

**User idea:** LDAP enumeration → group-membership abuse → WinRM

**Root idea:** BloodHound path analysis → WinRM → group-membership abuse → ACL abuse → delegation/RBCD → Kerberos ticket abuse → WMI/PowerShell remoting

[Open hint sheet](./htb-writeups/Support.md) · [Open official PDF](./htb-writeups/Support.pdf)

## 22- Access

**Keywords:** `FTP enumeration`, `Microsoft Access DB secrets`, `protected-archive cracking`, `Telnet`, `saved-credential abuse`, `runas /savecred`, `Administrator shell`

**User idea:** FTP enumeration → Microsoft Access DB secrets → protected-archive cracking → Telnet

**Root idea:** saved-credential abuse → runas /savecred → Administrator shell

[Open hint sheet](./htb-writeups/Access.md) · [Open official PDF](./htb-writeups/Access.pdf)

## 23- Heist

**Keywords:** `network-device config secrets`, `Cisco hash cracking`, `RPC user enumeration`, `WinRM`, `process enumeration`, `process-memory credential recovery`, `Administrator credential reuse`

**User idea:** network-device config secrets → Cisco hash cracking → RPC user enumeration → WinRM

**Root idea:** process enumeration → process-memory credential recovery → Administrator credential reuse

[Open hint sheet](./htb-writeups/Heist.md) · [Open official PDF](./htb-writeups/Heist.pdf)

## 24- Active

**Keywords:** `anonymous SMB access`, `GPP cpassword recovery`, `password decryption`, `SMB access as SVC_TGS`, `Kerberoasting`, `password cracking`, `pass-the-hash/PsExec`

**User idea:** anonymous SMB access → GPP cpassword recovery → password decryption → SMB access as SVC_TGS

**Root idea:** Kerberoasting → password cracking → pass-the-hash/PsExec

[Open hint sheet](./htb-writeups/Active.md) · [Open official PDF](./htb-writeups/Active.pdf)

## 25- Forest

**Keywords:** `AS-REP roasting`, `password cracking`, `WinRM`, `BloodHound path analysis`, `ACL abuse`, `LDAP enumeration`, `group-membership abuse`, `DCSync/secrets dump`, `WMI/PowerShell remoting`

**User idea:** AS-REP roasting → password cracking → WinRM

**Root idea:** BloodHound path analysis → WinRM → ACL abuse → LDAP enumeration → group-membership abuse → DCSync/secrets dump → password cracking → WMI/PowerShell remoting

[Open hint sheet](./htb-writeups/Forest.md) · [Open official PDF](./htb-writeups/Forest.pdf)

## 26- Sauna

**Keywords:** `AS-REP roasting`, `password cracking`, `WinRM`, `BloodHound path analysis`, `DCSync/secrets dump`, `WMI/PowerShell remoting`

**User idea:** AS-REP roasting → password cracking → WinRM

**Root idea:** WinRM → BloodHound path analysis → DCSync/secrets dump → WMI/PowerShell remoting

[Open hint sheet](./htb-writeups/Sauna.md) · [Open official PDF](./htb-writeups/Sauna.pdf)

## 27- Return

**Keywords:** `LDAP enumeration`, `SMB enumeration`, `WinRM`, `group-membership abuse`, `service abuse`, `reverse shell`, `file upload abuse`

**User idea:** LDAP enumeration → SMB enumeration → WinRM

**Root idea:** group-membership abuse → service abuse → reverse shell → WinRM → file upload abuse

[Open hint sheet](./htb-writeups/Return.md) · [Open official PDF](./htb-writeups/Return.pdf)

## 28- Timelapse

**Keywords:** `SMB share enumeration`, `protected-archive cracking`, `PFX/certificate-key recovery`, `WinRM`, `PowerShell history secrets`, `lateral movement`, `LAPS password retrieval`, `WinRM as Administrator`

**User idea:** SMB share enumeration → protected-archive cracking → PFX/certificate-key recovery → WinRM

**Root idea:** PowerShell history secrets → lateral movement → LAPS password retrieval → WinRM as Administrator

[Open hint sheet](./htb-writeups/Timelapse.md) · [Open official PDF](./htb-writeups/Timelapse.pdf)

## 29- Cicada

**Keywords:** `SMB enumeration`, `WinRM`, `SeBackup abuse`, `SeRestore abuse`, `registry secrets`, `group-membership abuse`, `DCSync/secrets dump`, `password cracking`, `pass-the-hash`

**User idea:** SMB enumeration → WinRM

**Root idea:** WinRM → SeBackup abuse → SeRestore abuse → registry secrets → group-membership abuse → DCSync/secrets dump → password cracking → pass-the-hash

[Open hint sheet](./htb-writeups/Cicada.md) · [Open official PDF](./htb-writeups/Cicada.pdf)

## 30- Fluffy

**Keywords:** `BloodHound path analysis`, `ACL abuse`, `WinRM`, `group-membership abuse`, `shadow credentials/PKINIT`, `AD CS/certificate abuse`, `SMB enumeration`, `LDAP enumeration`, `PFX/certificate-key recovery`, `private-key discovery`, `Kerberos ticket abuse`

**User idea:** BloodHound path analysis → ACL abuse → WinRM → group-membership abuse → shadow credentials/PKINIT → AD CS/certificate abuse

**Root idea:** AD CS/certificate abuse → SMB enumeration → LDAP enumeration → PFX/certificate-key recovery → private-key discovery → Kerberos ticket abuse → WinRM

[Open hint sheet](./htb-writeups/Fluffy.md) · [Open official PDF](./htb-writeups/Fluffy.pdf)

## 31- Eighteen

**Keywords:** `password cracking`, `database enumeration`, `password spraying`, `WinRM`, `BloodHound path analysis`, `ACL abuse`, `Kerberos ticket abuse`, `SMB enumeration`, `LDAP enumeration`, `saved-credential abuse`, `reverse shell`

**User idea:** password cracking → database enumeration → password spraying → WinRM

**Root idea:** BloodHound path analysis → WinRM → ACL abuse → Kerberos ticket abuse → SMB enumeration → LDAP enumeration → saved-credential abuse → reverse shell

[Open hint sheet](./htb-writeups/Eighteen.md) · [Open official PDF](./htb-writeups/Eighteen.pdf)

## 32- Escape

**Keywords:** `database enumeration`, `NTLM relay/coercion`, `password cracking`, `WinRM`, `AD CS/certificate abuse`, `PFX/certificate-key recovery`, `pass-the-hash`, `Kerberos ticket abuse`

**User idea:** database enumeration → NTLM relay/coercion → password cracking → WinRM

**Root idea:** WinRM → AD CS/certificate abuse → PFX/certificate-key recovery → pass-the-hash → database enumeration → Kerberos ticket abuse

[Open hint sheet](./htb-writeups/Escape.md) · [Open official PDF](./htb-writeups/Escape.pdf)

## 33- Monteverde

**Keywords:** `password spraying`, `Git history/config`, `SMB enumeration`, `WinRM`, `service abuse`, `registry secrets`, `DPAPI/browser secrets`, `database enumeration`, `writable script/file abuse`

**User idea:** password spraying → Git history/config → SMB enumeration → WinRM

**Root idea:** WinRM → service abuse → registry secrets → DPAPI/browser secrets → database enumeration → writable script/file abuse

[Open hint sheet](./htb-writeups/Monteverde.md) · [Open official PDF](./htb-writeups/Monteverde.pdf)

## 34- Cascade

**Keywords:** `registry secrets`, `WinRM`, `WMI/PowerShell remoting`, `SMB enumeration`, `LDAP enumeration`, `group-membership abuse`, `credential reuse`

**User idea:** registry secrets → WinRM → WMI/PowerShell remoting → SMB enumeration → LDAP enumeration

**Root idea:** group-membership abuse → credential reuse → WinRM

[Open hint sheet](./htb-writeups/Cascade.md) · [Open official PDF](./htb-writeups/Cascade.pdf)

## 35- TheFrizz

**Keywords:** `CVE-2023-45878`, `database enumeration`, `SSH`, `configuration-file secrets`, `group-membership abuse`, `web shell/RCE`, `Git history/config`, `reverse shell`, `password cracking`, `Kerberos ticket abuse`, `BloodHound path analysis`, `cron/scheduled-job abuse`

**User idea:** CVE-2023-45878 → database enumeration → SSH → configuration-file secrets → group-membership abuse → web shell/RCE → Git history/config → reverse shell → password cracking → Kerberos ticket abuse

**Root idea:** SSH → Kerberos ticket abuse → BloodHound path analysis → cron/scheduled-job abuse → reverse shell

[Open hint sheet](./htb-writeups/TheFrizz.md) · [Open official PDF](./htb-writeups/TheFrizz.pdf)

## 36- Puppy

**Keywords:** `group-membership abuse`, `SMB enumeration`, `KeePass database recovery`, `password cracking`, `WinRM`, `BloodHound path analysis`, `ACL abuse`, `LDAP enumeration`, `DPAPI/browser secrets`

**User idea:** group-membership abuse → SMB enumeration → KeePass database recovery → password cracking → WinRM → BloodHound path analysis → ACL abuse → LDAP enumeration

**Root idea:** DPAPI/browser secrets → WinRM

[Open hint sheet](./htb-writeups/Puppy.md) · [Open official PDF](./htb-writeups/Puppy.pdf)

## 37- Voleur

**Keywords:** `SMB enumeration`, `password cracking`, `configuration-file secrets`, `BloodHound path analysis`, `LDAP enumeration`, `Kerberoasting`, `Kerberos ticket abuse`, `WinRM`, `saved-credential abuse`, `DPAPI/browser secrets`, `private-key discovery`, `SSH`, `registry secrets`, `DCSync/secrets dump`

**User idea:** SMB enumeration → password cracking → configuration-file secrets → BloodHound path analysis → LDAP enumeration → Kerberoasting → Kerberos ticket abuse → WinRM → saved-credential abuse → DPAPI/browser secrets

**Root idea:** private-key discovery → WinRM → SSH → registry secrets → DCSync/secrets dump → Kerberos ticket abuse

[Open hint sheet](./htb-writeups/Voleur.md) · [Open official PDF](./htb-writeups/Voleur.pdf)

## 38- Signed

**Keywords:** `database enumeration`, `password cracking`, `Kerberos ticket abuse`, `CVE-2025-33073`, `group-membership abuse`, `NTLM relay/coercion`, `SMB enumeration`, `LDAP enumeration`, `reverse shell`, `SSH`, `AD CS/certificate abuse`, `DCSync/secrets dump`, `WinRM`

**User idea:** database enumeration → password cracking → Kerberos ticket abuse → CVE-2025-33073 → group-membership abuse → NTLM relay/coercion → SMB enumeration → LDAP enumeration → reverse shell

**Root idea:** SMB enumeration → LDAP enumeration → SSH → database enumeration → group-membership abuse → NTLM relay/coercion → AD CS/certificate abuse → DCSync/secrets dump → WinRM → Kerberos ticket abuse

[Open hint sheet](./htb-writeups/Signed.md) · [Open official PDF](./htb-writeups/Signed.pdf)

## 39- TombWatcher

**Keywords:** `Kerberoasting`, `password cracking`, `gMSA password retrieval`, `BloodHound path analysis`, `group-membership abuse`, `ACL abuse`, `WinRM`, `AD CS/certificate abuse`, `registry secrets`, `private-key discovery`, `CVE-2024-49019`, `PFX/certificate-key recovery`

**User idea:** Kerberoasting → password cracking → gMSA password retrieval → BloodHound path analysis → group-membership abuse → ACL abuse → WinRM

**Root idea:** BloodHound path analysis → password cracking → ACL abuse → AD CS/certificate abuse → registry secrets → private-key discovery → WinRM → CVE-2024-49019 → group-membership abuse → PFX/certificate-key recovery

[Open hint sheet](./htb-writeups/TombWatcher.md) · [Open official PDF](./htb-writeups/TombWatcher.pdf)

## 40- UpDown

**Keywords:** `group-membership abuse`, `subdomain enumeration`, `source-code review`, `LFI/local file read`, `web shell/RCE`, `FTP`, `reverse shell`, `SUID abuse`, `private-key discovery`, `SSH`, `sudo rule abuse`

**User idea:** group-membership abuse → subdomain enumeration → source-code review → LFI/local file read → web shell/RCE → FTP → reverse shell → SUID abuse → private-key discovery → SSH

**Root idea:** SSH → sudo rule abuse → SUID abuse

[Open hint sheet](./htb-writeups/UpDown.md) · [Open official PDF](./htb-writeups/UpDown.pdf)

## 41- Magic

**Keywords:** `reverse shell`, `web shell/RCE`, `group-membership abuse`, `database enumeration`, `SUID abuse`, `PATH hijacking`

**User idea:** reverse shell → web shell/RCE → group-membership abuse → database enumeration

**Root idea:** SUID abuse → PATH hijacking → group-membership abuse → reverse shell

[Open hint sheet](./htb-writeups/Magic.md) · [Open official PDF](./htb-writeups/Magic.pdf)

## 42- Monitored

**Keywords:** `web shell/RCE`, `group-membership abuse`, `reverse shell`, `sudo rule abuse`, `source-code review`, `configuration-file secrets`, `symlink/race-condition abuse`, `SSH`, `private-key discovery`

**User idea:** web shell/RCE → group-membership abuse → reverse shell

**Root idea:** sudo rule abuse → source-code review → configuration-file secrets → symlink/race-condition abuse → SSH → private-key discovery

[Open hint sheet](./htb-writeups/Monitored.md) · [Open official PDF](./htb-writeups/Monitored.pdf)

## 43- Builder

**Keywords:** `Docker/container escape`, `LFI/local file read`, `password cracking`, `configuration-file secrets`, `SSH`, `private-key discovery`

**User idea:** Docker/container escape → LFI/local file read → password cracking → configuration-file secrets → SSH → private-key discovery

**Root idea:** SSH → private-key discovery

[Open hint sheet](./htb-writeups/Builder.md) · [Open official PDF](./htb-writeups/Builder.pdf)

## 44- Browsed

**Keywords:** `Git history/config`, `source-code review`, `command injection`, `reverse shell`, `SSH`, `private-key discovery`, `sudo rule abuse`

**User idea:** Git history/config → source-code review → command injection → reverse shell → SSH → private-key discovery

**Root idea:** sudo rule abuse → source-code review

[Open hint sheet](./htb-writeups/Browsed.md) · [Open official PDF](./htb-writeups/Browsed.pdf)

## 45- StreamIO

**Keywords:** `subdomain enumeration`, `SQL injection`, `LFI/local file read`, `source-code review`, `RFI`, `database enumeration`, `DPAPI/browser secrets`, `BloodHound path analysis`, `group-membership abuse`, `LDAP enumeration`, `WinRM`, `Git history/config`, `SMB enumeration`, `ACL abuse`, `LAPS password retrieval`, `GPP cpassword recovery`

**User idea:** subdomain enumeration → SQL injection → LFI/local file read → source-code review → RFI → database enumeration → DPAPI/browser secrets → BloodHound path analysis → group-membership abuse → LDAP enumeration

**Root idea:** WinRM → DPAPI/browser secrets → Git history/config → SMB enumeration → BloodHound path analysis → ACL abuse → LAPS password retrieval → group-membership abuse → GPP cpassword recovery → LDAP enumeration

[Open hint sheet](./htb-writeups/StreamIO.md) · [Open official PDF](./htb-writeups/StreamIO.pdf)

## 46- Intelligence

**Keywords:** `default credentials`, `SMB enumeration`, `source-code review`, `group-membership abuse`, `NTLM relay/coercion`, `password cracking`, `BloodHound path analysis`, `delegation/RBCD`, `WMI/PowerShell remoting`, `Kerberos ticket abuse`

**User idea:** default credentials → SMB enumeration → source-code review → group-membership abuse → NTLM relay/coercion → password cracking

**Root idea:** BloodHound path analysis → delegation/RBCD → WMI/PowerShell remoting → Kerberos ticket abuse

[Open hint sheet](./htb-writeups/Intelligence.md) · [Open official PDF](./htb-writeups/Intelligence.pdf)

## 47- Jeeves

**Keywords:** `pass-the-hash`, `reverse shell`, `KeePass database recovery`, `password cracking`

**User idea:** pass-the-hash → reverse shell → KeePass database recovery

**Root idea:** KeePass database recovery → password cracking → pass-the-hash

[Open hint sheet](./htb-writeups/Jeeves.md) · [Open official PDF](./htb-writeups/Jeeves.pdf)

## 48- Manager

**Keywords:** `database enumeration`, `configuration-file secrets`, `LDAP enumeration`, `WinRM`, `AD CS/certificate abuse`, `subdomain enumeration`, `private-key discovery`, `PFX/certificate-key recovery`, `Kerberos ticket abuse`

**User idea:** database enumeration → configuration-file secrets → LDAP enumeration → WinRM

**Root idea:** AD CS/certificate abuse → subdomain enumeration → LDAP enumeration → private-key discovery → PFX/certificate-key recovery → Kerberos ticket abuse → WinRM

[Open hint sheet](./htb-writeups/Manager.md) · [Open official PDF](./htb-writeups/Manager.pdf)

## 49- Aero

**Keywords:** `reverse shell`, `group-membership abuse`, `CVE-2023-28252`

**User idea:** reverse shell → group-membership abuse

**Root idea:** CVE-2023-28252 → reverse shell

[Open hint sheet](./htb-writeups/Aero.md) · [Open official PDF](./htb-writeups/Aero.pdf)

## 50- Administrator

**Keywords:** `ACL abuse`, `BloodHound path analysis`, `WinRM`, `GPP cpassword recovery`, `FTP`, `password cracking`, `SMB enumeration`, `Kerberoasting`, `Kerberos ticket abuse`, `DCSync/secrets dump`

**User idea:** ACL abuse → BloodHound path analysis → WinRM → GPP cpassword recovery → FTP → password cracking → SMB enumeration

**Root idea:** BloodHound path analysis → ACL abuse → Kerberoasting → Kerberos ticket abuse → password cracking → DCSync/secrets dump → WinRM

[Open hint sheet](./htb-writeups/Administrator.md) · [Open official PDF](./htb-writeups/Administrator.pdf)

## 51- Certified

**Keywords:** `ACL abuse`, `group-membership abuse`, `shadow credentials/PKINIT`, `PFX/certificate-key recovery`, `AD CS/certificate abuse`, `Kerberos ticket abuse`, `pass-the-hash`, `WinRM`, `BloodHound path analysis`, `SMB enumeration`, `LDAP enumeration`, `private-key discovery`

**User idea:** ACL abuse → group-membership abuse → shadow credentials/PKINIT → PFX/certificate-key recovery → AD CS/certificate abuse → Kerberos ticket abuse → pass-the-hash → WinRM → BloodHound path analysis

**Root idea:** SMB enumeration → LDAP enumeration → AD CS/certificate abuse → PFX/certificate-key recovery → private-key discovery → Kerberos ticket abuse → WinRM

[Open hint sheet](./htb-writeups/Certified.md) · [Open official PDF](./htb-writeups/Certified.pdf)

## 52- Mailing

**Keywords:** `path traversal`, `configuration-file secrets`, `password cracking`, `CVE-2024-21413`, `SMB enumeration`, `web shell/RCE`, `WinRM`, `CVE-2023-2255`, `Git history/config`, `reverse shell`

**User idea:** path traversal → configuration-file secrets → password cracking → CVE-2024-21413 → SMB enumeration → web shell/RCE → WinRM

**Root idea:** WinRM → CVE-2023-2255 → Git history/config → reverse shell

[Open hint sheet](./htb-writeups/Mailing.md) · [Open official PDF](./htb-writeups/Mailing.pdf)

## 53- Intentions

**Keywords:** `web shell/RCE`, `RFI`, `reverse shell`, `Git history/config`, `configuration-file secrets`, `SSH`, `SUID abuse`, `Linux capabilities`, `private-key discovery`

**User idea:** web shell/RCE → RFI → reverse shell → Git history/config → configuration-file secrets → SSH

**Root idea:** SUID abuse → Linux capabilities → SSH → private-key discovery

[Open hint sheet](./htb-writeups/Intentions.md) · [Open official PDF](./htb-writeups/Intentions.pdf)

## 54- Flight

**Keywords:** `SMB enumeration`, `password spraying`, `NTLM relay/coercion`, `password cracking`, `web shell/RCE`, `saved-credential abuse`, `DCSync/secrets dump`, `Kerberos ticket abuse`, `WMI/PowerShell remoting`, `SeImpersonate/token abuse`

**User idea:** SMB enumeration → password spraying → NTLM relay/coercion → password cracking

**Root idea:** web shell/RCE → saved-credential abuse → DCSync/secrets dump → Kerberos ticket abuse → WMI/PowerShell remoting → SeImpersonate/token abuse

[Open hint sheet](./htb-writeups/Flight.md) · [Open official PDF](./htb-writeups/Flight.pdf)

## 55- Blackfield

**Keywords:** `SMB enumeration`, `AS-REP roasting`, `password cracking`, `BloodHound path analysis`, `process-memory credential recovery`, `Kerberos ticket abuse`, `DPAPI/browser secrets`, `saved-credential abuse`, `cron/scheduled-job abuse`, `WMI/PowerShell remoting`, `SeBackup abuse`, `SeRestore abuse`, `pass-the-hash`, `DCSync/secrets dump`, `registry secrets`, `WinRM`

**User idea:** SMB enumeration → AS-REP roasting → password cracking → BloodHound path analysis → process-memory credential recovery → Kerberos ticket abuse → DPAPI/browser secrets → saved-credential abuse → cron/scheduled-job abuse → WMI/PowerShell remoting

**Root idea:** SeBackup abuse → SeRestore abuse → pass-the-hash → SMB enumeration → DCSync/secrets dump → registry secrets → WMI/PowerShell remoting → WinRM

[Open hint sheet](./htb-writeups/Blackfield.md) · [Open official PDF](./htb-writeups/Blackfield.pdf)

## 56- EscapeTwo

**Keywords:** `database enumeration`, `reverse shell`, `configuration-file secrets`, `GPP cpassword recovery`, `SMB enumeration`, `password spraying`, `WinRM`, `BloodHound path analysis`, `LDAP enumeration`, `ACL abuse`, `AD CS/certificate abuse`, `PFX/certificate-key recovery`, `private-key discovery`, `pass-the-hash`, `Kerberos ticket abuse`

**User idea:** database enumeration → reverse shell → configuration-file secrets → GPP cpassword recovery → SMB enumeration → password spraying → WinRM → BloodHound path analysis → LDAP enumeration

**Root idea:** BloodHound path analysis → ACL abuse → WinRM → SMB enumeration → AD CS/certificate abuse → PFX/certificate-key recovery → private-key discovery → pass-the-hash → Kerberos ticket abuse

[Open hint sheet](./htb-writeups/EscapeTwo.md) · [Open official PDF](./htb-writeups/EscapeTwo.pdf)

## 57- Mentor

**Keywords:** `SNMP enumeration`, `group-membership abuse`, `reverse shell`, `Docker/container escape`, `database enumeration`, `default credentials`, `web shell/RCE`, `password cracking`, `SSH`, `sudo rule abuse`

**User idea:** SNMP enumeration → group-membership abuse → reverse shell → Docker/container escape → database enumeration → default credentials → web shell/RCE → password cracking → SSH

**Root idea:** SNMP enumeration → sudo rule abuse

[Open hint sheet](./htb-writeups/Mentor.md) · [Open official PDF](./htb-writeups/Mentor.pdf)

## 58- Outdated

**Keywords:** `password cracking`, `path traversal`, `reverse shell`, `web shell/RCE`, `group-membership abuse`, `BloodHound path analysis`, `SMB enumeration`, `shadow credentials/PKINIT`, `private-key discovery`, `Kerberos ticket abuse`, `registry secrets`, `WinRM`, `WMI/PowerShell remoting`

**User idea:** password cracking → path traversal → reverse shell → web shell/RCE → group-membership abuse → BloodHound path analysis → SMB enumeration → shadow credentials/PKINIT → private-key discovery → Kerberos ticket abuse

**Root idea:** registry secrets → BloodHound path analysis → group-membership abuse → WinRM → reverse shell → WMI/PowerShell remoting

[Open hint sheet](./htb-writeups/Outdated.md) · [Open official PDF](./htb-writeups/Outdated.pdf)

## 59- Atom

**Keywords:** `web shell/RCE`, `command injection`, `SMB enumeration`, `database enumeration`, `WinRM`

**User idea:** web shell/RCE → command injection → SMB enumeration

**Root idea:** database enumeration → WinRM

[Open hint sheet](./htb-writeups/Atom.md) · [Open official PDF](./htb-writeups/Atom.pdf)

## 60- Authority

**Keywords:** `LDAP enumeration`, `reverse shell`, `WinRM`, `AD CS/certificate abuse`, `registry secrets`, `BloodHound path analysis`, `private-key discovery`, `subdomain enumeration`, `group-membership abuse`, `SMB enumeration`, `PFX/certificate-key recovery`, `pass-the-hash`

**User idea:** LDAP enumeration → reverse shell → WinRM

**Root idea:** AD CS/certificate abuse → registry secrets → BloodHound path analysis → private-key discovery → subdomain enumeration → group-membership abuse → SMB enumeration → LDAP enumeration → PFX/certificate-key recovery → pass-the-hash

[Open hint sheet](./htb-writeups/Authority.md) · [Open official PDF](./htb-writeups/Authority.pdf)

## 61- Clicker

**Keywords:** `source-code review`, `SQL injection`, `web shell/RCE`, `reverse shell`, `SUID abuse`, `NFS/share abuse`, `database enumeration`, `path traversal`, `SSH`, `private-key discovery`, `sudo rule abuse`, `XXE`

**User idea:** source-code review → SQL injection → web shell/RCE → reverse shell → SUID abuse → NFS/share abuse → database enumeration → path traversal → SSH → private-key discovery

**Root idea:** sudo rule abuse → XXE → private-key discovery → SSH

[Open hint sheet](./htb-writeups/Clicker.md) · [Open official PDF](./htb-writeups/Clicker.pdf)

## 62- Cerberus

**Keywords:** `SSH`, `LFI/local file read`, `private-key discovery`, `path traversal`, `reverse shell`, `SUID abuse`, `symlink/race-condition abuse`, `password cracking`, `WinRM`, `Docker/container escape`, `default credentials`

**User idea:** SSH → LFI/local file read → private-key discovery → path traversal → reverse shell → SUID abuse → symlink/race-condition abuse → password cracking → WinRM → Docker/container escape

**Root idea:** WinRM → default credentials

[Open hint sheet](./htb-writeups/Cerberus.md) · [Open official PDF](./htb-writeups/Cerberus.pdf)

## 63- Cereal

**Keywords:** `unsafe deserialization`, `source-code review`, `configuration-file secrets`, `web shell/RCE`, `Git history/config`, `SSH`, `SeImpersonate/token abuse`, `SSRF`, `reverse shell`, `NTLM relay/coercion`

**User idea:** unsafe deserialization → source-code review → configuration-file secrets → web shell/RCE → Git history/config → SSH

**Root idea:** SeImpersonate/token abuse → SSH → SSRF → reverse shell → NTLM relay/coercion

[Open hint sheet](./htb-writeups/Cereal.md) · [Open official PDF](./htb-writeups/Cereal.pdf)

## 64- Quick

**Keywords:** `reverse shell`, `printer-configuration secrets`, `virtual-host enumeration`, `group-membership abuse`, `subdomain enumeration`, `default credentials`, `database enumeration`, `source-code review`, `symlink/race-condition abuse`, `SSH`, `password decryption`, `credential reuse`, `su to root`

**User idea:** reverse shell → printer-configuration secrets → virtual-host enumeration → group-membership abuse → subdomain enumeration → default credentials → database enumeration → source-code review → symlink/race-condition abuse → SSH

**Root idea:** printer-configuration secrets → password decryption → credential reuse → su to root

[Open hint sheet](./htb-writeups/Quick.md) · [Open official PDF](./htb-writeups/Quick.pdf)

## 65- Vintage

**Keywords:** `SMB enumeration`, `LDAP enumeration`, `BloodHound path analysis`, `ACL abuse`, `gMSA password retrieval`, `group-membership abuse`, `Kerberos ticket abuse`, `Kerberoasting`, `password cracking`, `WinRM`, `delegation/RBCD`

**User idea:** SMB enumeration → LDAP enumeration → BloodHound path analysis → ACL abuse → gMSA password retrieval → group-membership abuse → Kerberos ticket abuse → Kerberoasting → password cracking → WinRM

**Root idea:** Kerberos ticket abuse → delegation/RBCD → WinRM

[Open hint sheet](./htb-writeups/Vintage.md) · [Open official PDF](./htb-writeups/Vintage.pdf)

## 66- RustyKey

**Keywords:** `BloodHound path analysis`, `group-membership abuse`, `ticket/helpdesk disclosure`, `ACL abuse`, `WinRM`, `Kerberos ticket abuse`, `SMB enumeration`, `registry secrets`, `reverse shell`, `saved-credential abuse`, `delegation/RBCD`, `LDAP enumeration`, `DCSync/secrets dump`, `WMI/PowerShell remoting`

**User idea:** BloodHound path analysis → group-membership abuse → ticket/helpdesk disclosure → ACL abuse → WinRM → Kerberos ticket abuse → SMB enumeration → registry secrets → reverse shell → saved-credential abuse

**Root idea:** group-membership abuse → delegation/RBCD → LDAP enumeration → ACL abuse → reverse shell → ticket/helpdesk disclosure → Kerberos ticket abuse → BloodHound path analysis → DCSync/secrets dump → WMI/PowerShell remoting

[Open hint sheet](./htb-writeups/RustyKey.md) · [Open official PDF](./htb-writeups/RustyKey.pdf)

## 67- DarkZero

**Keywords:** `database enumeration`, `password cracking`, `reverse shell`, `SeImpersonate/token abuse`, `registry secrets`, `saved-credential abuse`, `Kerberos ticket abuse`, `AD CS/certificate abuse`, `PFX/certificate-key recovery`, `private-key discovery`, `delegation/RBCD`, `DCSync/secrets dump`, `WinRM`, `group-membership abuse`

**User idea:** database enumeration → password cracking → reverse shell → SeImpersonate/token abuse → registry secrets → saved-credential abuse → Kerberos ticket abuse → AD CS/certificate abuse → PFX/certificate-key recovery → private-key discovery

**Root idea:** delegation/RBCD → database enumeration → password cracking → Kerberos ticket abuse → DCSync/secrets dump → WinRM → SeImpersonate/token abuse → group-membership abuse

[Open hint sheet](./htb-writeups/DarkZero.md) · [Open official PDF](./htb-writeups/DarkZero.pdf)

## 68- Absolute

**Keywords:** `AS-REP roasting`, `password cracking`, `LDAP enumeration`, `Kerberos ticket abuse`, `SMB enumeration`, `WinRM`, `BloodHound path analysis`, `default credentials`, `ACL abuse`, `group-membership abuse`, `saved-credential abuse`

**User idea:** AS-REP roasting → password cracking → LDAP enumeration → Kerberos ticket abuse → SMB enumeration → WinRM → BloodHound path analysis → default credentials → ACL abuse → group-membership abuse

**Root idea:** Kerberos ticket abuse → LDAP enumeration → WinRM → saved-credential abuse

[Open hint sheet](./htb-writeups/Absolute.md) · [Open official PDF](./htb-writeups/Absolute.pdf)

## 69- APT

**Keywords:** `WinRM`, `registry secrets`, `SeImpersonate/token abuse`, `Git history/config`, `NTLM relay/coercion`, `password cracking`, `DCSync/secrets dump`

**User idea:** WinRM → registry secrets

**Root idea:** SeImpersonate/token abuse → Git history/config → NTLM relay/coercion → password cracking → DCSync/secrets dump → WinRM

[Open hint sheet](./htb-writeups/APT.md) · [Open official PDF](./htb-writeups/APT.pdf)

## 70- Multimaster

**Keywords:** `SMB enumeration`, `WinRM`, `reverse shell`, `source-code review`, `password spraying`, `BloodHound path analysis`, `ACL abuse`, `group-membership abuse`, `AS-REP roasting`, `password cracking`, `service abuse`, `WMI/PowerShell remoting`, `SeBackup abuse`, `CVE-2020-1472`

**User idea:** SMB enumeration → WinRM → reverse shell → source-code review → password spraying

**Root idea:** BloodHound path analysis → ACL abuse → group-membership abuse → AS-REP roasting → password cracking → WinRM → service abuse → WMI/PowerShell remoting → SeBackup abuse → CVE-2020-1472

[Open hint sheet](./htb-writeups/Multimaster.md) · [Open official PDF](./htb-writeups/Multimaster.pdf)

## 71- Rebound

**Keywords:** `AS-REP roasting`, `Kerberoasting`, `Git history/config`, `password cracking`, `BloodHound path analysis`, `ACL abuse`, `LDAP enumeration`, `group-membership abuse`, `SMB enumeration`, `WinRM`, `delegation/RBCD`, `Kerberos ticket abuse`, `DCSync/secrets dump`

**User idea:** AS-REP roasting → Kerberoasting → Git history/config → password cracking → BloodHound path analysis → ACL abuse → LDAP enumeration → group-membership abuse → SMB enumeration → WinRM

**Root idea:** delegation/RBCD → BloodHound path analysis → Kerberos ticket abuse → DCSync/secrets dump → WinRM

[Open hint sheet](./htb-writeups/Rebound.md) · [Open official PDF](./htb-writeups/Rebound.pdf)
