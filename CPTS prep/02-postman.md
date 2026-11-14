<p align="center">
  <img src="../assets/branding/nu11s3c-banner.svg" alt="nu11s3c security writeups" width="100%" />
</p>

# postman - hack the box

**platform:** Hack The Box · retired machine  
**os:** Linux (Ubuntu 18.04.3 LTS)  
**outcome:** SSH as `redis`, local access as `Matt`, Linux root  
**focus:** Redis · SSH · insecure file permissions · John the Ripper · password reuse · Webmin · CVE-2019-12840

---

## `00 / prep`

init engagement structure, using fish for environment variables and tmux for logging.

```bash
CPTS/CPTS-Day03/postman
❯ ll
total 24K
-rw-rw-r-- 1 kali kali    0 Nov 14 00:07 env.fish
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:07 logs/
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:07 loot/
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:07 notes/
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:08 report/
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:08 scans/
drwxrwxr-x 2 kali kali 4.0K Nov 14 00:07 screenshots/

CPTS/CPTS-Day03/postman took 14s
❯ cat env.fish
# Engagement configuration
set -gx ENGAGEMENT "delivery"
set -gx IP "10.129.2.1"

# New logging destination
set -gx ENGAGEMENT_DIR (pwd -P)

# Copy to tmux session
if set -q TMUX
	set -l session (tmux display-message -p '#S')

	tmux set-environment -t "$session" ENGAGEMENT "$ENGAGEMENT"
	tmux set-environment -t "$session" IP "$IP"
	tmux set-environment -t "$session" ENGAGEMENT_DIR "$ENGAGEMENT_DIR"
end

CPTS/CPTS-Day03/postman
❯ source ./env.fish

CPTS/CPTS-Day03/postman
❯ tree
.
├── env.fish
├── logs
│   └── tmux
│       ├── s0-p0-20261114T001411-aB6NwW.log
│       └── s0-p12-20261114T001615-6dZTn1.log
├── loot
├── notes
│   └── Notes.md
└── scans

6 directories, 4 files
```


---

## `01 / recon`

### service discovery

began with a port and service scan of the top ports

```console
❯ nmap -sVC -oA scans/initial $IP
Nmap scan report for 10.129.2.1
Host is up (0.045s latency).
Not shown: 997 closed tcp ports (reset)

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: The Cyber Geek's Personal Website
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

a full port scan exposed another service, Redis on port 6379

```shell
❯ nmap -p- --min-rate 1000 -oA scans/allports $IP
Nmap scan report for 10.129.2.1
Host is up (0.044s latency).
Not shown: 65531 closed tcp ports (reset)

PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
6379/tcp  open  redis
10000/tcp open  snet-sensor-mgmt

