AD: Basic Enumeration Room - THM

**Task 1** - Introduction
In the real world penetration tests, you will be given VPN access to the target network without user credentials. You have to gather as much informatin as possible about the domain: user, groups, and policies.
To ensure that your connection is established use command `route` or `ip route`, them will show you networks which you had access to and if you see the target network here you are connected with the target network.

**Task 2** - Mapping Out the Network
Very first thing to do is to scan the network to check which hosts and services are online. You can use `Nmap` as you have been doing to this time, or use `fping` tool.
**Nmap Scan:**
```bash
└─$ nmap -sn 10.211.11.0/24
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-22 09:18 +0200
Nmap scan report for 10.211.11.10
Host is up (0.046s latency).
Nmap scan report for 10.211.11.20
Host is up (0.044s latency).
Nmap scan report for 10.211.11.250
Host is up (0.047s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 6.61 seconds
```

`-sn` - Ping scan to determinate wchich hosts are up without port scanning.

**fping Scan:**
```bash
└─$ fping -agq 10.211.11.0/24
10.211.11.10
10.211.11.20
10.211.11.250
```

`-a` - Shows systems that are alive.
`-g` - Generates a terget list from a supplied IP netmask.
`-q` - Quiet mode, doesn't show per-probe results or ICMP error messages.

In this case there are 3 hosts up, but `10.211.11.250` is a VPN server so you can ignore it. Next make a file with these two hosts IP Addresses named `hosts.txt`, it will be used for port scans.

Common AD ports and protocols:
==========================================================================
||Port||     Protocol    ||What it Means                                ||
|| 88 ||     Kerberos    ||Potential for Kerberos-based enumeration     ||
||135 ||      MS-RPC     ||Potential for RPC enumeration (null sessions)||
||139 ||   SMB/NetBIOS   ||Legacy SMB access				||
||389 ||       LDAP      ||LDAP queries to AD				||
||445 ||        SMB      ||Modern SMB access, critical for enumeration	||
||464 ||Kerberos(kpasswd)||Password-related Kerberos service		||
==========================================================================

To scan common ports on hosts you don't have to scan every host separately, you can scan all hosts from the `hosts.txt` file.
**Nmap Ports Scan:**
```bash
└─$ nmap -p 88,135,139,389,445 -sV -sC -iL hosts.txt 
```
`-sV` - This enables version detection. Nmap will try to determiate the version of the services running on the open ports.
`-sC` - Runs Nmap Scripting Engine (NSE) scripts in the default category.
`-iL` - This tells Nmap to read the list of target hosts from the file `hosts.txt`. Each line in this file should contain a single IP Address or hostname.

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-22 09:32 +0200
Nmap scan report for 10.211.11.20
Host is up (0.044s latency).

PORT    STATE  SERVICE       VERSION
88/tcp  closed kerberos-sec
135/tcp open   msrpc         Microsoft Windows RPC
139/tcp open   netbios-ssn   Microsoft Windows netbios-ssn
389/tcp closed ldap
445/tcp open   microsoft-ds?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-05-22T07:32:46
|_  start_date: N/A
|_clock-skew: -1s

Nmap scan report for 10.211.11.10
Host is up (0.044s latency).

PORT    STATE SERVICE      VERSION
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-05-22 07:32:41Z)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: tryhackme.loc, Site: Default-First-Site-Name)
445/tcp open  microsoft-ds Windows Server 2019 Datacenter 17763 microsoft-ds (workgroup: TRYHACKME)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2019 Datacenter 17763 (Windows Server 2019 Datacenter 6.3)
|   Computer name: DC
|   NetBIOS computer name: DC\x00
|   Domain name: tryhackme.loc
|   Forest name: tryhackme.loc
|   FQDN: DC.tryhackme.loc
|_  System time: 2026-05-22T07:32:53+00:00
| smb2-time: 
|   date: 2026-05-22T07:32:52
|_  start_date: N/A
|_clock-skew: mean: 2s, deviation: 5s, median: 0s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 2 IP addresses (2 hosts up) scanned in 27.65 seconds
```

You can spot the Domain Controller (DC) because it will often have ports 88, 389,and 445 open, with banners like `Windows Server` or even domain names revealed in the output.
You can also scan all ports on the hosts to not miss the non-standard ports.
**Full Ports Scan on Hosts:**
```bash
nmap -sS -p- -T3 -iL hosts.txt -oN full_port_scan.txt
```

`-sS` - TCP SYN scan, which is stealthier than a full connect scan.
`-p-` - Scans all 65535 TCP ports.
`-T3` - Sets the timming template to "normal" to balance speed and stealth.
`-iL hosts.txt` - Inputs the list of live hosts from the previous nmap command.
`-oN full_port_scan.txt` - Outputs the result to a file.

```
└─$ nmap -sS -p- -T3 -iL hosts.txt -oN full_port_scan.txt
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-22 09:40 +0200
Nmap scan report for 10.211.11.20
Host is up (0.054s latency).
Not shown: 65519 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49677/tcp open  unknown
50096/tcp open  unknown

