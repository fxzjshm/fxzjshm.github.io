---
layout: post
title: GDB 9 incompatible with Clang 14
date: 2023-07-29
author: Fxzjshm
category: HPC
tags: [Linux, simple-radio-telescope-backend, SYCL, CUDA, HIP, ROCm, MUSA]
---

When testing SYCL programs on some backend based on Clang 14, it is noticed that the `gdb-9` bundled with Ubuntu 20.04 cannot correctly read debug info of binary produced by `clang-14`, then stack info is not available at all.

Although haven't figured out why, compiling & installing a `gdb-13` simply fixes this problem.

NOTE: tested with vendor-specific compiler based on `clang-14`, not sure if applicable to vanilla Clang.

<!-- more -->
