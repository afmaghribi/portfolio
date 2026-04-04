---
title: "HTB Business CTF 2022 - Operator"
date: 2025-06-24
draft: false
author: "Lychnobyte"
tags:
  - cloud
  - ctf
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1751848401365/d096eefa-5ccf-4ef1-95a6-0b6677255a4b.jpeg"
description: "Solving the 'Operator' challenge from HTB Business CTF 2022, involving Gogs, AWX-operator, and Kubernetes."
toc: true
---

> **_Cover Illustration source_** [https://www.pixiv.net/en/artworks/89193760](https://www.pixiv.net/en/artworks/89193760)

Alright, let’s continues to 2nd challenge called `Operator`. Here is the challenge description:

> We have located Monkey Business operator blog where they are leaking personal informations. We would like you to break into their system and figure out a way to gain full control.

Just like previous challenge we only got an `IP address`. So i just running `Nmap` to scan the `IP address` to know what services are running and in which port.

![Nmap output](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhFvoDudyenMEZeX2XQ_vqF44m-WjZgO-WNehLB2OcgEsjQHbpyZBHKnEl2p1qFc-kC9bHePc7wOdOH06m3_9H-KVynuQBKBvGEAGeThBeM8MDsQa3qL0gEYHWLRr93xnFEcdGq7AkQQngI_7NKe7quvdam-s3VZPdS5bSC3Bu_mFUOAvBdgvM1qnmrmw/w640-h200/Untitled.png)

The scan result show there is 6 open ports:

1. 22 - ssh

2. 80 - http

3. 3000 - ppp

4. 8443 - https (most likely kube api server)

5. 10250 - http (most likely kubelet)

6. 30080 - http nginx

Let’s explore each services one-by-one

### 1. 80 - Web server

Access from browser we got website page like this.

![Blog page](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiFj06MHvi36jxgWkPKzZaBZE3bqQi_UHZvm9lIHH2e7LGkqYC-Xf_P4ntuNquoneDgGNsixbIyqv1X1Yp8uxM72dW82KSNfQyzKY1mnZC4rihU5Fjw4OZvyH4vqQBzZmdIHtqsTG6k-7b9uIGwq-Vp2ywV5OfPuTUfwVzQlYx2A3XY5PnYa4pF_3V09w/w640-h344/Untitled%20(1).png)

After check all posts on the website, there is one post that provide a link that leads to port 3000. Let’s visit that link.

![Blog link](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgppb0CXApaDzhJHoIhf9Kv0mxyy6DRipcF82iODS90o06GEtNwfsBLFhiGTkqHZui6oZbpQvP-1-nWrJORzr_9GTIa3PwWMH8MrdILdK5ET-KRZ4QqLLUCb7jKe_h5BtZ0OCimznIE3U7GD2Bv5SKRkYFMjcYx1pL9bvYjD1z-sFll66mZT7OphV6FbA/w640-h340/Untitled%20(3).png)

### 2. 3000 - gogs (self-hosted git server)

The link that we found earlier leads us to git repository.

![Gogs repo](https://blogger.googleusercontent.com/img/a/AVvXsEgNd0lOIiqbubaxkeKpK2XaUOtHamlKIXYQPfQpw75gQoBDIWGYt2Y_m9HoHtWutG5ArQek4vIgOYPCDnJsl6cJNyCZV0jj2DEzbI5QHAD1T1wyllzOBDLzTjpVn-k9HdpFQQoEqhqP6HPGpNI4iMhRzoQGn8-RHyUXldnKkBZoWOA7uAtze4LzEUS2PQ/w640-h338/Untitled%20(4).png)

Since the repository itself seems not have useful information i tried to explore the gogs server to find another repository.

Yeah, there another repository named `awx-operator`

![AWX Operator](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjtYBffTJE4Rsm_Mae88dwNxnPBQPdlRBZhNTrdMn_95IIpOb73d3OULz1DU-Cm938eV9DKqm4HYzRnLet7DgIjRnJuxfEXQGo82_8-dtWPRzbpPWcVWNuuW5x_1jiVlhuxmJwNE4K8meF7EuWjO0jvJpOwALZXrcNPzXH-yexhHQPlTexO3Szm5BQLiA/w640-h138/Untitled%20(5).png)

Explore the repository and i found a closed issue. That issue mentioned another repository called `awx-k8s-config`.

![Gogs issues](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiJJjmWZh8ontzI34hIBgAvHVcnw0idZxfUy6PoWYoix6esL6KmR8_eFJEQFbr1yHQssw47pz_LXauEAIlixNPWDB9cWPtkuFtyZakq0XZfza5BoHheQ-fs2LBL1F3TF8Cf7B60D2rgqhMAPM0G__KclWFHxjNzs8DbDTLKrJhe127XayfpECb3zFoPTw/w640-h276/Untitled%20(6).png)

Then i open the repository and try to explore the files stored in the repository. Seems like the files stored in the repository are manifest file to deploy `awx-operator` in kubernetes. From commit history i found credentials for login to `awx operator dashboard`

![Credentials found](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiyqGOCnE-nCbXbNNFc9liPbEshvab_WKU9Svl5DyWJfR8KRw2o3cMEkJMo9gStvymkSQHgtNLQ1xCRhOd7w5F62bhHACmCgTDQDUNtMffkschE5ZGBPNUy9AqzTch7LWRn1SEBJoygYEMwXbhx_7vxwoZ3mXqVq45p978irgTvdV0BOGHOspyyOe1NJg/w640-h230/Untitled%20(7).png)

### 3. 30080 - Awx dashboard

Login to awx dashboard using credentials we got before, here the dashboard look like.

![AWX Dashboard](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEgFRVGT9ItniruWdUt9uehVauotEFIH7XjUeothYFV8ec9YlLQNTeaK2Y8Eq8HJut6PjICfCNJ-3aKQfsd-6OfuEc5sRXxmt5Qx34PT_mDxAPTYdnIfToNEwhJ2SGGtJ6iD7NFh7RH_8ZzReftTBVMoSAUx5jU27Li2B4UqWy3ZA81RFUcbuboM-OAQsg/w640-h340/Untitled%20(8).png)

Briefly, how awx work is awx will deploy pod in kubernetes cluster then make it as runner to to run ansible playbook. So, we can simple create our own ansible playbook to get kubernetes serviceaccount token that we can use to access kubernetes cluster.

First, we create new user and repository on gogs server. Then we stored our ansible playbook that print serviceaccount token that stored in file `/var/run/secrets/kubernetes.io/serviceaccount/token` inside the pod. Here the playbook that i use.

![Ansible Playbook](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiGrBxyEOkA1_NR2cGvUWBUwaxGRiwO2N9vZ4UngPlMVQWdpbhWo39HGjeNQqNclf93ngOskjH2LW3WOHxuFBVUoQXjcpJ6zmyQ73XBOQDdoYdNJvJiKFjDuXsiUZLYXqr_Wq50YM7aAnTSe_fnRh0npDn3D7a_ZuU76GgAwKTzm9F-3RYPjNCM7o9qBg/w640-h206/Untitled%20(17).png)

Then on awx dashboard we create new project that use our new repository as source control.

![AWX Project](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhHaoV2dBmE3_VZlVheNrQGVsMPNiDUAJcTwkbCZhmFm3ft5LRD0Dpeaf3_Po77DJUZLzPChOKMNdhZ7tlFn72ASSP58K6xIEQxsIGwo-W7YsZMYksdoMnWFMUyqhkeW9jLtIjWxEk-8hXPakHr-7yitDOl3BkuNVjFQunTl-uYCKpisji3h5w-VcOvkw/w640-h300/Untitled%20(9).png)

Then we create new job template for the project to run ansible playbook.

![AWX Job Template](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjW4i9lOy05oNcjEe21Jvb7ubOLhLtOgF-jTJxvQRF_RsrLn4uQpyoA8XGS2prpnpoe8MMFymRdkx5XmxtQwnpZZrG7kEwxNHG0cQAhQM42hhKrUIbvvB2W_oSQoLbDyxOHA2VIJQZ6nsDbpNQ68DIBABjWGHAjiIDNwf5HsBbomqGqpLWirg44z0YtHw/w640-h314/Untitled%20(10).png)

Then we run the job, but i got an error. Seems like the file that contain serviceaccount doesn’t exist in the runner pod.

![AWX Error](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjSuwG1clieA8jbnKxJutlthQc6_1Qtp153hWx8lLkGzzSzuciORe7h33Nu2zbK-RDNDOCCxULaj3Gt1x5j_gbs28dWSFfeWaA6NrGCI8EMThn0dzxYher7xTxKi4Ilv7SVOJnFHZekqAJMyfPR5_OwgkSY03NwkgMDUorsTKt5sdopBvEglRY-esnCdg/w640-h324/Untitled%20(11).png)

After explore a while i found that the pod spec that deploy as runner has `automountServiceAccountToken` option set to `false`. We need to change that option to `true`. The configuration can found in `Instance Group` -> `choose group` -> `customize pod configuration`.

![AWX Pod Config](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEg_Wq5xpj7srsCGnw9bz4qeskk1vKVfM4wdbuzPioQb4188igja-l_1mof_APWofAXLcngBO5Pm_enKoTizdjsbldBi2zoKhjlP_3SHUlZA39xSATRPWjUof5MNx8so5ggVrQJOqUEIlE7GrpAKVKSanDDy6efrhRjmz58KrL7iBEvMWEw0WzfBgNbkbA/w640-h252/Untitled%20(12).png)

Then re-run the job we’ve created. Now finally we got the serviceaccount token.

![AWX Success](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEizCKaG288QUf5HryxnHZxkCTWAXsnzfOPBZs_vq7dtOCpH5KLNjdhQMtxnG7MwYcE_s9NKNwOGvm_jz_25-32aU4q1EhQz_YcZ7rN--cxDteG1ykgWqcbSQWaDloosBzieIa_gNVxhqNBjN_Nc_72V3vXKbkyW0Kt8IoetWCo1Cxx-a7Bt5ZCQYpVPHA/w640-h332/Untitled%20(13).png)

Next we use that token to access the kubernetes cluster with kubectl.

![Kubectl access](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjDX9c3FfqPm_Fh-Tt8kAE7Vz3Z1FRsNT-O71fVBAj-5h_b33o5A3vWv1OGPITeUD9ObJ6SC3zJtcchiLGjRWzixLo44YRgcl81LuFHkIzhqL_HXPs4PGBFrTT9uRAkIqa2zVO85u6R8pt_TBH6HRTMcAKdiOlOUj_8QaKZo5IsxgyPtuSGEB9nlf5Jng/w640-h154/Untitled%20(18).png)

When i tried to list pods running got error permission. Seems like our token doesn’t has such permission, since we use the `default` serviceaccount. Well we change it but first we need to know what serviceaccount that exist in the cluster. We can open `awx-operator.yaml` file in `awx-k8-config` repository to see that we can use `awx-operator-controller-manager` serviceaccount.

![Service Account info](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEiDKNLInik7kAiXD_GgAjSu2VP7Y0CxF2r4GCmW4iO9uhvNHV703cO9cwKqSHUH-1dz5FCieUMML9FQxGSqTg0fK8P8kOkBMHoNRo-joRwExq18SzuNlXXPpScUgnUnKmCbTeaJ8LKwS6Ipa76gStZqLaXX49VDAUZc8hA1-1ohZv5Q8IffUZOlte1W7A/w640-h228/Untitled%20(14).png)

To make it our runner pod using that serviceaccount we need to set option `serviceAccountName` in pod configuration.

![AWX Service Account config](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhnMDXSr-bO7nuz33JH_r7VTID6pvFAjqxG6f7vBpeiMGXWPB47sbZoUlOJZm88BpbF8U5hcimaz54XFh6fJKSkpqm6ZXSarmVVTfFYHX505ExMfzd25H_NhbDo5KlH4DTXDj-wwlBerZ7p1-9-QJLdg9v7Ihb7-YL1k1qSGknBKY_yXMkdqOb1-OPbpQ/w640-h258/Untitled%20(15).png)

Re-run the job to get new token from new serviceaccount

Then try run kubectl using new token, here i check the permission we got.

![Auth check](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhu0KfxLd4SXfjTFW7mrehQihAYQKA4segLCUhGGS1iTjI_mMyNzCw7XLZ2z3OWpSoD2kyuKHcfu7UPWN97Nt98VuJAWVD-cyPehv0RAVtseZhRT85GYXns7ajI8i2zvDuCrhkvWnkEmrpxVYufh4fk_h-8TvENchpETObHs7laWKyyWYsWoTnmexp8oA/w640-h280/Untitled%20(16).png)

Seems like now we can create new pod. Since the `flag.txt` file are stored in the host machine we can create pod that mount `/` host machine to the pod, so we can read the `flag.txt`. Here the pod manifest that i use.

![Pod Manifest](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhomFHRpyC2-Ibb1lf7MoTzGOFuWStTZyO4UBgryKWPvVdNjPJ9YgQTcOQFHVBBRACoDRUHzi-Kahq3SMSTGPbAyByoTEGMM7GEfZgHOUUX0_XwFJlZ3eOptjaffzHCAPXwcTn8Iba2oTt8jjLjUx4y8JZZJVh3-TLJetlEqb48peaZlOMIWEgQKRgB6A/w640-h372/Untitled%20(20).png)

Next just create the pod and exec shell inside pod then read the `flag.txt` file.

![Flag found](https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEjsKpKknNViBuouFpEXxktS4U-qTOeO4CSoSObAb-rGybHVDQe6dLdwvhyP247JLtIwyg-rlXCQcbAQ1SHZO-3biOCMd6yzTaYc3-6hxbexvB0eml6jwgW1yxw10q99ugErkOfL-tfTSoaMuJ5wcVuAyTfT4BBdKPIyvFIJD2lQ0fwziLotoWxCd0smIQ/w640-h202/Untitled%20(19).png)

Reference:

* [https://faun.pub/attacking-kubernetes-clusters-using-the-kubelet-api-abafc36126ca](https://faun.pub/attacking-kubernetes-clusters-using-the-kubelet-api-abafc36126ca)

* [https://github.com/kubernetes/kubernetes/issues/2797](https://github.com/kubernetes/kubernetes/issues/2797)
