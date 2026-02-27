# Target: Meow

**Platform:** HackTheBox
[Meow on HackTheBox](https://app.hackthebox.com/machines/Meow)

| Target Name | IP Address | OS | Difficulty |
| --- | --- | --- | --- |
| Meow | 10.129.x.x | Linux | Very Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by scanning the IP address with Nmap to identify open ports and services.

**Nmap Scan:**

```bash
nmap 10.129.x.x

```

After the scan finished, I saw that port 23 was open with the Telnet service running.

## Phase 2: Initial Foothold (Gaining Access)

Since port 23 was open, I accessed the machine by using the `telnet` command in the Linux terminal.

**Connecting with the Machine:**

```bash
telnet 10.129.x.x

```

This showed me the HackTheBox login page. I logged in by using the username `root`, which was hinted in the machine's description. After logging in, I successfully got access to the machine terminal.

## Phase 3: Privilege Escalation & Post-Exploitation

Because I logged in directly as `root`, I already had the highest privileges on the system and no privilege escalation was required. I proceeded to explore the system by checking the current directory contents.

**Local Enumeration:**

```bash
ls

```

```
flag.txt

```

By using the `ls` command, I saw that there is a file named `flag.txt`, which was the exact flag I was looking for. I used the `cat` command to read its contents.

**Reading the flag:**

```bash
cat flag.txt

```

```
[Hashed flag]

```

The system showed me the flag, which I pasted on the page and it was correct. It ends the main task on this machine.

## Conclusion & Takeaways

**Summary:** This machine was very easy, making it a good starting point for beginners. The main task was short, easy, and didn't require any specific or complex tools. Access was gained simply by enumerating open ports and logging into the exposed Telnet service using an unauthenticated `root` account.

**What I Learned:** * **Basic Enumeration:** The importance of scanning for standard open ports like Telnet (port 23).

* **Default Credentials:** How critical it is to check for default or blank passwords, especially on administrative accounts.

**Remediation:** 
1. **Disable Telnet:** Telnet is an insecure protocol that transmits data in plain text. It should be disabled and replaced with SSH (Secure Shell) for remote administration.
2. **Secure Accounts:** Administrative accounts, such as `root`, should never be left with blank or default passwords. Strong, complex passwords or SSH keys should be strictly enforced.
