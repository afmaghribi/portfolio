---
title: "HTB Business CTF 2022 - Trade"
date: 2025-06-23
draft: false
author: "Lychnobyte"
tags:
  - cloud-ctf
  - cloud
  - ctf
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1751579912885/c7c3d172-a44f-41bb-9162-0057813c3701.jpeg"
description: "Solving the 'Trade' challenge from HTB Business CTF 2022, featuring Subversion, AWS SNS, and DynamoDB injection."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/112523817](https://www.pixiv.net/en/artworks/112523817)

Hello all, so i just decide to start blogging again but i don’t know what to write in this blog. So i just rewrite my old blog post into english with some adjustment and revision. :D

Back in 2022 i participated alone in HTB Bussiness CTF just for fun because i curious with their `Cloud` category and it’s only 2 challenge in that category, lol.

Let’s start with first challenge called `Trade`, here the challenge description:

> With increasing breaches there has been equal increased demand for exploits and compromised hosts. Dark APT group has released an online store to sell such digital equipment. Being part of defense operations can you help disrupting their service ?

Anyway before start to solving the challenge as a context, to access the challenge i need to connect vpn that HTB platform provided. That’s why the `AWS` endpoint here kinda different that usual one. I assumed they deploy `AWS-like` services using tools called `localstack` in their own server.

This challenge only give us an `IP address` which service running without any further explanation what kind of service are running. So, my first step is running `Nmap` to scan the `IP address` to know what services are running and in which port.

![Nmap output](https://blogger.googleusercontent.com/img/a/AVvXsEhHbOhc-kinMUUIWVsgMN34ZZ1t85uLTfs9WYr_zn6I91xe_babcRIvhXE_ikd9nMb0jHmcqLxa9D0H5p2D4-jSU0YzAyc2x2J-LsUj_2U8YkpXCGSgyoLeWxhnq0kXZiisQjlrQRnfWasSiytEPs9fYv6MWpXPBWF73Rc_IYBWuhNdLgBiTA0SkECE7A=w640-h260)

As we can see there is 3 ports open with 3 different services running, `ssh server`,`http server` and `subversion`

Let’s dig dive each services running.

### 1. Port 80 - Web server

Open the given `Ip address` in the browser then we got this login page, just usual login page nothing special

![Login page](https://blogger.googleusercontent.com/img/a/AVvXsEgmoZT8m1Mg_huPPBDzECLvRGh30Ahv93VkF4uVGnS72NLqz9McUMOqxhEV2XTAgUhxlzI2fxskFDfHS47Cyj1P7jhcpk7RxlMq7mCRjqIUULOh-qyixLriYL936wb4Bs4-Y9L5SDihtW2GYnW4dLWZ66BXWib8F9zvZBTWMhdNbNSY5GiyuBMgg77FnA=w640-h482)

### 2. Port 3690 - Subversion

Next, access to `subversion` using `svn` client. Fyi, `subversion` is `version control system` just like `git` which is tools that usually use to control our code in `repository`. So, the idea here maybe we can dump the code that running in `http server` to get some `credentials`.

First, we list the repository that available in `subversion` then we `clone` the repository to our local. We found `/store` repository then `clone` to our local. We got 3 files `README.md`, `dynamo.py` and `sns.py`

![SVN clone](https://blogger.googleusercontent.com/img/a/AVvXsEgPqLyLxbbVy0EsQOzaLwESf3QNueA2FMky4YAlLOpBdZ6sjBelwmuqPrEGFDFrRckLyP9ms11omNbskqh7PlUPz-y1jltf1nVp8L_FauI70l_cBBm3uEv9aT9AcX7fBaZaq3yc52wIrfjBQNVIvSm3R8dTUdv6dgt9JLlP4q0xoLLjMZJdmRpZHKJJOw=w640-h222)

In `dynamo.py` file there is hard-code `username` and `password` to store in dynamodb, i assume we can use this credentials to login to login page we found earlier.

![Dynamo.py credentials](https://blogger.googleusercontent.com/img/a/AVvXsEjhPW2NW4VWe8CLBKwsknsEz7UldCJz_cp7iZ8BcKzSxVZSCo1WgNN8lnjSvg0FV21_Lfm-3L99O_BnyVtP81CCMRw53eb889ycqq6jpc11i8DIxw9jP8fG_0n_I9sz-DduUqjUHr2yYAuDEv2dEEKTqGEqNo8CRFoYufasbHy4UeGMFh9LlfamLviZcQ=w600-h640)

So, i tried to login using the credentials i just found it works. But, there is another authentication that ask for our `OTP`.

![OTP page](https://blogger.googleusercontent.com/img/a/AVvXsEiE_ZYz7yk2oCz6vxO-3EJQFB_QD5XFWL_Ji6fkow_0BpSPm_KMy9sDOs6sOUKx-lSL4Z7OxEfl0FlZGkf1ifmlJCgCCL1pI3cERfKkHWlie4aCR_hw1gyxyNWbB5Kn8_Clyic_ZL7sX0lBVVXE8DXEAstC0xW12IJ3DxZJ3oCd_uvwFgfB1jszba78hg=w640-h512)

Well, there is another file called `sns.py` but nothing interesting there. But, since it is stored in `version control system` maybe we can found something interesting in previous version. Check logs of the repository we found there is several modification, then try to revert one by one to check previous code version.

![SVN logs](https://blogger.googleusercontent.com/img/a/AVvXsEiXgU7vOlKKt5rA4O3xJDcZstVgC36J94OhE9IyNXvl043GeOsa8qIrHpDRgSj4oy274KfcqgaVGwFDK3KrBA-D0Ng6YzUl7d_K6CGJpp3ou0VL2eIpMiEl_GhbQGjHgh8iq0ODbVejF6A9ymQSmW6_WgAMdy89qjjcFJkAVEsv1_XP4TA0ChfvvYlywg=w640-h300)

When back to revision 2, there is hard-code AWS access and secret key.

![Revision 2 keys](https://blogger.googleusercontent.com/img/a/AVvXsEg4kTrrKmggUCPutVOESoUgHTSI54rddtH3OZwiKgQ16qjeSj3uHI5eEOOZ5X9JaYLLnEZwDQoTWv86f5RxapGIi_a8JgtCHkAmTBG9q4ONOal6pp8NLwmFt4BFBCinXsErb1ROJYrfLMKN6dVdoDkQ4PO2mhmw6QcNSqbn31p4eWrUUNOxW3enhsUyQA=w640-h428)

So, using those keys we can auth to `AWS-like` services. To setup i just need to add hostname `cloud.htb` with challenge `IP address` in `/etc/hosts` then use `cloud.htb` as our `endpoint-url`. Then try to list what topics that available in our account.

![SNS topics](https://blogger.googleusercontent.com/img/a/AVvXsEik6z-7Xc8fkmmRy_Vh9vkA4b9o400E16guTsJZNfRHeSZn2FzR6e5VYzWWZ5Sbx9ng7N4JnPsyIp_unVmeDwojrtR-wjXe6tAbj_buW0jy3HkqtHX4u_Mx184WqT-2RpHYxCRsgYj-GUhaZAtk_znhYhWiXR-aorRyKFOynodZ6TQGsbzGwuC2yFPbaA=w640-h184)

Well, there is only 1 topic called `otp`. We need to subscribe to `otp` topic so we can continue our login. Since the service are running inside HTB network easiest way to subscribe is using `http` endpoint.

The setup is simple, we just need to open port `80` in our `vpn IP address` in this case i’m using `nc` command so i can see `raw` http request. After setup done, i just subscribe the `otp` topic using command below:

![SNS subscribe](https://blogger.googleusercontent.com/img/a/AVvXsEgWqwLm5EgaoK11jvb8-8g5bmiii4h_Mv4FeX2jgyqcUsAL4yLHbcRvE2nALORkuuxqysJnVj562IRtu4U5KIKdjv5xWFOvKQuqfD-t8218nXG7Jrtt6JW35dciojoztkbWO1JCySoPGF9lQApzdQqPuPmbpGbB4XmfESGMqTAtYN_rgxm7UPEQGQARbA=w640-h76)

Then, try to login again and wait the `otp` hit our `http endopoint`, get the `otp` code and continue our login.

![OTP received](https://blogger.googleusercontent.com/img/a/AVvXsEiQN8TyzbfQEf0-QfkVROBnDpj1sOgf6sWEBK5WmxzBOiYlu344PK_Dtp7IuyxGwois0cAo_P1UYDrEHvBA4y5Y-GP9yqsOJLWS6GMxomB0tziEnXo6xWvyjKmhIZZGxkWsakctA2rSVHmQSbvsDha4teZQnNltaw-Oap7Hp45L7m1aCPbva3YYHxrFuA=w640-h68)

Here the website appearance after we success login.

![Logged in page](https://blogger.googleusercontent.com/img/a/AVvXsEjD5tLQiZEm0kPQ5ki2fHSlYFeuc4dU7airHCa6EQ8J6AY-XfQRz9ZnuSd4aO2vAiBZAK90FLFKfgTX0-4pAZTgJq-WGPfQd1PzgfbNiJCx3rvZnNSfyxUDGBEveLVNLJRosCekKu2bdTXiQ4cLxdXOwfpZBOsGVenQffxmcsjDJHBsetOU1reO307Pqg=w640-h338)

After explore a while, i found `search` page that we can receive some `input` string. After some trial i got an error message that reveal some information while i input `”`.

![DynamoDB error](https://blogger.googleusercontent.com/img/a/AVvXsEj-47BRRmTW1eGtAdHW00ZVCjwTCIiAivTTpUVla1QuA4WSrsqrFcTr80-LyukTZe5Kc3SRmiYAnOe5JcN9t8U45-dyCCOQBbTZ7w_BemP5P7Nhd2bs_-XWDaPRqu1eJYRLcYv9BUaKtDeWqlNdb3kN9Zg3Bo1tRntFDmGKWhnHbFoJdOq6VPKy4ihE3Q=w640-h214)

Look like this input handled directly to `dynamodb`. So, i just googled for maybe some `dynamodb` injection or something to dump some information from db.

Well, found that we can do some injection to dump all data inside `dynamodb` and here the `input` i use to dump all data.

![DynamoDB injection](https://blogger.googleusercontent.com/img/a/AVvXsEiw5PcUhQ2-rzklb0KNUjgxsUk2YxMbb3eugO6_Lyq5hA9OY8mOY-kcPwzELzLN2UAE9BGLNfYRpg9-LjtW4DH8_B0Vdl8lxS63lHG5IJ2tqhKMTbZnHqT7QvNk2V6DUuwPFMYq4fKzI36VZy3v7qWVAhO_4HnThVXs7OlggM_X6p5h3XRVcsWVXlW-KA=w640-h42)

So, we can see all data inside the database. But i still have no idea what these credential used for.

![Database dump](https://blogger.googleusercontent.com/img/a/AVvXsEhjU0LqcmbfU2IxcSyI6SaovuQa6cyYgrluirunWI7s5YPvdSgUOXqEMztPAucC4q6CASlGxa9ZO1IKQ-p7lfF3PWKRtf2APjaWHSNH9zgSmSFv3KasLyh8xrV3yY_-GW7CKb1gqIZ0IWskUteqEY2ZS20SKlYl1GyAfGFD1V9bUYXSjqnDDUaCe0FKUw=w640-h336)

After a while i guess it is credentials to access the machine directly using `ssh` since it is the only services we not touching yet.

Finally, when i `ssh` using username `mario` the authentication success then i just need to read `flag.txt` file. Yeaaay first challenge solved!!!

![SSH flag](https://blogger.googleusercontent.com/img/a/AVvXsEifNAc2Lpyz0UwfTEBVoYdFTfLp0YDecCVzpxMNInQKIG5plows06lR61_HmRpxoJrlN3ROd_UQ-SedEi-uFOHFEkC51mttsLJWpHKgY-0OMiP6wdut3xFznZ8HPxHHbAnj2YD7GtoYaFkecWr4W6dLQ0cmCGOKiyffxRVQtAD1upSIbq0TZ0Tw_Bg08g=w640-h208)

Reference:

* [https://book.hacktricks.xyz/network-services-pentesting/3690-pentesting-subversion-svn-server](https://book.hacktricks.xyz/network-services-pentesting/3690-pentesting-subversion-svn-server)

* [https://tutorgeeks.blogspot.com/2019/11/publicly-exposed-aws-sns-topics.html](https://tutorgeeks.blogspot.com/2019/11/publicly-exposed-aws-sns-topics.html)

* [https://medium.com/appsecengineer/dynamodb-injection-1db99c2454ac](https://medium.com/appsecengineer/dynamodb-injection-1db99c2454ac)
