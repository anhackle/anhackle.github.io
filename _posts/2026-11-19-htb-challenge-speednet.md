---
title: "HTB Challenge - SpeedNet Writeup"
date: 2026-11-19 15:44:00 +0700
categories: [HackTheBox, Challenges]
tags: [easy]
---
## P0 & P1
First of all, use firefox browser to access the URL challenge:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-45.png){: w="700" }

After registering a new account, backend provides a JWT and redirects  us to `profile` page. To retrieve user information, it uses the query `GetUserProfile`:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-46.png){: w="700" }

Here we can see, this query receives `userId` as a parameter input, I immediately test `IDOR` and fortunately, I find out `admin` information:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-47.png){: w="700" }

Next, for easily interact with graphql api, in burpsuite, I install an extension name `InQL - GraphQL Scanner`.

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-48.png){: w="700" }

Next, I will take advatange of that extension to fastly analyze all `queris` as well as `mutations` which exists:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-49.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-50.png){: w="700" }

First of all, I detect that there is a mutation name `devForgotPassword`. This maybe used for forget password function, so I decide to use it with `admin` account `admin@speednet.htb`

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-51.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-52.png){: w="700" }

Nice it works and returns to me a `Password reset token`. So next we will use `resetPassword` mutation. This mutation receives a token (we have just found above) and a new password:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-52_1.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-54.png){: w="700" }

Great, now we try to login with `admin` account. Unfortunately, this account has turned on `Two-Factor Authentication`:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-56.png){: w="700" }

I try a random OTP for fastly checking the respective request:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_15-57.png){: w="700" }

## P2

I have rechecked again all queries as well as mutations but I can not find any vulnerabilties here. I also use bruteforce method by sending above requests many times, you know, 10000 cases and backend immediately blocks me and return 429 rate limit.<br>

So I comeback with [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/GraphQL%20Injection/README.md#graphql-batching-attacks) and find out this method `graphql-batching-attacks`. Instead of sending too many requests to bruteforce I will use `graphql-batching` which allow me to run many queries or mutations in only one http request:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_20-14.png){: w="700" }

In above image, I try to insert many mutations in one http request. To distinguish between them, I use aliases for example: `a0,a1,a2,etc.`<br>

To avoid encountering error `413 Request Entity Too Large`, we should only insert maximum 500 OTPs in one http request.

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_16-09.png){: w="700" }

After running bruteforce by above method, we achieve access token of `admin` user. Abuse that we can find out the flag easily:

![Upgrade_Notice.pdf](assets/image/htb-challenges/speednet/2026-11-19_16-10.png){: w="700" }
