---
title: "Simple Way To Import .ova file to KVM"
date: 2025-10-06
draft: true
author: "Lychnobyte"
tags:
  - Linux Related Tips
image: "https://cdn.hashnode.com/res/hashnode/image/upload/v1751849254101/029ff624-075b-4b4c-8033-cd56f838de68.jpeg"
description: "A simple guide on how to import .ova files into KVM by converting them to qcow2 format."
toc: true
---

Well, simple post here.

So, while i train/recall my pwn skills i found some cool `PWN labs` by `samiux` here [https://cybersecurity-ninjas.com/ctf-pwn.html](https://cybersecurity-ninjas.com/ctf-pwn.html)

The provided file to deploy the labs is file with format `.ova` which is used by virtualbox and vmware. Since i’m a KVM enjoyer, i need to convert it before the vm can run in KVM.

Here the simple steps

```bash
# Install virt-v2v to convert file
sudo apt install virt-v2v

# Run convert command
virt-v2v -i ova /PwnCTF_22.04_v20230202.ova -o libvirt -of qcow2 -os nvme-pool -n host-network
```

**options:**

*   **-i** : input mode

*   **-o** : output mode

*   **-of** : output format file

*   **-os** : output storage (storage pool in kvm)

*   **-n** : network used by vm

After execute command above, now the vm been listed in `virsh`

```bash
 virsh list --all
 Id   Name                     State
-----------------------------------------
 -    PwnCTF_22.04_v20230202   shut off
```

Well, sometimes we need to make adjustment inside the VM like interface name. We can reset the `root` password for vm using command below:

```bash
# Check disk output file path
virsh vol-list nvme-pool
 Name                         Path
-------------------------------------------------------------------
 PwnCTF_22.04_v20230202-sda   /nvme-lv/PwnCTF_22.04_v20230202-sda

# Reset password
virt-customize -a /nvme-lv/PwnCTF_22.04_v20230202-sda --root-password password:<new-password-here>
```

Then, we can start the VM, login and make some adjustment if needed

```bash
# Start the vm
virsh start PwnCTF_22.04_v20230202

# Login console
virsh console PwnCTF_22.04_v20230202
```