Nmap scan report for 10.211.11.10
Host is up (0.047s latency).
Not shown: 65507 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
3389/tcp  open  ms-wbt-server
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49670/tcp open  unknown
49672/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49677/tcp open  unknown
49697/tcp open  unknown
49770/tcp open  unknown

Nmap done: 2 IP addresses (2 hosts up) scanned in 65.80 seconds
```

**Task 3** - Network Enumeration With SMB
SMB - Server Message Block protocol.
We will limit our scan to the following common ports:
* TCP 88 (Kerberos): KErberos uses this port of authentication in the Active Directory. From a penetration testing poing of view, it can be a goldmine for a ticket attacks like Pass-the-Tichek and Kerberoasting.
* TCP 135 (RPC Endpoint Mapper): Thic TCP port is used for Remote Procedure Calls (RPC). It might be leveraged to identify services for lateral movement or remote code execution via DCOM.
* TCP 139 (NetBIOS Sessio Service): This port is used for file sharing in older Windows systems It can be abused for null sessions and information gathering.
* TCP 389 (LDAP): This TCP poert is used by the Lightweight Directory Access Protocol (LDAP). It is in plaintext and can be a prime target for enuerating AD objects, users, and policies.
* TCP 445 (SMB): Critical for file sharing and remote admin; abused for exploits like EternalBlue, SMB raley attacks and credentials theft.
* TCP 636 (LDAPS): This port is used by Secure LDAP. ALthough it is encrypted, it can still expose AD structure if miscofigured and can be abused via certificate-based attacks like AD CS exploitation.

**Default Nmap Host scan:**
```bash
nmap -p 88,135,139,389,445,636 -sV -sC TARGET_IP
```

`-sV` - attempt to detect service version
`-sC` - allow default scritps to run

```
└─$ nmap -p 88,135,139,445,636 -sV -sC 10.211.11.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-27 18:42 +0200
Nmap scan report for 10.211.11.10
Host is up (0.081s latency).

PORT    STATE SERVICE      VERSION
88/tcp  open  kerberos-sec Microsoft Windows Kerberos (server time: 2026-05-27 16:42:28Z)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2019 Datacenter 17763 microsoft-ds (workgroup: TRYHACKME)
636/tcp open  tcpwrapped
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2019 Datacenter 17763 (Windows Server 2019 Datacenter 6.3)
|   Computer name: DC
|   NetBIOS computer name: DC\x00
|   Domain name: tryhackme.loc
|   Forest name: tryhackme.loc
|   FQDN: DC.tryhackme.loc
|_  System time: 2026-05-27T16:42:32+00:00
|_clock-skew: mean: 1s, deviation: 2s, median: 0s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2026-05-27T16:42:33
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.82 seconds
```

On this host the SMB port is open so we can try to get credentials from it by anonymous connection. To maintain it we will use tolls like `smbclient` and `smbmap`.
`smbclient` - command-line tool that allows interaction with SMB shares. It is similar to FTP client. It allows to upload, download, and borwsere files on a remote SMB server. You have to login to the server to use it, sometimes, admin configures SMB server, with a null/anonymous session, so you can use smblcient without passing login credentials.
**Connecting with SMB via smbclient:**
```bash
smblcient -L //TARGET_IP -N
```

`-L` - gives list of shares avaiable on host
`-N` - don't ask for password

```
└─$ smbclient -L //10.211.11.10 -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        AnonShare       Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        SharedFiles     Disk      
        SYSVOL          Disk      Logon server share 
        UserBackups     Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.211.11.10 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