❯ nmap -sV -p 6379 $IP
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 4.0.9
```

| port | service | version |
|---|---|---|
| 22 | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80 | HTTP | Apache 2.4.29 |
| 6379 | Redis | 4.0.9 |
| 10000 | Webmin | MiniServ 1.910 |

the conventional service name for port 10000 wasn't particularly useful. the version scan had already identified it as Webmin. Redis caught my attention since it wasn't even in the initial scan.

### redis enumeration

used the native Redis client to see what information I could get from the service. didn't even need to authenticate.

```console
❯ redis-cli -h $IP -p 6379
10.129.2.1:6379> info
# Server
redis_version:4.0.9
redis_mode:standalone
os:Linux 4.15.0-58-generic x86_64
tcp_port:6379
executable:/usr/bin/redis-server
config_file:/etc/redis/redis.conf
# Persistence
rdb_last_bgsave_status:ok
aof_enabled:0
# Replication
role:master
connected_slaves:0
# Cluster
cluster_enabled:0
# Keyspace
```

I wasn't particularly familiar with Redis so spent some time going through its built-in help, including the configuration commands.

```console
10.129.2.1:6379> help CONFIG
<--snip-->
10.129.2.1:6379> config get *
<--snip-->
```

some of the more relevant configuration values:

| parameter | value |
|---|---|
| requirepass | empty |
| protected-mode | no |
| bind | `0.0.0.0 ::1` |
| dir | `/var/lib/redis` |
| dbfilename | `dump.rdb` |
| appendonly | no |

not only was the service accessible remotely without a password, but I could read its configuration as well. started looking into Redis persistence and how that could affect files on the host.

---

## `02 / initial access`

### redis persistence and SSH access

spent more time on this than I probably needed to. went down the Redis RCE rabbit hole before finding an article about Redis and SSH keys.

Redis's persistence functionality can write a database snapshot to disk. in this case the lack of authentication and the service account's filesystem permissions made it possible to write our own ssh keys to the box

```console
❯ ssh-keygen
Generating public/private ed25519 key pair.
Your identification has been saved in ./ssh
Your public key has been saved in ./ssh.pub
```

after testing the Redis persistence issue, I verified that SSH access actually worked.

```shell
❯ ssh -i ssh redis@$IP
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)
<--snip-->
redis@Postman:~$
```

initial access as the service account

reference: [Elastic - Potential Redis Configuration Set SSH Key Injection](https://www.elastic.co/docs/reference/security/prebuilt-rules/rules/network/persistence_potential_redis_config_set_ssh_key_injection)

---

## `03 / local enumeration and user access`

### users and files

began with some basic enumeration as `redis`. sudo asked for a password, so I continued looking around the filesystem rather than assuming sudo was going to get me anywhere.

```shell
redis@Postman:~$ pwd
/var/lib/redis
redis@Postman:~$ sudo -l
[sudo] password for redis:
Sorry, try again.
```

```shell
redis@Postman:~$ cat /etc/passwd
<--snip-->
Matt:x:1000:1000:,,,:/home/Matt:/bin/bash
redis:x:107:114::/var/lib/redis:/bin/bash

redis@Postman:~$ ls -la /home
total 12
drwxr-xr-x  3 root root 4096 Sep 11  2019 .
drwxr-xr-x 22 root root 4096 Aug 25  2019 ..
drwxr-xr-x  6 Matt Matt 4096 Sep 11  2019 Matt

redis@Postman:~$ ls -la /opt
total 12
drwxr-xr-x  2 root root 4096 Sep 11  2019 .
drwxr-xr-x 22 root root 4096 Aug 25  2019 ..
-rwxr-xr-x  1 Matt Matt 1743 Aug 26  2019 id_rsa.bak
```

`Matt` was the regular user on the box. found a backup of his SSH private key sitting in `/opt`, readable by the Redis service account. the key was encrypted, so simply having a copy of it wasn't enough.

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,[redacted]
<--key material redacted-->
-----END RSA PRIVATE KEY-----
```

### offline passphrase recovery

used John the Ripper to assess the strength of the private key's passphrase. it recovered the passphrase in about a second.

```text
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
<--snip-->
[redacted]     (key)
1g 0:00:00:01 DONE (2026-11-14 10:44)
Session completed
```

### authentication and password reuse

initially tried to use the recovered key for SSH. had the permissions on my local copy wrong and OpenSSH refused to use it.

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0664 for 'key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "key": bad permissions
```

a subsequent SSH attempt against `matt` didn't get me in either. also worth noting that the actual Linux username was `Matt` with a capital M.

instead, the recovered passphrase turned out to be reused as Matt's local account password.

```shell
redis@Postman:~$ su Matt
Password:
Matt@Postman:/var/lib/redis$
```

same lesson as Delivery: different credentials and different authentication mechanisms don't necessarily mean different passwords.


---

## `04 / privilege escalation`

### process and service enumeration

Webmin had been sitting there on port 10000 since the first scan. I wanted to verify what was actually running on the host and under what user.

```shell
Matt@Postman:/var/www/html$ ps auxww
<--snip-->
root        797  0.0  3.1  91064 28800 ?        Ss   14:31   0:00 /usr/bin/perl /usr/share/webmin/miniserv.pl /etc/webmin/miniserv.conf
<--snip-->
```

also looked at the process tree, listening ports, and running services.

```shell
Matt@Postman:/var/www/html$ ps -eo user,pid,ppid,args --forest
<--snip-->
Matt@Postman:/var/www/html$ ss -tulpn
<--snip-->
Matt@Postman:/var/www/html$ systemctl list-units --type=service --state=running
<--snip-->
webmin.service   loaded active running LSB: web-based administration interfa

