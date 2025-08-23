---
layout: post
title: MiniPC Setup for Malware analysis homelab
---

# Setup
I have recently purchased an ACEMAGICIAN AM06PRO to use as a host for VMs as my Reverse engineering and Malware analysis small homelab.
It comes with 16GB of ram and a Ryzen 7 5825U (8C, 16T) and 512GB of M.2 SSD to which I added a 1TB Sata SSD.
![screenshot of the homelab RDP session showing a FlareVM installed in a qemu/KVM environment](https://github.com/rtlcopymemory/rtlcopymemory.github.io/blob/master/images/homelab/image.png?raw=true)

# Installations
I immediately replaced Windows 11 with Ubuntu Desktop installed using ZFS for the main filesystem and formatted my Sata SSD, which will be used for VM disks, as btrfs.
However [a friend of mine](https://ioctl.fail/) who knows much more about VMs gave me some tips:
- `nodatacow` is needed for btrfs to not lose all the performance when using it as a VM disk destination
- Use [bees](https://github.com/Zygo/bees/tree/master) to perform deduplication on btrfs

I also Installed tailscale and enabled RDP so that I can access it from my main PC and even when I'm not home!

# VMs
## REMnux
I converted the official `.ova` after installing REMnux from scratch twice and having issues with default packages.  
After booting up the VM, internet wouldn't work. Checking `ip a` I see the interfaces are not configured correctly so I checked the `netplan` config and found out the NIC name was wrong in the config file, changing that to my NAT default fixes the internet. After that I also added an extra NIC for a "host only" network and cofnigured it with a static IP.

# FlareVM
For this I just installed Windows 10 on a VM and followed the instructions to install FlareVM. At the end I replaced the NIC network from NAT to the "host only" network I created and gave to REMnux too.

