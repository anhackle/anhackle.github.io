---
title: "HTB Challenge - OnlyHacks Writeup"
date: 2026-11-17 09:14:00 +0700
categories: [HackTheBox, Challenges]
tags: [easy]
---
## P0 & P1
First of all, try to access to the URL of the challenge. We face with a dating and matching website:

![Upgrade_Notice.pdf](assets/image/htb-challenges/onlyhacks/2026-11-17_09-15.png){: w="700" }

Nothing really cool right now so I decide to register an account to check more deeply other functions.

![Upgrade_Notice.pdf](assets/image/htb-challenges/onlyhacks/2026-11-17_09-16.png){: w="700" }

After successfully registering, I start surfing the application by choosing and luckily matching with a strange girl here :)

![Upgrade_Notice.pdf](assets/image/htb-challenges/onlyhacks/2026-11-17_09-17.png){: w="700" }

Watch all requests on `burpsuite`, I detect that there is a very interesting request like below:

![Upgrade_Notice.pdf](assets/image/htb-challenges/onlyhacks/2026-11-17_09-17_1.png){: w="700" }

## P2
What do we usually think to immediately when we see a request with an id like this ? Yes, it's a `IDOR` exploitation. So I fastly push that request to `Repeater` and try to change the value of `rid` parameter. Fortunately, I find out `rid=3` return the flag:

![Upgrade_Notice.pdf](assets/image/htb-challenges/onlyhacks/2026-11-17_09-17_2.png){: w="700" }