Matt@Postman:/var/www/html$ systemctl status webmin
● webmin.service - LSB: web-based administration interface for Unix systems
   Loaded: loaded (/etc/init.d/webmin; generated)
   Active: active (running)
<--snip-->

Matt@Postman:/var/www/html$ dpkg -l | grep -i webmin
ii  webmin  1.910  all  web-based administration interface for Unix systems

Matt@Postman:/var/www/html$ ps -p 797 -o user,pid,ppid,args
USER        PID   PPID COMMAND
root        797      1 /usr/bin/perl /usr/share/webmin/miniserv.pl /etc/webmin/miniserv
```

confirmed Webmin 1.910, running as root.

### authenticated enumeration

was able to authenticate to Webmin as Matt. didn't have access to much of the interface, just system information and software package updates/repositories.

![](../assets/images/postman/Pasted%20image%2020261114123529.png)

Webmin's dashboard confirmed version 1.910 and 181 available package updates, 115 of which were security updates.

![](../assets/images/postman/Pasted%20image%2020261114123557.png)

under System, the only visible module was Software Package Updates.

had already found some authenticated RCE vulnerabilities for Webmin, but was struggling to make the connection between those and the limited functionality I had access to. initially wondered if I was supposed to create a malicious package or repository. that wasn't necessary.

the important distinction here was authentication vs module permissions. Matt didn't have full administrative access to Webmin, but did have access to a module that interacted with the system package manager.

### CVE-2019-12840

went back through the vulnerabilities with those constraints in mind. CVE-2019-12840 matched the installed version and the Software Package Updates module.

spent some time messing around with shell quoting, base64 and URL encoding in fish while investigating this. eventually used a public proof of concept by KrE80r for the lab.

```text
Webmin <= 1.910 RCE (Authorization Required)

[*] logging in ...
[+] got sid [redacted]
[*] sending command [payload omitted]
[+] exploit finished successfully!!
```

verified the result rather than relying on the exploit's success message.

```shell
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

root.


---

## `05 / findings and lessons`

### systems / methods / findings

| system | method | finding |
|---|---|---|
| Redis | full port scan and native client enumeration | unauthenticated service with protected mode disabled |
| Redis / SSH | persistence assessment | filesystem access affected SSH authentication; shell as redis |
| Linux | manual user and filesystem enumeration | readable backup of Matt's encrypted SSH private key |
| SSH key | offline passphrase assessment | weak passphrase recovered with John the Ripper |
| Linux | local authentication | passphrase reuse allowed access as Matt |
| Webmin | process, version and permission enumeration | root-owned Webmin 1.910 with access to Software Package Updates |
| Webmin | CVE-2019-12840 assessment | authenticated command execution confirmed with root privileges |

### methodology

- full port scans matter. Redis wasn't in my first scan and ended up being the initial access vector.
- use the service's native client and understand its intended functionality before looking for an exploit.
- check file ownership and permissions. an encrypted private key is still sensitive, especially if it has a weak passphrase.
- don't confuse a failed SSH login with a bad password. local key permissions, the username and the server's authentication policy are separate things.
- enumerate what an authenticated application user can actually do. a version match alone doesn't establish that a vulnerability applies.

### references

- [Elastic - Potential Redis Configuration Set SSH Key Injection](https://www.elastic.co/docs/reference/security/prebuilt-rules/rules/network/persistence_potential_redis_config_set_ssh_key_injection)
- CVE-2019-12840 - Webmin Package Updates Remote Command Execution (proof of concept credited to KrE80r in the original notes).