There we can see list of shares avaiable on anonymous session on current host.
`smbmap` - roconnaissance tool to enumerates SMB shares across a host. Is allows to display, read, and write permission for each share. It quiclky identify accessible or misconfigured shares without manually connecting to ech one.

**Enumerating shares on SMB with smbmap:**
```bash
smbmap -H TARGET_IP
```

`-H` - Host IP or FQDB (Fully Qualified Domain Name)

```
└─$ smbmap -H 10.211.11.10
[+] IP: 10.211.11.10:445        Name: 10.211.11.10              Status: Authenticated
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        AnonShare                                               READ, WRITE
        C$                                                      NO ACCESS       Default share
        IPC$                                                    NO ACCESS       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        SharedFiles                                             READ, WRITE
        SYSVOL                                                  NO ACCESS       Logon server share 
        UserBackups                                             READ, WRITE
```

This gives us list of shares on current host and tells current (anonymous in this case) user permissions.
Besides standard shares on this hosts there are three non-standard shares: `AnonShares`, `SharedFiles`, and `UserBackups`.
This enumeration could be done with Nmap too. You have to use Nmap's `smb-enum-shares` script.
**Enumerating shares on SMB via Nmap:**
```bash
nmap -p 445 --script smb-enum-shares TARGET IP
```

`--script` - enables scritp usage
`smb-enum-shares` - script name, this script enumerate shares with READ/WRITE, READ, or no access permissions.

```
└─$ nmap -p445 --script smb-enum-shares 10.211.11.10
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-27 19:18 +0200
Nmap scan report for 10.211.11.10
Host is up (0.059s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: <blank>
|   \\10.211.11.10\ADMIN$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Remote Admin
|     Anonymous access: <none>
|   \\10.211.11.10\AnonShare: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: READ/WRITE
|   \\10.211.11.10\C$: 
|     Type: STYPE_DISKTREE_HIDDEN
|     Comment: Default share
|     Anonymous access: <none>
|   \\10.211.11.10\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: Remote IPC
|     Anonymous access: READ/WRITE
|   \\10.211.11.10\NETLOGON: 
|     Type: STYPE_DISKTREE
|     Comment: Logon server share 
|     Anonymous access: READ
|   \\10.211.11.10\SYSVOL: 
|     Type: STYPE_DISKTREE
|     Comment: Logon server share 
|     Anonymous access: READ
|   \\10.211.11.10\SharedFiles: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Anonymous access: READ/WRITE
|   \\10.211.11.10\UserBackups: 
|     Type: STYPE_DISKTREE
|     Comment: 
|_    Anonymous access: READ/WRITE

Nmap done: 1 IP address (1 host up) scanned in 74.68 seconds
```

This gives us also list of the same shares as previous tools.
To connect with share you can use `smblcient`.
**Connecting with avaiable share via smblcient:**
```bash
smbclient //TARGET_IP/SHARE_NAME -N
```

```
└─$ smbclient //10.211.11.10/SharedFiles -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed May 27 19:27:05 2026
  ..                                  D        0  Wed May 27 19:27:05 2026
  Mouse_and_Malware.txt               A     1141  Thu May 15 11:40:19 2025

                7863807 blocks of size 4096. 3478341 blocks available
smb: \> get Mouse_and_Malware.txt 
getting file \Mouse_and_Malware.txt of size 1141 as Mouse_and_Malware.txt (5.0 KiloBytes/sec) (average 5.0 KiloBytes/sec)
smb: \> exit
```

There I connected with share as an anonymous user, listed shares with `ls` command and downloaded txt file with `get file_name` command. Now I can read this file, because it is on my Linux.
Because of no login credentials we have to use `-N` flag to log as an anonymous user. If you have a username or password to SMB user, you can specify them with `user=USERNAME`, `--password=PASSWORD` or `-U 'username%password'`. For domain accounts, you need to specify domain using `-W`.
SMB shares could contian configuration files, backup files, scripts, and even documents, so it is crucial to look there as a penetration tester. Some of this files might contain usernames or even credentials with a password that never got changed.

