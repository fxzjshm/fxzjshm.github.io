---
layout: post
title: 实序列快速 Fourier 变换
date: 2025-12-28
author: liam-ji
category: Math
tags: [Math]
---

方法转载自: <https://www.cnblogs.com/liam-ji/p/11742941.html>

## 一、功能

用 N 点复序列快速傅立叶变换来计算 2N 点实序列的离散傅立叶变换。

## 二、方法简介

假设 $ x(n) $ 是长度为 $ 2N $ 的实序列，其离散傅立叶变换为

$$
X(k)=\sum_{n=0}^{2N-1}x(n)W_{2N}^{nk} \ , \ k=0,1,...,2N-1
$$

为有效地计算傅立叶变换 $ X(k) $, 我们将 $ x(n) $ 分为偶数组和奇数组，形成两个新序列 $ x(n) $ 和 $ g(n) $，即

$$
\left\{\begin{matrix}\begin{align*}f(n)&=x(2n)\\ g(n)&=x(2n+1)\end{align*}\end{matrix}\right. , n=0,1,...,N-1
$$

然后将 $ f(n) $ 和 $ g(n) $ 组成一个复序列 $ h(n) $

$$
h(n)=f(n)+jg(n), \ n = 0,1,...,N-1
$$

用 FFT 计算 $ h(n) $ 的 $ N $ 点傅立叶变换 $ H(k) $, 并且 $ H(k) $ 可表示为

$$
H(k)=F(k)+jG(k), \ n = 0,1,...,N-1
$$

由上容易推出

$$
\left\{\begin{matrix}\begin{align*}F(k)&=\frac{1}{2}[H(k)+H^{*}(N-k)]\\ G(k)&=-\frac{j}{2}[H(k)-H^{*}(N-k)]\end{align*}\end{matrix}\right. , n=0,1,...,N-1
$$

求得 $ F(k) $ 和 $ G(k) $ 后，利用下面的蝶形运算计算 $ x(n) $ 的离散傅立叶变换 $ X(k) $

$$
\left\{\begin{matrix}\begin{align*}X(k)&=F(k)+G(k)W_{2N}^{k}\\ X(k+n)&=F(k)-G(k)W_{2N}^{k}\end{align*}\end{matrix}\right. , n=0,1,...,N-1
$$

这种实序列 FFT 算法比相同长度的复序列 FFT 算法大约可减少一半的运算量。

---

$$ W_{N}^{nk}=e^{-j\frac{2\pi nk}{N}} $$
