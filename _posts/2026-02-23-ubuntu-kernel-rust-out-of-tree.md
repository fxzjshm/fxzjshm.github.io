---
layout: post
title: Ubuntu kernel install out-of-tree rust kernel module
date: 2026-02-23
author: Fxzjshm
category: Linux
tags: [Linux]
---

Here records steps for building [rust-out-of-tree-module](https://github.com/Rust-for-Linux/rust-out-of-tree-module) on Ubuntu 24.04 HWE kernel 6.17.0-14-generic

## Useful Links

* Ubuntu kernel is getting “Rusty” in Lunar <https://discourse.ubuntu.com/t/ubuntu-kernel-is-getting-rusty-in-lunar/34977>

## Step

First Rust support in Ubuntu kernel is a separate package other than `linux-header-...` packages, so:

```bash
sudo apt install linux-lib-rust-`uname -r`
```

Then for HWE kernels, e.g. linux-hwe-6.17, `/usr/src/linux-headers-6.17.0-14-generic/rust` points to `../linux-hwe-6.17-lib-rust-6.17.0-14-generic/rust` but folder installed from package `linux-lib-rust-6.17.0-14-generic` is `linux-lib-rust-6.17.0-14-generic/rust`, thus make a soft link:

```bash
cd /usr/src
sudo ln -s linux-lib-rust-6.17.0-14-generic  linux-hwe-6.17-lib-rust-6.17.0-14-generic
```

Then comes to compiler. If using a different compiler to that when compiling kernel,
build script complains:

```text
  RUSTC [M] rust_out_of_tree.o
error[E0514]: found crate `core` compiled by an incompatible version of rustc
  |
  = note: the following crate versions were found:
          crate `core` compiled by rustc 1.82.0 (f6e511eec 2024-10-15) (built from a source tarball): /usr/src/linux-lib-rust-6.17.0-14-generic/rust/libcore.rmeta
  = help: please recompile that crate using this compiler (rustc 1.85.0 (4d91de4e4 2025-02-17)) (consider running `cargo clean` first)
```

So install that version of rustc from apt:

```bash
sudo apt install rustc-1.82 rust-1.82-src
```

then build with this specific version:

```bash
make KDIR=/usr/src/linux-headers-$(uname -r) LLVM=1 RUSTC=rustc-1.82 -j16
```

Another error complaining unrecognized command line arguments appears:

```text
  CC [M]  rust_out_of_tree.mod.o
clang: error: unknown argument: '-fconserve-stack'
clang: error: unsupported argument 'bounds-strict' to option '-fsanitize='
clang: error: unsupported option '-mrecord-mcount' for target 'x86_64-unknown-linux-gnu'
```

One of the workarounds/solutions is setting C compiler to gcc

```bash
make KDIR=/usr/src/linux-headers-$(uname -r) LLVM=1 RUSTC=rustc-1.82 CC=gcc
```

And load it into kernel and remove it

```bash
sudo insmod ./rust_out_of_tree.ko
sudo rmmod ./rust_out_of_tree.ko
```

Log from dmesg:

```bash
[500433.753777] rust_out_of_tree: loading out-of-tree module taints kernel.
[500433.753785] rust_out_of_tree: module verification failed: signature and/or required key missing - tainting kernel
[500433.754087] rust_out_of_tree: Rust out-of-tree sample (init)
[500443.206517] rust_out_of_tree: My numbers are [72, 108, 200]
[500443.206523] rust_out_of_tree: Rust out-of-tree sample (exit)
```