Another tool and rescources to enumerate SMB are:
* `impacket-smblcient` - Python based implementation of smbclient, which is avaiable form the Impacket toolkit.
* `CrackMapExec` - it's used for post-exploitation and enumeration. Includes many SMB modules for listing shares, testing credentials, and many others.
* `enum4linux`, `enum4linux-ng` - performs extensive enumeration over SMB.

Flag to this task was hidden in the `UserBackup` share so I had to connect with it and download file `flag.txt`.

**Task 4** - Domain Enumeration

In domain enumeration we will focus on enumerating users, we will be looking for usernames. Them could be gather through various unauthenticated methods, each reying on different misconfigurations.
First method is `LDAP Enumerationt (Anonymous Bind)`.
LDAP - Lightweight Directory Access Protocol. It's used for acessing and managing directory services. It's helpt locate and organise resources within a network such a: users, groups, devices and ograniasational information. Some LDAP servers alllow anonymous users to perform read-only queries. This can expose user accounts and other directory information.
To perform this attack we will use `ldapsearch` tool, which is command line tool.
**Testing if anonymous LDAP bind is enabled:**
```bash
ldapsearch -x -H ldap://TARGET_IP -s base
```

`-x` - Simplte authentication like anonymous authentication.
`-H` - Specifies the LDAP server.
`-s` - Limits uety only to base object and does not search subtrees or sholdre.

If anonymous session is enabled, this should give us output like that:
```
└─$ ldapsearch -x -H ldap://10.211.11.10 -s base
# extended LDIF
#
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: ALL
#

#
dn:
domainFunctionality: 6
forestFunctionality: 6
domainControllerFunctionality: 7
rootDomainNamingContext: DC=tryhackme,DC=loc
ldapServiceName: tryhackme.loc:dc$@TRYHACKME.LOC
isGlobalCatalogReady: TRUE
dsServiceName: CN=NTDS Settings,CN=DC,CN=Servers,CN=Default-First-Site-Name,CN
 =Sites,CN=Configuration,DC=tryhackme,DC=loc
dnsHostName: DC.tryhackme.loc
defaultNamingContext: DC=tryhackme,DC=loc
currentTime: 20260528165358.0Z
configurationNamingContext: CN=Configuration,DC=tryhackme,DC=loc

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```

Then we can enumerate users with this command:
**User eunumeration with ldapsearch:**
```bash
ldapsearch -x -H ldap://TARGET_IP -b "DEFAULT_NAMING_CONTEXT" "(objectClass=person)"
```

`-b` - specifies domain name.

```
└─$ ldapsearch -x -H ldap://10.211.11.10 -b "dc=tryhackme,dc=loc" "(objectClass=person)"
# extended LDIF
#
# LDAPv3
# base <dc=tryhackme,dc=loc> with scope subtree
# filter: (objectClass=person)
# requesting: ALL
#

# Guest, Users, tryhackme.loc
dn: CN=Guest,CN=Users,DC=tryhackme,DC=loc
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Guest
description: Built-in account for guest access to the computer/domain
distinguishedName: CN=Guest,CN=Users,DC=tryhackme,DC=loc
instanceType: 4
whenCreated: 20250402151315.0Z
whenChanged: 20260527212636.0Z
uSNCreated: 8197
memberOf: CN=Guests,CN=Builtin,DC=tryhackme,DC=loc
uSNChanged: 123241
name: Guest
objectGUID:: JhTYTKXnbUC5hNj+7ly0bg==
userAccountControl: 66082
badPwdCount: 4
codePage: 0
countryCode: 0
badPasswordTime: 134244443881982157
lastLogoff: 0
lastLogon: 0
pwdLastSet: 0
primaryGroupID: 514
objectSid:: AQUAAAAAAAUVAAAAKeA2dTgJ371Q0KEA9QEAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: Guest
sAMAccountType: 805306368
lockoutTime: 134243907962226037
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=tryhackme,DC=loc
isCriticalSystemObject: TRUE
dSCorePropagationData: 20250514163924.0Z
dSCorePropagationData: 20250402151422.0Z
dSCorePropagationData: 16010101000417.0Z
```

