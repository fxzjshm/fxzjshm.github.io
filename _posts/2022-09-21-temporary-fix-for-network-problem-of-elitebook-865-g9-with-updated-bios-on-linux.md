---
layout: post
title: Temporary Fix for Networking Problem of HP Elitebook 865 G9 with Updated BIOS on Linux 
date: 2022-09-21
author: Fxzjshm
category: Other
tags: [Linux]
---

Update: seems fixed on Linux kernel 6.0.9 / 6.1, commit [1598bfa](https://github.com/torvalds/linux/commit/1598bfa8e1faa932de42e1ee7628a1c4c4263f0a) "platform/x86: hp_wmi: Fix rfkill causing soft blocked wifi"

---

**TL;DR** The temporary fix is either blacklist the kernel module `hp_wmi` currently, or comment out codes that provides rfkill function in that module.

---

Recently Windows auto-updated the BIOS of my laptop HP Elitebook 865 G9 to (U82) 01.02.01 Rev.A, 
then in Linux the wifi and bluetooth device keeps on and off, seems 1-2 time(s) per second,
and character `^@` of unknown source keeps being inputed, even in emergency mode 
(which make it almost impossible to type in password for root, thus cannot access root shell.)

<!-- more -->

I tried to boot into different kernels and found that both 5.19.x and 5.17.0.1016.15 (current `linux-oem-22.04a`) have this problem, but 5.15.0.47.47 (`linux-generic` of Ubuntu 22.04) not. 

I even bought an Intel AX210 to replace the Qualcomm Fastconnect 6900 inside, but still nothing changes.

Today I occasionally disabled the `hp_wmi` kernel module, 
and the icon of network does not flicker anymore! 
I found an update of `hp_wmi` during 5.15 and 5.17 and tried to debug that, but no luck. 
However commenting out codes about rfkill in `hp_wmi` does help, 
so maybe some changes of rfkill and/or the userspace process and/or whatever else 
are incompatible with current version of BIOS.

The temporary fix is either blacklist the kernel module `hp_wmi` currently, or comment out codes that provides rfkill function in that module.
