---
title: "HTB Challenge - Saturn Writeup"
date: 2026-11-23 09:57:00 +0700
categories: [HackTheBox, Challenges]
tags: [easy]
---
## P0 & P1
First of all, use firefox browser to access the URL challenge:

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-48.png){: w="700" }

I try to type `https://google.com`, then it will load the content of it and present on the frontend like this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-58.png){: w="700" }

It looks interesting. With situations like these, I usually think to `Server side Request Forgery (SSRF)` because it allows user input a url and then backend will load it and present directly on the frontend.<br>

I try to review the source code, and detect a block of code like this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-56.png){: w="700" }

Everything now is more clear. The path `secret` will receive a request and check that whether the IP address of the client is equal to `127.0.0.1`. If true it will return the content of `flag.txt` file. So our mission right now is exploiting `SSRF` with the repesctive URL `http://127.0.0.1:1337/secret`.

## P2
As you know, backend now is filter `127.0.0.1` address by using `safeurl` package so I have tried all the payload which I can find from [PayloadAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Request%20Forgery) which can help me bypass the filter and force backend accept `127.0.0.1` but none of them work at this moment.

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-50.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-50_1.png){: w="700" }

 So I decide changing to other strategy. I will craft a server which looks like normal, but when having a request to, it will redirect to `http://127.0.0.1:1337/secret`. To make it public so that backend from HTB challenge can reach, I will use ngrok.

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-53.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-54_1.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-54.png){: w="700" }

Unfortunately, we fail, `safeurl` package is strong enough to filter our exploits :D. Okey, come back with source code to review again.

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_09-55.png){: w="700" }

I realize that, after verifing the url by executing `su.execute(url)` and escaping the `try-catch` block, the backend will make a get request to that url again by running `requests.get(url)`. So I think out other strategy like this:

1. I will craft an own python server and then make it public with grok.
2. When it receives the first request (from `su.execute(url)`) from the challenge backend, it will keep normal and does not redirect anything.
3. When it receives the second request (from `requests.get(url)`), our server will return a redirect to `http://127.0.0.1:1337/secret`.

So first of all, I will take advantage of the old python server above and change it a little bit like this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_10-44.png){: w="700" }

And then, input the public ngrok url again. This time, we succeed:

![Upgrade_Notice.pdf](assets/image/htb-challenges/saturn/2026-11-23_10-45.png){: w="700" }