This is only output from one user, this command gives us every data about every user on this domain.

Second option is `Enum4linux-ng`.
`enum4linux-ng` is a tool that automates various enumeration against Windows system, includes user enumeration. It relies on SMB and RPC protocols. It can give such informations as: user lists, group memberships, and share details.
**Enumerating via enum4linux-ng:**
```bash
enum4linux-ng -A TARGET_IP > results.txt
```

`-A` - Performs all avaiable enumeration functions (users, groups, shares, password policy, RID cycling, OS information and NetBIOS information).
This command saved output to `results.txt` file. There are a lot of information gather about the AD, like usernames or groups.

Third option is `RPC Enumeration (Null Sessions)`.

MSRPC - Microsoft Remote Procedure Call protocol that enables a program running on one computer to request services from a program on another comuputer, without understanding details of the network. This services could be accessed over the SMB protocol. When SMB is misconfigured, that it allow null session that doesn't require authentication, then and unauthenticated user can connect to the IPC$ share and eumumerate domain.
**Veryfiying null session with rpcclient:**
```bash
rpcclient -U "" TARGET_NAME -N
```

`-U` - Used to specify the username, `""` is used for anonymous login.
`-N` - Tells RPC not to prompt for a password.

```
└─$ rpcclient -U "" 10.211.11.10 -N
rpcclient $> 
```

If it connected successfully we can enumerate with: `enumdomusers`.
```
└─$ rpcclient -U "" 10.211.11.10 -N
rpcclient $> enumdomusers 
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[sshd] rid:[0x649]
user:[gerald.burgess] rid:[0x650]
user:[nigel.parsons] rid:[0x651]
user:[guy.smith] rid:[0x652]
user:[jeremy.booth] rid:[0x653]
user:[barbara.jones] rid:[0x654]
user:[marion.kay] rid:[0x655]
user:[kathryn.williams] rid:[0x656]
user:[danny.baker] rid:[0x657]
user:[gary.clarke] rid:[0x658]
user:[daniel.turner] rid:[0x659]
user:[debra.yates] rid:[0x65a]
user:[jeffrey.thompson] rid:[0x65b]
user:[martin.riley] rid:[0x65c]
user:[danielle.lee] rid:[0x65d]
user:[douglas.roberts] rid:[0x65e]
user:[dawn.bolton] rid:[0x65f]
user:[danielle.ali] rid:[0x660]
user:[michelle.palmer] rid:[0x661]
user:[katie.thomas] rid:[0x662]
user:[jennifer.harding] rid:[0x663]
user:[strategos] rid:[0x664]
user:[empanadal0v3r] rid:[0x665]
user:[drgonz0] rid:[0x666]
user:[strate905] rid:[0x667]
user:[krbtgtsvc] rid:[0x668]
user:[asrepuser1] rid:[0x669]
user:[rduke] rid:[0xa31]
user:[user] rid:[0x1201]
rpcclient $> 
```

This gives us only list of users with their rid.

Fourht option is `RID Cycling`.
RID - Relative Identifier, it's used to identify users and roup objects. RIDs are components of the Security Identifier (SID), which identifies each object with a domian. Some of the RIDs are well-known and stadardises:
- 500 is the Administrator account
- 501 is the Guest account
- 512-514 ore for the following groups: Domain Admins, Domain users, Domain guests.
- over 1000 typcally start user accounts.
If `enumdomusers` is restricted, we can try manually querying each individual user RID with this bash command:
**Manually querying users RID:**
```bash
for i in $(seq 500 2000); do echo "queryuser $i" |rpcclient -U "" -N TARGET_IP 2>/dev/null | grep -i "User Name"; done
```

