# Looking Glass
  ![](Attachment/Banner.png)

**Room:** [Looking Glass](https://tryhackme.com/room/lookingglass)  
**Difficulty:** Medium  
**Target:** Linux

---

## 1. Reconnaissance

### Port Scanning

I started with a full TCP port scan to identify exposed services on the target machine.

```
nmap -A -vv <IP> -p- -sV -oN nmap
```

The scan revealed an unusually large number of open TCP ports, including SSH services. I performed additional service detection against individual ports to identify the available SSH implementations.

```
nmap <IP> -p13883 -sV
```

**Result:**

```
PORT      STATE SERVICE VERSION
13883/tcp open  ssh     Dropbear sshd (protocol 2.0)
```

I also examined the standard SSH port.

```
nmap <IP> -p22 -sV
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
```

The target exposed multiple SSH endpoints, making further investigation necessary to identify the intended service.

## 2. Identifying the Correct SSH Port

The challenge required finding the correct SSH port using directional hints. Connecting to a port returned either `Higher` or `Lower`, allowing the search range to be narrowed.

```
ssh -p 9745 -o HostKeyAlgorithms=+ssh-rsa <IP>
```

```
Lower
Connection to <IP> closed.
```

Another attempt returned the opposite hint:

```
ssh -p 12448 -o HostKeyAlgorithms=+ssh-rsa <IP>
```

```
Higher
Connection to <IP> closed.
```

I used these responses to narrow down the candidate port range. A binary-search approach is suitable for this challenge, provided the search bounds are updated correctly.

```python
min_port = int(input("Enter min: "))
max_port = int(input("Enter max: "))

while min_port <= max_port:
	mid = (max_port + min_port) // 2
	print(f"ssh -p {mid} -o HostKeyAlgorithms=+ssh-rsa 10.48.151.117")
	print("H = Higher and L = Lower")
	choice = input("Result: ").strip().upper()
	
	if choice == "H":
		max_port = mid + 1
	elif choice == "L":
		min_port = mid - 1
	else:
		print("Invalid choice. Enter H or L.")
```

After identifying the correct endpoint, I connected to port `13039`. -> Port changes after reset

```
ssh -p 13039 -o HostKeyAlgorithms=+ssh-rsa <IP>
```

The service returned a cryptographic challenge.

## 3. Solving the Cipher

The SSH service presented an encrypted poem titled _Jabberwocky_ and requested a secret.
  ![](Attachment/cipher.png)

I identified the ciphertext as a Vigenère cipher and used [Boxentriq's Vigenère Cipher tool](https://www.boxentriq.com/ciphers/vigenere-cipher) to analyze it.

**Cipher key:**

```
thealphabetcipher
```

After decrypting the ciphertext, I obtained the secret:

```
bewareTheJabberwock
```

This secret was accepted by the challenge, allowing me to proceed with SSH authentication.

![](Attachment/ssh_cred.png)
## 4. Initial Access

Using the recovered secret, I obtained the SSH credentials:

```
Username: jabberwock
Password: <****>
```

I used these credentials to access the target machine and began enumerating the environment.

After logging in, I located the user flag.

**User flag:** Obtained.

![](Attachment/initial.png)
## 5. Privilege Escalation Enumeration

After gaining initial access, I enumerated the system to identify potential privilege escalation paths.

### Sudo Permissions

I checked whether the `jabberwock` user could execute commands with elevated privileges.

```
sudo -l
```

The output confirmed that the user was not permitted to execute sudo commands.

```
Sorry, user jabberwock may not run sudo on ip-10-48-137-22.
```

I also attempted to reboot the machine using sudo:

```
sudo /usr/sbin/reboot
```

The command was denied because the account did not have the required sudo privileges.

### Scheduled Script Investigation

Further investigation revealed a scheduled startup task involving the script `twasBrillig.sh`.

![](Attachment/cronjob.png)
  

The relevant configuration was:

```
@reboot tweedledum bash /home/jabberwock/twasBrillig.sh
```

This indicates that the script is executed at system startup under the `tweedledum` account. The script and its permissions are therefore important areas to investigate for the intended privilege escalation path.

The intended approach is to investigate whether `jabberwock` can modify `/home/jabberwock/twasBrillig.sh` and determine the privileges under which the scheduled command executes. However, the available output does not establish the script's permissions or confirm that a privilege escalation has succeeded.

## 6. Current Status

- **Initial access:** Successful.
    
- **User flag:** Obtained.
    
- **Sudo privileges:** Not available to `jabberwock`.
    
- **Privilege escalation:** Investigated the startup script; escalation not completed.
    
- **Root flag:** Not obtained.
    

---
