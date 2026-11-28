<p align="center">
  <img src="assets/branding/nu11s3c-banner.svg" alt="nu11s3c — Offensive security writeups" width="100%" />
</p>

<h1 align="center">nu11s3c / writeups</h1>


<p align="center">
  <kbd>windows</kbd> &nbsp; <kbd>linux</kbd> &nbsp; <kbd>networking</kbd> &nbsp; <kbd>web</kbd> &nbsp; <kbd>active directory</kbd>
</p>

---

## `00 / index`

| lab | platform | target | techniques |
|:---|:---|:---|:---|
| [access](CPTS%20prep/00-access.md) | hack the box · retired | windows server 2008 R2 | anonymous ftp -> file analysis -> telnet -> stored credentials / DPAPI |
| [delivery](CPTS%20prep/01-delivery.md) | hack the box · retired | linux (debian) | osTicket -> mattermost -> ssh -> mariaDB / bcrypt -> password reuse |
| [postman](CPTS%20prep/02-postman.md) | hack the box · retired | linux (ubunut) | redis -> ssh -> exposed key / passphrase reuse -> webmin |
| [underpass](CPTS%20prep/03-underpass.md) | hack the box · retired | linux (ubunut) | UDP / SNMP · daloRADIUS · credential security · sudo / mosh |
| [busqueda](CPTS%20prep/04-busqueda.md) | hack the box · retired | linux (ubunut) | searchor · git credentials · docker inspection · sudo / relative path |
| [monitored](CPTS%20prep/05-monitored.md) | hack the box · retired | linux (debian) | SNMP -> Nagios XI · SQL injection / API keys · monitor commands · sudo / systemd |
| [trick](CPTS%20prep/06-trick.md) | hack the box · retired | linux (debian) | DNS AXFR · payroll SQLi / PHP source · marketing LFI / SSH key · fail2ban |
| [forest](CPTS%20prep/07-forest.md) | hack the box · retired | windows server 2016 | LDAP / RPC · AS-REP roasting -> WinRM · Exchange permissions / WriteDacl -> DCSync · pass-the-hash |
| [driver](CPTS%20prep/08-driver.md) | hack the box · retired | windows 10 | HTTP Basic auth · SCF / NetNTLMv2 · WinRM · Ricoh driver ACLs · PrintNightmare |
| [magic](CPTS%20prep/09-magic.md) | hack the box · retired | linux (ubuntu) | SQL injection · upload validation · password reuse · SUID / PATH |
| [cascade](CPTS%20prep/10-cascade.md) | hack the box · retired | windows server 2008 R2 | LDAP · SMB · TightVNC · .NET decompilation / AES · AD Recycle Bin · password reuse |


---

<p align="center"><sub>nu11s3c</sub></p>