`for i in $(seq 500 2000)` - First run a loop to iterate through a range of possible RIDs to identify valid user accounts.
`echo "queryuser $1"` - Queries information about the user associated with RID $i.
`2>/dev/null` - Redirects any error mesages to /dev/null, efectively silencing them.
`|grep -i "User Name"` - Filters the output to display lines containing "user Name", ingoring case sansitivity (-i).
This command could take 2-3 minutes to complete.
```
└─$ for i in $(seq 500 2000); do echo "queryuser $i" |rpcclient -U "" -N 10.211.11.10 2>/dev/null | grep -i "User Name"; done
        User Name   :   Administrator
        User Name   :   Guest
        User Name   :   krbtgt
        User Name   :   DC$
        User Name   :   WRK$
        User Name   :   sshd
        User Name   :   gerald.burgess
        User Name   :   nigel.parsons
        User Name   :   guy.smith
        User Name   :   jeremy.booth
        User Name   :   barbara.jones
        User Name   :   marion.kay
        User Name   :   kathryn.williams
        User Name   :   danny.baker
        User Name   :   gary.clarke
        User Name   :   daniel.turner
        User Name   :   debra.yates
        User Name   :   jeffrey.thompson
        User Name   :   martin.riley
        User Name   :   danielle.lee
        User Name   :   douglas.roberts
        User Name   :   dawn.bolton
        User Name   :   danielle.ali
        User Name   :   michelle.palmer
        User Name   :   katie.thomas
        User Name   :   jennifer.harding
        User Name   :   strategos
        User Name   :   empanadal0v3r
        User Name   :   drgonz0
        User Name   :   strate905
        User Name   :   krbtgtsvc
        User Name   :   asrepuser1
```

Fifth option is `User Enumeration With Kerbrute`.
`Kerberos` is the primary authentication protocol for Microsoft Windows doamains. `Kerberos` uses a ticket-based system managed by trusted third party, the `Key Distribution Centre (KDC)`. It works differently from `NTLM`, wchich relies on challange-response mechanism. `Kerberos` is more resilient against attack due to stronger encryption methids, and it enables mutual authentication between client and server. `Kerbrute` is popular enumeration tool to enumerate valid Active Directory users by abusing the Kerberos pre-authentication, it relies on brute-force mechanism. Unlike `enum4linux-ng` or `rpcclient` which could return: disabled accounts, non-domain accounts, fake honeypot users, false positives, `kerbrute` confirm which ones are real, active AD users, which allows us to target them more accurately with password sprays.
From previous enumeration I made user list: `user.txt`.
**Enumerating users with kerbrute:**
```bash
kerbrute userenm --dc TARGET_IP -d DOMAIN_NAME users.txt
```

`--dc` - Specifies TARGET_IP.
`-d` - Specifies domain name (e.g., tryhackme.loc).
`userneum` - User enumeration mode.

```
└─$ kerbrute userenum --dc 10.211.11.10 -d tryhackme.loc users.txt 

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 05/29/26 - Ronnie Flathers @ropnop

2026/05/29 12:29:12 >  Using KDC(s):
2026/05/29 12:29:12 >   10.211.11.10:88

2026/05/29 12:29:12 >  [+] VALID USERNAME:       DC$@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       Administrator@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       sshd@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       WRK$@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       jeremy.booth@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       guy.smith@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       nigel.parsons@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       gerald.burgess@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       marion.kay@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       danny.baker@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       kathryn.williams@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       barbara.jones@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       gary.clarke@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       debra.yates@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       jeffrey.thompson@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       danielle.lee@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       daniel.turner@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       martin.riley@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       douglas.roberts@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       danielle.ali@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       jennifer.harding@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       strategos@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       drgonz0@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       empanadal0v3r@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       krbtgtsvc@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       strate905@tryhackme.loc
2026/05/29 12:29:12 >  [+] VALID USERNAME:       asrepuser1@tryhackme.loc
2026/05/29 12:29:12 >  Done! Tested 32 usernames (27 valid) in 0.259 seconds
```

This tests every user, and gives us valid usernames on AD with their addresses. Now we can update our user list with this information.
If any other tool doesn't work and we had to use only `kerbrute` we can use huge list of common usernmes located: `/usr/share/seclists/Usernames/Names/names.txt`. 

