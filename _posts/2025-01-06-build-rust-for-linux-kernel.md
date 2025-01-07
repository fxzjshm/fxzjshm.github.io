---
layout: post
title: Build Rust-For-Linux kernel
date: 2025-01-06
author: Fxzjshm
category: Linux
tags: [Linux]
---

Ref: <https://leosaa.medium.com/build-the-latest-kernel-package-for-debian-85b09ec6af88>

```bash
cp -v /boot/config-$(uname -r) .config
make menuconfig
make olddefconfig
```

Disable signing keys and debug:

```bash
scripts/config --disable SYSTEM_TRUSTED_KEYS
scripts/config --disable SYSTEM_REVOCATION_KEYS
scripts/config --disable DEBUG_INFO
scripts/config --enable DEBUG_INFO_NONE
```

Note, currently rust support requires MODVERSION to be disabled,
double check if Rust supports is shown in menuconfig.

Check setup of rust by:

```bash
make -j`nproc` LOCALVERSION=-rust LLVM=-19 rustavailable
```

where `-19` is version of Clang used.

Then build:

```bash
make -j`nproc` LOCALVERSION=-rust LLVM=-19 bindeb-pkg
```
