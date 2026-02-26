# Target: [Machine Name]
**Platform:** [HackTheBox / TryHackMe / Other]
[Machine Name on Platform](Link to the machine)

| Target Name | IP Address | OS | Difficulty |
| :--- | :--- | :--- | :--- |
| Name_Here | 10.10.10.x | Linux/Windows | Easy/Medium/Hard |

---

## Phase 1: Reconnaissance & Enumeration

**Nmap Scan:**
```bash
#Paste nmap command and output here
```

**Directory Brute-forcing:**
```bash
#Paste gobuster/dirbuster command and output here
```

## Phase 2: Initial Foothold (Gaining Access)

**Vulnerability Identified:** [e.g., WordPress Plugin X.X SQLi / CVE-202X-XXXX]
**Exploit Used:** [Link to Exploit-DB or GitHub script]
**Payload/Command:**
```bash
#Your exploit command or reverse shell payload goes here
```

**Listener Setup:**
```bash
nc -lvnp 4444
```

**Proof of Access:**
whoami
#output here
cat user.txt
```

## Phase 3: Privilege Escalation & Post-Exploitation

**Local Enumeration:**
```bash
#e.g., uploading/running LinPEAS, WinPEAS, or manual checks like 'sudo -l'
```

**Privilege Escalation Vector:** [e.g., SUID binary, vulnerable cron job, kernel exploit]
**Exploit/Execution:**
```bash
#Commands or scripts used to gain root/administrator
```
**Proof of Root/Admin:**
```bash
whoami
#output here
cat root.txt
```

## Conclusion & Takeaways

**Summary:** [Briefly describe the overall path from scanning to root]
**What I Learned:** [Any new tool, commands, or concepts you picked up]
**Remediation:** [How a system administrator could patch these vulnerabilities]
