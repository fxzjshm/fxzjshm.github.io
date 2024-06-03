---
layout: post
title: 安装 CASPER 工具链的注意事项 | Notes for setting up CASPER Toolflow
date: 2024-05-14
author: Fxzjshm
category: Radio Astronomy
tags: [Linux, Ubuntu, CASPER, SNAP]
---

**Manual English translation available below.**

* 部分内容参考 kjlee 的文档
* CASPER 的教程: <https://casper-toolflow.readthedocs.io/en/latest/src/Installing-the-Toolflow.html>
* 本文使用 "推荐" 搭配 (Ubuntu 20.04 + Matlab 2021a + Vivado 2021.1)

## 安装 Vivado 2021.1
* 安装 `ncurses-devel`, `ncurses-compat-libs` (RHEL 8 或衍生版本) 或 `libncurses5-dev` `libtinfo-dev` `libtinfo5` (Ubuntu 20.04) 以安装 libncurses.so & libtinfo.so.5 两个库; 否则安装后的 "list system devices" 卡死无法结束, 可在安装器输出中看到类似 "cannot load libncurses.so.5: no such file or directory"
* 安装 Vivado 2021.1 & Vitis 2021.1
  * 注意 2021.1 对应的 Model Composer 可能不支持 RHEL 8.x (检测系统后拒绝启动)

## 安装 Matlab 2021a
* 不要安装 "Matlab Compiler" and "Matlab Compiler SDK" 否则调用 Vivado 时会卡死
  * <https://github.com/casper-astro/mlib_devel/blob/a557a844f8421f9860876b0216dd6758508d8f2e/docs/index.rst?plain=1#L117>
  * <https://github.com/casper-astro/mlib_devel/blob/a557a844f8421f9860876b0216dd6758508d8f2e/docs/src/How-to-install-Matlab.md?plain=1#L22>
  * <https://support.xilinx.com/s/question/0D52E00006vF6FOSA0/model-composer-v20212-matlab-r2021a-gets-stuck-at-initialization-stage-on-ubuntu-20041?language=en_US>

## 解决运行库依赖冲突
* 大部分情况是 Matlab 或 Vivado 的组件 A 依赖运行库 B, 运行库 B 依赖运行库 C, B 由系统提供, Matlab 或 Vivado 试图提供 C 但太老使得 B 无法加载, 故必须使用系统提供的 C'
<!-- more -->
* "/opt/Matlab/R2021a/bin/glnxa64/MATLABWindow: symbol lookup error: /lib/x86_64-linux-gnu/libgnutls.so.30: undefined symbol: __gmpz_limbs_write", `ldd MATLABWindow` 可见 `libgmp` 太老,
```bash
cd /opt/Xilinx/Model_Composer/2021.1/lib/lnx64.o/Ubuntu
mv libgmp.so.10.0.3 libgmp.so.10.0.3.original
ln -s /usr/lib/x86_64-linux-gnu/libgmp.so.10 libgmp.so.10.0.3
```
* 对 `/opt/Xilinx/Vivado/2021.1/lib/lnx64.o/Ubuntu`, `/opt/Xilinx/Vivado/2021.1/lib/lnx64.o/` 里的 `libgmp.so.10.0.3` 采取相同做法
* Vivado Sysgen 需要 Qt4 (可使用 `ldd sysgensockgui` 检查情况, 文件位于 `/opt/Xilinx/Vivado/2021.1/bin/unwrapped/lnx64.o/`), 可以使用 ppa:ubuntuhandbook1/ppa, 需要安装 `libqt4-network`, `libqt4-xml`, `libqtcore`, `libqtgui`, `qt4-dev-tools`, `qt4-default`
* 使用 simulink 模拟时, Vivado 自带的 binutils 太老导致 Ubuntu 20.04 的 gcc 无法使用 ("no such instruction: endbr64"), 所以不要使用这个 binutils (在 Matlab 里 `!which as` 可见位置):
```bash
cd /opt/Xilinx/Vivado/2021.1/tps/lnx64
mv binutils-2.26 binutils-2.26-bak
```

<br/>
<br/>

---

<br/>
<br/>

* based on notes by kjlee
* setup: "recommended" setup (Ubuntu 20.04 + Matlab 2021a + Vivado 2021.1)

## Install Vivado 2021.1
* install `ncurses-devel`, `ncurses-compat-libs` (RHEL 8-like) or `libncurses5-dev` `libtinfo-dev` `libtinfo5` (Ubuntu 20.04) for libncurses.so & libtinfo.so.5; otherwise post-install "list system devices" never finishes, install log shows something like "cannot load libncurses.so.5: no such file or directory"
* install Vivado 2021.1 & Vitis 2021.1
  * note that Model Composer in 2021.1 seems not supporting RHEL 8.x (refuse to start after detecting OS)

## Install Matlab 2021a
* do not install "Matlab Compiler" and "Matlab Compiler SDK" otherwise Vivado will stuck
  * <https://github.com/casper-astro/mlib_devel/blob/a557a844f8421f9860876b0216dd6758508d8f2e/docs/index.rst?plain=1#L117>
  * <https://github.com/casper-astro/mlib_devel/blob/a557a844f8421f9860876b0216dd6758508d8f2e/docs/src/How-to-install-Matlab.md?plain=1#L22>
  * <https://support.xilinx.com/s/question/0D52E00006vF6FOSA0/model-composer-v20212-matlab-r2021a-gets-stuck-at-initialization-stage-on-ubuntu-20041?language=en_US>

## Fix library incompatibility
* most cases are that component of Matlab or Vivado A requires library B, library B requires library C, OS provides B and Matlab/Vivado try to provide C but too old that B cannot load, so use OS-provided C' instead
* "/opt/Matlab/R2021a/bin/glnxa64/MATLABWindow: symbol lookup error: /lib/x86_64-linux-gnu/libgnutls.so.30: undefined symbol: __gmpz_limbs_write", `ldd MATLABWindow` shows `libgmp` too old,
```bash
cd /opt/Xilinx/Model_Composer/2021.1/lib/lnx64.o/Ubuntu
mv libgmp.so.10.0.3 libgmp.so.10.0.3.original
ln -s /usr/lib/x86_64-linux-gnu/libgmp.so.10 libgmp.so.10.0.3
```
* same for the one in `/opt/Xilinx/Vivado/2021.1/lib/lnx64.o/Ubuntu`
* install Qt4 for Vivado Sysgen (which can be checked by `ldd sysgensockgui` which is in `/opt/Xilinx/Vivado/2021.1/bin/unwrapped/lnx64.o/`): ppa:ubuntuhandbook1/ppa, need to install `libqt4-network`, `libqt4-xml`, `libqtcore`, `libqtgui`, `qt4-dev-tools`, `qt4-default`
* when simulating in simulink, binutils bundled by Vivado is too old and gcc 9 of Ubuntu 20.04 will complain ("no such instruction: endbr64"), so don't use it (found by `!which as` in Matlab):
```bash
cd /opt/Xilinx/Vivado/2021.1/tps/lnx64
mv binutils-2.26 binutils-2.26-bak
```