**Task 5** - Password Spraying.
Password Spraying - attack technique, where small set of common passowrds are tested across many accounts. It avoids account lockouts, opposite to traditional brute-force attack, because here we only take a few attempts on each accounts. It's effective, because still a lot of corporation accounts had poor password policies. Common password lists for spraying include: seasonal passwords, default passwords uysed by IT teams (e.g., `Password123`), passwords leaked in data breaches like `rockyou.txt`.
Before setting an attack we need to undestand target's password policy. It will help us to retrive information about the minimum password length, complexity, and the number of failed attempts that will lock out an account. To gather this infomrations we could use `rpcclient`. Connect with the rpc and then use `getdompwinfo` command:
```
└─$ rpcclient -U "" 10.211.11.10 -N
rpcclient $> getdompwinfo
min_password_length: 7
password_properties: 0x00000001
        DOMAIN_PASSWORD_COMPLEX
rpcclient $> 
```

Another way to get the policy is to use `crackmapexec` tool.
`CrackMapExec` is network service exploitation tool. It allows us to perform enumeration, commmand execution, and post-exploitation attacks in Windows enviorments. It works on many network protocols: SMB, LDAP, RDP, and SSH. If anonymous access is permitted, we can retrive the password policy without credentials.
**Rertiving password policy via CrackMapeExec:**
```bash
crackmapexec smb 10.211.11.10 --pass-pol
```

`smb` - Specifies the protocol to work on.
`--pass-pol` - Specifies to get password policy.

```
┌──(kali㉿kali)-[~/work/machines/AD:_Basic_Enumeration]
└─$ crackmapexec smb 10.211.11.10 --pass-pol
[*] First time use detected
[*] Creating home directory structure
[*] Creating default workspace
[*] Initializing SMB protocol database
[*] Initializing WINRM protocol database
[*] Initializing LDAP protocol database
[*] Initializing MSSQL protocol database
[*] Initializing SSH protocol database
[*] Initializing FTP protocol database
[*] Initializing RDP protocol database
[*] Copying default configuration file
[*] Generating SSL certificate
SMB         10.211.11.10    445    DC               [*] Windows Server 2019 Datacenter 17763 x64 (name:DC) (domain:tryhackme.loc) (signing:True) (SMBv1:True)
SMB         10.211.11.10    445    DC               [+] Dumping password info for domain: TRYHACKME
SMB         10.211.11.10    445    DC               Minimum password length: 7
SMB         10.211.11.10    445    DC               Password history length: 24
SMB         10.211.11.10    445    DC               Maximum password age: 41 days 23 hours 53 minutes 
SMB         10.211.11.10    445    DC               
SMB         10.211.11.10    445    DC               Password Complexity Flags: 000001
SMB         10.211.11.10    445    DC                   Domain Refuse Password Change: 0
SMB         10.211.11.10    445    DC                   Domain Password Store Cleartext: 0
SMB         10.211.11.10    445    DC                   Domain Password Lockout Admins: 0
SMB         10.211.11.10    445    DC                   Domain Password No Clear Change: 0
SMB         10.211.11.10    445    DC                   Domain Password No Anon Change: 0
SMB         10.211.11.10    445    DC                   Domain Password Complex: 1
SMB         10.211.11.10    445    DC               
SMB         10.211.11.10    445    DC               Minimum password age: 1 day 4 minutes 
SMB         10.211.11.10    445    DC               Reset Account Lockout Counter: 2 minutes 
SMB         10.211.11.10    445    DC               Locked Account Duration: 2 minutes 
SMB         10.211.11.10    445    DC               Account Lockout Threshold: 10
SMB         10.211.11.10    445    DC               Forced Log off Time: Not Set
```

This gives us whole password policy on current AD server.
`password_properties: 0x00000001` and `Password Complexity Flags: 000001` mean that passwords to this domain must respect fhree of four conditions: 1. Uppercase letters, 2. Lowercase letters, 3. Digits, 4. Special characters.
In this scenario we know that passwords are varations of strign "Password". We can create list, making sure to respect password policy. Our current list could look like this:
```
└─$ cat passwords.txt                                
Password!
Password1
Password1!
P@ssword
Pa55word1
```

