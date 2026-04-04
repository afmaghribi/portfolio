---
title: "My Own Challenge - Last Forever"
date: 2025-07-08
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - ctf
  - aws
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1751944476886/d9c349a9-f197-4380-bc15-e4efdc3c1d19.jpeg"
description: "Exploring AWS S3 bucket versioning to retrieve a hidden flag."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/97570152](https://www.pixiv.net/en/artworks/97570152)

Hello all, i will continue write about cloud ctf challenges. But, instead of solving challenges from ctf competition out there. In this post i talk about my own challenges. Well, the challenges was listed on one of ctf playground platform but now the platform itself is closed so i decide to write about it.

<details><summary>(yapping) unrelated with challenge</summary><div data-type="detailsContent">Well, this is just personal note regarding the challenges i created. If you realized the theme of my challenge is heart break related, because that what moves me to make the challenge. So, i just say sorry if the challenge itself looks cringe and annoying, lmao😬. During that time i experienced the broken heart (again), but that time hits different. Cause like for just a day she make me felt like “this is what i looking for my whole 25 years of living”. Aside of the person itself that amazing (for me) the whole experience that day was exceptional, cause whenever i remember that day (till now) i always feel so unreal just like a dream especially for a person like me to have experience like that. So, why created challenges while i broken heart?. I just make myself busy (easy way to pass the phase) and express my sadness by draining my tears (literally crying during make the chall, lmao) while also do something i love (created things) and same time also can be useful (maybe there is takeaway in challenges, idk lol) it is win-win move isn’t it? lmao. So, yeah i just did and one year later i still here feel the same ._.</div></details>

So, first challenge is called `Last Forever` which is about aws s3 bucket. Here the challenge description.

> I have erased all my memories of you. But, why are you still in the deepest part of my heart? :')
>
> [`http://forever.lychnobyte.my.id`](http://forever.lychnobyte.my.id)

Well, the challenge is still up and maybe you can try to solve it by yourself before reading this post. I will give source code and how to deploy the challenge later in this post.

Let’s start to solve this challenge.

First open the link provided in description with browser, here the page we got.

![Challenge Site](https://cdn.hashnode.com/res/hashnode/image/upload/v1751947908374/cba4910b-a3a2-4cf9-9329-591a126548bf.png)

Nothing special with the website itself just a static page. Since it is `cloud` challenge it might useful if we use `dig` to know what cloud provider that use in this challenge.

![Dig command](https://cdn.hashnode.com/res/hashnode/image/upload/v1751948119555/e8675d4a-2779-4e61-b525-bd70ea6d3388.png)

From `dig` answer section we can see that website are `served` as static website that provided by aws `s3` bucket. Because it using CNAME `s3-website.<region>.amazonaws.com`.

Since it is static website the bucket should has public access to list objects. So, we can just go to `http://forever.lychnobyte.my.id.s3.us-east-2.amazonaws.com` to list all objects in the bucket.

![Bucket listing](https://cdn.hashnode.com/res/hashnode/image/upload/v1751948746009/ff3a92e9-c180-4ab1-b445-79023da0e2dd.png)

As we can see there is several objects, the unusual objects are `memories.txt` and `myheart.txt`.

Try open the `memories.txt` object, seems like we need to open `myheart.txt`

![Memories.txt](https://cdn.hashnode.com/res/hashnode/image/upload/v1751948993822/95d63925-24fb-495d-a13e-7cb8e4898220.png)

While open `myheart.txt` object, it mentioned `deepest`.

![Myheart.txt](https://cdn.hashnode.com/res/hashnode/image/upload/v1751949054849/a9705ca7-a8d5-4586-8d98-b6fdc57e375b.png)

So i guess it is related to `bucket versioning` in aws `s3` bucket. It is a feature to enabled bucket to still stored the old version of object.

Well, we can list the old version by just append `/?versions` in the bucket url.

![Bucket versions](https://cdn.hashnode.com/res/hashnode/image/upload/v1751949287101/573634f2-1f76-46fd-9e50-fa8245e43aac.png)

Well, there is many versions available for object `myheart.txt`. Let’s try to open one of old version `myheart.txt`. The link pattern to open object in certain version is by append `<object-path>?versionId=<version-id>`.

![Object version](https://cdn.hashnode.com/res/hashnode/image/upload/v1751949623544/07fd775d-98f7-4573-800a-0be123a93acd.png)

Hmm, it is only show 1 letter. So, i assume to get whole `flag` we need to retrieve all letters then combine it. Because manual works is so boring, let’s use some script solver. Here the solver i use

```python
import requests
import xml.etree.ElementTree as ET

res = requests.get('http://forever.lychnobyte.my.id.s3.amazonaws.com/?versions')

root = ET.fromstring(res.text)

all_versions = []

for versions in root.findall('{http://s3.amazonaws.com/doc/2006-03-01/}Version'):
    version_id = versions.find('{http://s3.amazonaws.com/doc/2006-03-01/}VersionId').text
    file_name = versions.find('{http://s3.amazonaws.com/doc/2006-03-01/}Key').text
    if file_name == "myheart.txt":
        all_versions.append(version_id)

flag = ""

for version in all_versions[1:]:
    res = requests.get('http://forever.lychnobyte.my.id.s3.amazonaws.com/myheart.txt?versionId=' + version)
    flag += res.text.strip()

print(flag[::-1])
```

So, just run the solver then we got the flag.

![Flag obtained](https://cdn.hashnode.com/res/hashnode/image/upload/v1751949952392/153160db-314f-473e-afca-6247e9b91522.png)

Flag: `TCP1P{jus7_l1k3_wh4t_1_s4id_y0u_4lw4ys_r3m4in5_h3r3_f0r3v3r_:')}`

It just a simple challenge isn’t? :)

Well, you can see all source code for this challenge in my repository here [https://github.com/afmaghribi/BrokenHeartEdition/tree/master/Cloud/Last_Forever](https://github.com/afmaghribi/BrokenHeartEdition/tree/master/Cloud/Last_Forever)

Since, the challenge is still up and bucket still exist if you want to deploy your own bucket you change the `credentials` and `bucket name` in `main.tf` file

[https://github.com/afmaghribi/BrokenHeartEdition/blob/25a7da19f48a40ed346ce960d7dd30c110a7b14e/Cloud/Last_Forever/deploy/main.tf#L9C1-L20C1](https://github.com/afmaghribi/BrokenHeartEdition/blob/25a7da19f48a40ed346ce960d7dd30c110a7b14e/Cloud/Last_Forever/deploy/main.tf#L9C1-L20C1)

```python
provider "aws" {
  profile = "awscli"
  region  = "us-east-2"
  shared_credentials_files = ["/home/curiozan/.aws/credentials"] >> Change here
}

# S3 Bucket name

resource "aws_s3_bucket" "my_s3_bucket" {
  bucket = "forever.lychnobyte.my.id" >> Change here
}
```

### **Reference:**

1. [https://docs.aws.amazon.com/AmazonS3/latest/userguide/list-obj-version-enabled-bucket.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/list-obj-version-enabled-bucket.html)

2. [https://docs.aws.amazon.com/id_id/AmazonS3/latest/userguide/RetrievingObjectVersions.html](https://docs.aws.amazon.com/id_id/AmazonS3/latest/userguide/RetrievingObjectVersions.html)
