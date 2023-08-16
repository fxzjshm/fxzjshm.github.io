---
layout: post
title: fp_contract causes inaccurate result
date: 2023-07-27
author: Fxzjshm
category: HPC
tags: [Linux, simple-radio-telescope-backend, SYCL, CUDA, HIP, ROCm]
---

Strange things happened when debugging the new pipeline structure of simple-radio-telescope-backend: when `emulated_fp64` is switched on (that is, the `df64` type in ["Emulated double precision Double single routine header"](https://forums.developer.nvidia.com/t/emulated-double-precision-double-single-routine-header/4686) by StickGuy, Norbert Juffa, Reimar, et al. is used), under Debug configuration every thing works fine, but Release configuration gives wrong results!

An example is evaluating phase delay of radio wave in plasma caused by electrodynamics, the difference of `cos()` and `sin()` of such phase evaluated using `float64` and `df64` is shown below:

![df64_double_debug.svg]({{ "/img/2023/fp-contract-causes-inaccurate-result/df64_double_debug.svg" | prepend: site.baseurl | prepend: site.url }})

*Difference of result given by `df64` & `double` under Debug configuration, tested on NVIDIA RTX A4000*

![df64_double_release.svg]({{ "/img/2023/fp-contract-causes-inaccurate-result/df64_double_release.svg" | prepend: site.baseurl | prepend: site.url }})

*Difference of result given by `df64` & `double` under Release configuration, tested on NVIDIA RTX A4000*

<!-- more -->

This is most likely caused by some inappropriate optimization, and after some investigation this post on NVIDIA forum ["Different results in Debug and Release mode compile"](https://forums.developer.nvidia.com/t/different-results-in-debug-and-release-mode-compile/39860/2) leads to fused multiply add (FMA) operations, then this Stack Overflow post ["clang/gcc only generates fma with -ffast-math; why?"](https://stackoverflow.com/questions/55974090/clang-gcc-only-generates-fma-with-ffast-math-why) gives some clue: `#pragma STDC FP_CONTRACT ON/OFF`.

Referring to [Clang's documentation of `-ffp-contract`](https://clang.llvm.org/docs/UsersManual.html#cmdoption-ffp-contract), the accepted values are:

> * fast (fuse across statements disregarding pragmas, default for CUDA)
> * on (fuse in the same statement unless dictated by pragmas, default for languages other than CUDA/HIP)
> * off (never fuse)
> * fast-honor-pragmas (fuse across statements unless dictated by pragmas, default for HIP)

Tested `-ffp-contract=on` and the problem went away.

Till now everything was good, but then I realized that there's again an exception for CUDA: it simply ignores `#pragma` derivatives, unlike HIP's `fast-honor-pragmas`, and the later one isn't supported by CUDA. Quite annoying. 

Then I have to completely set `-ffp-contract=on` on CUDA for correctness of the results, and any performance degradation is none of my business; in fact, the whole `df64` technique had been completely unnecessary, until CUDA came into the game.

> "NVIDIA, f**k you."  -- Linus Torvalds
