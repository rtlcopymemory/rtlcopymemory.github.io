---
layout: post
title: MiniPC Setup for Malware analysis homelab
---

# Setup
I have recently purchased an ACEMAGICIAN AM06PRO to use as a host for VMs as my Reverse engineering and Malware analysis small homelab.
It comes with 16GB of ram and a Ryzen 7 5825U (8C, 16T) and 512GB of M.2 SSD to which I added a 1TB Sata SSD,

# Installations
I immediately replaced Windows 11 with Ubuntu Desktop installed using ZFS for the main filesystem and formatted my Sata SSD, which will be used for VM disks, as btrfs.
However a friend of mine who knows much more about VMs gave me some tips:
- `nodatacow` is needed for btrfs to not lose all the performance when using it as a VM disk destination
- Use [bees](https://github.com/Zygo/bees/tree/master) to perform deduplication on btrfs

I also installed tailscale and enabled RDP so that I can access it from my main PC and even when I'm not home!