Now we can use `CrackMapExec` to run our pasword spraying attack against the WRK computer.
**Running password spraying via CracMapExec:**
```bash
crackmapexec smb TARGET_IP -u USER_LIST -p PASSWORD_LIST
```

`-u` - Gives usernames list.
`-p` - Gives passwords list.

```
┌──(kali㉿kali)-[~/work/machines/AD:_Basic_Enumeration]
└─$ crackmapexec smb 10.211.11.10 -u users.txt -p passwords.txt                                                              
SMB         10.211.11.10    445    DC               [*] Windows Server 2019 Datacenter 17763 x64 (name:DC) (domain:tryhackme.loc) (signing:True) (SMBv1:True)
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Administrator:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Administrator:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Administrator:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Administrator:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Administrator:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Guest:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Guest:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Guest:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Guest:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\Guest:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgt:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgt:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgt:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgt:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgt:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\DC$:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\DC$:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\DC$:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\DC$:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\DC$:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\WRK$:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\WRK$:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\WRK$:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\WRK$:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\WRK$:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\sshd:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\sshd:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\sshd:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\sshd:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\sshd:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gerald.burgess:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gerald.burgess:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gerald.burgess:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gerald.burgess:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gerald.burgess:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\nigel.parsons:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\nigel.parsons:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\nigel.parsons:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\nigel.parsons:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\nigel.parsons:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\guy.smith:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\guy.smith:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\guy.smith:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\guy.smith:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\guy.smith:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeremy.booth:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeremy.booth:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeremy.booth:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeremy.booth:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeremy.booth:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\barbara.jones:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\barbara.jones:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\barbara.jones:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\barbara.jones:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\barbara.jones:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\marion.kay:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\marion.kay:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\marion.kay:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\marion.kay:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\marion.kay:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\kathryn.williams:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\kathryn.williams:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\kathryn.williams:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\kathryn.williams:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\kathryn.williams:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danny.baker:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danny.baker:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danny.baker:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danny.baker:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danny.baker:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gary.clarke:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gary.clarke:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gary.clarke:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gary.clarke:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\gary.clarke:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\daniel.turner:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\daniel.turner:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\daniel.turner:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\daniel.turner:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\daniel.turner:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\debra.yates:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\debra.yates:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\debra.yates:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\debra.yates:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\debra.yates:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeffrey.thompson:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeffrey.thompson:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeffrey.thompson:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeffrey.thompson:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jeffrey.thompson:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\martin.riley:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\martin.riley:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\martin.riley:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\martin.riley:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\martin.riley:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.lee:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.lee:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.lee:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.lee:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.lee:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\douglas.roberts:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\douglas.roberts:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\douglas.roberts:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\douglas.roberts:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\douglas.roberts:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\dawn.bolton:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\dawn.bolton:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\dawn.bolton:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\dawn.bolton:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\dawn.bolton:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.ali:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.ali:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.ali:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.ali:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\danielle.ali:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\michelle.palmer:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\michelle.palmer:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\michelle.palmer:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\michelle.palmer:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\michelle.palmer:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\katie.thomas:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\katie.thomas:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\katie.thomas:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\katie.thomas:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\katie.thomas:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jennifer.harding:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jennifer.harding:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jennifer.harding:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jennifer.harding:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\jennifer.harding:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strategos:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strategos:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strategos:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strategos:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strategos:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\empanadal0v3r:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\empanadal0v3r:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\empanadal0v3r:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\empanadal0v3r:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\empanadal0v3r:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\drgonz0:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\drgonz0:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\drgonz0:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\drgonz0:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\drgonz0:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strate905:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strate905:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strate905:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strate905:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\strate905:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgtsvc:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgtsvc:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgtsvc:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgtsvc:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\krbtgtsvc:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\asrepuser1:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\asrepuser1:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\asrepuser1:Password1! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\asrepuser1:P@ssword STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\asrepuser1:Pa55word1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\rduke:Password! STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [-] tryhackme.loc\rduke:Password1 STATUS_LOGON_FAILURE 
SMB         10.211.11.10    445    DC               [+] tryhackme.loc\rduke:Password1!
```

[+] in the last line mean that we found login credentials to one user.
