---
title: "My Own Challenge - That Day"
date: 2025-07-09
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - ctf
  - kubernetes
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1752029083028/7b70675a-a55d-42b7-be7c-9688b06afd28.jpeg"
description: "Walkthrough for the 'That Day' challenge, focusing on exploiting misconfigured Kubelet APIs and service accounts."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/104822829](https://www.pixiv.net/en/artworks/104822829)

Hello all, let’s continue to my second challenge called `That Day`. This challenge is about get access to kubernetes cluster. Here the challenge description

> But perhaps you hate a thing and it's good for you And perhaps you love a thing and it's bad for you :')
>
> [`http://foryou.lychnobyte.my.id`](http://foryou.lychnobyte.my.id)

So, this challenge was once deployed in VPS and attached with my domain but now the challenge is down. If you wanna try to solve it you need to deploy the cluster and service yourself.

All you need is ubuntu 20.04 (or later version) VM with specs 2 vcpu and 2 GB ram. Then follow steps on `deploy_notes.md` after that deploy manifest `deploy_sites.yaml` with `k create -f deploy_sites.yaml`.

[https://github.com/afmaghribi/BrokenHeartEdition/tree/master/Cloud/That_Day/deploy](https://github.com/afmaghribi/BrokenHeartEdition/tree/master/Cloud/That_Day/deploy)

Well, i use `microk8s` here for simplicity to deploy single node kubernetes cluster. You can deploy kubernetes cluster with another tools but i things it’s too much just for simple service.

After all set you can add static domain to `/etc/host` on your pc to resolve vm ip to `foryou.lychnobyte.my.id` or you can directly access your vm ip address in browser. Here the sites page look like.

![Challenge Site](https://cdn.hashnode.com/res/hashnode/image/upload/v1752032234263/fbc3bfdc-01fe-4dd1-98fa-0ec3cbfe9a46.png)

I know, it is too cringe like asdfafgdafgsdag >_< sorry -_-”

Anyway, back to challenge objective and assume we don’t know what service are running in vm. Again it is just a static website so we need to scan the ip vm to know what services running on vm using `nmap`

```python
nmap -sS -sV -Pn -p- -T5 -n foryou.lychnobyte.my.id
```

Wait a while and here the result look like

![Nmap results](https://cdn.hashnode.com/res/hashnode/image/upload/v1752033038303/8bd77da8-f319-43ba-884f-8c58cde54b68.png)

Many ports open but we know that in `Operator` challenge [here](https://blog.lychnobyte.my.id/htb-business-ctf-2022-operator) port `10250` are exposed by `kubelet` service. So, it is most likely the vm is `kubernetes` node and for `apiserver` port commonly use `6443` or `16443`.

Let’s access `apiserver` using browser and we got `forbidden` response. But, confirm that it is `kubernetes` cluster `apiserver` endpoint.

![API Server forbidden](https://cdn.hashnode.com/res/hashnode/image/upload/v1752034677170/c8ed8db2-93be-4de3-bb24-9ee94522c2c1.png)

Well, so we don’t have access directly to the `apiserver` port but what about access in `kubelet` port?

![Kubelet 404](https://cdn.hashnode.com/res/hashnode/image/upload/v1752034990335/a80bc04c-6100-4890-a493-d34c591369f5.png)

Well, it is only response `404 not found` not `forbidden`. Then try to list running pods using `/pods/` endpoint

![Pods List](https://cdn.hashnode.com/res/hashnode/image/upload/v1752036223532/aba73d01-807d-4a0f-8557-bd656795c999.png)

Yeah, we got list of pods along with details metadata for each pod.

Alright, next step is as usual. We retrieve serviceaccoount token inside the pod then use it as authentication to access the kubernetes cluster.

First, lets find good pod candidate that probably has high serviceaccount privilege.

Pods `situs-diriku-f9b4bdc98-mp7dk` seems a good candidate

![Pod Candidate](https://cdn.hashnode.com/res/hashnode/image/upload/v1752036545004/28145e9a-fa22-4d37-a020-ab3922786145.png)

Because the pods use custom serviceaccount name `sa-diriku` and has `automountServiceAccountToken` set to `true`

![Service Account Token Info](https://cdn.hashnode.com/res/hashnode/image/upload/v1752036994764/8e00b964-8f87-40f3-bc3f-11a7d61fdc09.png)

So, how to retrieve the token inside pod? Well, `kubelet` has endpoint `/run` that can be use to run command inside desired pod. Here the pattern to remote command execution.

```bash
curl -XPOST -k \
https://${IP_ADDRESS}:10250/run/<namespace>/<pod>/<container> \
-d cmd="command to exec"
```

In our case we use we want to retrieve serviceaccount token, so this is our `curl` request.

```bash
curl -XPOST -k \
  https://foryou.lychnobyte.my.id:10250/run/jakarta/situs-diriku-f9b4bdc98-mp7dk/situs-diriku\
?cmd="cat%20/var/run/secrets/kubernetes.io/serviceaccount/token"
```

Successfully retrieve the serviceaccount token.

![Token Retrieved](https://cdn.hashnode.com/res/hashnode/image/upload/v1752039204780/b936b0ca-967a-438a-9ae9-095908e0113e.png)

Now, we can use the token to access `kubernetes` cluster using `kubectl`. Check authorization of the token we have, don’t forget to specify `namespace` to `jakarta` because `serviceaccount` only has `namespaced` scope permission.

```bash
TOKEN="serviceaccount token"

kubectl --insecure-skip-tls-verify=true --server https://foryou.lychnobyte.my.id:16443 --token $TOKEN --namespace jakarta auth can-i --list
```

Well, we have permission to `list` and `get` object `secrets`.

![Auth can-i list](https://cdn.hashnode.com/res/hashnode/image/upload/v1752039849815/48c1281d-e836-412d-bb63-988a4b37002b.png)

Then we can just `list` and `get` the `secrets`, don’t forget to decode as `base64` to retrieve the plaintext.

![Secret retrieval](https://cdn.hashnode.com/res/hashnode/image/upload/v1752039956709/e1546302-19cc-47ea-a513-0d247873dbf4.png)

That it we got the flag!

Flag: `TCP1P{4nd_c3l3br4t3_y0ur_h4pp13st_d4ys_th3r3_:')}`

### **Reference:**

1. [https://faun.pub/attacking-kubernetes-clusters-using-the-kubelet-api-abafc36126ca](https://faun.pub/attacking-kubernetes-clusters-using-the-kubelet-api-abafc36126ca)

2. [https://microk8s.io/docs/services-and-ports](https://microk8s.io/docs/services-and-ports)
