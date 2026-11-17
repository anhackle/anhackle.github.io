---
title: "HTB Challenge - Armaxis Writeup"
date: 2026-11-17 09:14:00 +0700
categories: [HackTheBox, Challenges]
tags: [easy]
---
## P0 & P1
First of all, try to access to the URL of the challenge. We face with a website like this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-22.png){: w="700" }

Next, I try to register a new account with `test@gmail.com` and access to inner functions but nothing here:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-22_1.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-23.png){: w="700" }

In the reality, when pentesting for a website with an account which is registered by ourselves but no function enabled, we usually think to "This created user doesn't have any valid permissions to access". To verify that my speculations were correct, I tried reading the source code provided by the challenge.<br>

In `challenge/routes/index.js`, we found a path `/weapon/dispatch` which require the role `admin`:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-32.png){: w="700" }

## P2
So our mission right now is we must find a way either upgrading our current user to role `admin` or compromising the real `admin` account. I try to compromise the `JWT token` but all exploitations fail.

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-25.png){: w="700" }

So I try to dive more deeply in the source code and I find out an admin email in `challenge/database.js` file. It is `admin@armaxis.htb`:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-27.png){: w="700" }

In addtion to, I also note that, this website has a fucntion `reset password` which allow user to change they own password. Everytime users need to change the password, backend will send an `password reset token` to their email. By using that token to verify, users can change their password easily.

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-29_1.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-47.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-28.png){: w="700" }

But backend has a big problem. It does not verify the owner of that token. We can see it clearly in the source code below:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-29.png){: w="700" }

This means normal users can use their own `password reset token` to verify and reset password for admin users. From here we can easily occupy the admin account:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-53.png){: w="700" }

Now we can login the website with `admin@armaxis.htb` account and successfully access the `dispatch weapon` function:

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-54.png){: w="700" }

We continue to more deeply dive in the source code, especially is the code which is responsible for processing `dispatch weapon` function. I detect that it is using a function name `parseMarkdown` which receive `note` as a input.

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_10-55.png){: w="700" }

Fastly checking `challenge/markdown.js` file, I realize that this function uses this regex `/\!\[.*?\]\((.*?)\)/g` to match Markdown image syntax `![alt text](image URL)` and captures the URL portion (the first capture group). After find out the image URL, it will put it as a input for `execSync` function. So now, maybe all of you know what to do right now, `command injection` by **abusing url of a image after image syntax in Markdown**.

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_11-02.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_11-03.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/armaxis/2026-11-17_11-03_1.png){: w="700" }

Base64 decode that string, we find out the flag:
```
└─$ echo "SFRCe200cmtkMHduX2J1Z3NfMW5fdGgzX3cxbGQhfQo=" | base64 -d
HTB{m4rkd0wn_bugs_1n_th3_w1ld!}
```
