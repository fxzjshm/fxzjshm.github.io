---
layout: post
title: 安装脉冲星相关软件 | Pulsar software installation
date: 2024-08-09
author: Fxzjshm
category: Pulsar
tags: [Linux]
---

<!-- # 安装脉冲星相关软件 | Pulsar software installation -->

本文假定安装软件到 `$PREFIX`, 比如 `PREFIX=/opt/$NAME`. 当然也可将不同软件安装到同一目录, 例如 `PREFIX=$HOME/pulsar_software`. 以下内容中请把 `$PREFIX` 替换为你想要的值.  
This article assumes installing software into `$PREFIX`, e.g. `PREFIX=/opt/$NAME`. Of course you can install into same directory, e.g. `PREFIX=$HOME/pulsar_software`. Please replace `$PREFIX` with the value you want in the following content.

本文以类 Debian 环境为例, 使用其他包管理器的用户可自行搜索软件包名. 对于 RHEL, 通常 `-dev` -> `-devel`.  
This article uses Debian-like environment. Users with other package managers may search for package name by yourself. For RHEL, usually `-dev` -> `-devel`.

如有任何问题或评论, 可在 <https://github.com/fxzjshm/fxzjshm.github.io/issues> 提出.  
If you have any question or comment, consider post it at <https://github.com/fxzjshm/fxzjshm.github.io/issues>.

## 相关环境变量 | Useful environment variables

* `PATH`
* `LD_LIBRARY_PATH`
* `LIBRARY_PATH`
* `CPATH`

## 构建需要的软件 | Dependencies of building

<!-- more -->

要编译软件, 显然需要编译器, 这里以 gcc 与 gfortran 为例.  
To compile software apparently compilers are required. Here gcc and gfortran is used as example.

```bash
sudo apt install gcc gfortran g++
```

很多软件的构建系统是 Autoconf 和 CMake, 所以需要先安装好它们以便正确配置构建脚本.  
Many of these software uses Autoconf or CMake as building system, so these need to be installed so that it can configure build script properly.

```bash
sudo apt install autoconf cmake
```

## PGPLOT

* <https://sites.astro.caltech.edu/~tjp/pgplot/>
* **简而言之: 不要自己编译 PGPLOT. 用软件源里的版本.**  
  **TL;DR: Don't build PGPLOT. Use package from distro.**

### Debian-like

```bash
sudo apt install pgplot5
```

如果没找到, 可能需要在 `/etc/apt/sources.list` 中启用 `non-free` (Debian) / `multiverse` (Ubuntu):  
If not found, you may need to enable `non-free` (Debian) / `multiverse` (Ubuntu) in `/etc/apt/sources.list`:

```diff
- deb http://mirrors.pku.edu.cn/debian/ bookworm main
+ deb http://mirrors.pku.edu.cn/debian/ bookworm main contrib non-free
```

```diff
- deb https://mirrors.pku.edu.cn/ubuntu/ jammy main
+ deb https://mirrors.pku.edu.cn/ubuntu/ jammy main restricted universe multiverse
```

因为 PGPLOT 按定义不是 "自由软件".  
because PGPLOT is not "free software" by definition.

### RHEL-like

RHEL 8 & 9 官方软件源中没有 PGPLOT.  
There's no PGPLOT in official repo as of RHEL 8 & 9.

考虑在第三方软件源中搜索 PGPLOT (比如 <https://pkgs.org/search/?q=pgplot>), 风险自担.  
Consider searching third-party repo for PGPLOT (e.g. <https://pkgs.org/search/?q=pgplot>) and use it at your own risk.

<details>

### 如果不得不编译 PGPLOT, | If you have to build PGPLOT,

Credit: Heng XU, et al.

* 本段未经完全测试, 可能无法使用  
  this is not fully tested, may not work
* 使用 Debian 的 PGPLOT 的源码包里的补丁, 尤其是在 Makefile 中:  
  apply patches from Debian source package of PGPLOT, especially in Makefile:

```makefile
SHARED_LIB_LIBS="-lX11 -lpng -lc -lgfortran"
```

* 构建:  
  build:

```bash
# Note: this is not fully tested and may not work
make
make cpg
ld -shared -o libcpgplot.so --whole-archive libcpgplot.a
strip --strip-unneeded libpgplot.so
strip --strip-unneeded libcpgplot.so
```

</details>

## Tempo

* <https://tempo.sourceforge.net/>

```bash
git clone https://git.code.sf.net/p/tempo/tempo
cd tempo

# 注意到 prepare 脚本使用 csh, 而大部分系统中并不预装 csh, 故直接执行对应命令即可
# noticed that "prepare" uses csh which isn't pre-installed in most systems, so just run command inside
autoreconf --install

./configure --prefix=$TEMPO_PREFIX

# 使用 autoconf 产生的 Makefile 编译, -j 选项用于并行编译, nproc 给出机器的 CPU 核心数, 也可以手动指定
make -j`nproc`
# 安装到 $PREFIX
make install -j`nproc`
```

## PRESTO

* <https://github.com/scottransom/presto>

参考仓库中的 [INSTALL.md](https://github.com/scottransom/presto/blob/master/INSTALL.md).  
Refer to [INSTALL.md](https://github.com/scottransom/presto/blob/master/INSTALL.md) in repo.

安装依赖项: PGPLOT, FFTW 3, GLib 2, cfitsio.  
Install dependencies: PGPLOT, FFTW 3, GLib 2, cfitsio.

```bash
sudo apt install pgplot5 libfftw3-dev libglib2.0-dev libcfitsio-bin libcfitsio-dev
```

安装构建工具 meson. 这里直接使用了软件源, 一些人可能认为在虚拟环境中安装更好.  
Install building system used: meson. Here OS repo is used, some people prefer installing in Python virtual environment.

```bash
sudo apt install meson
```

然后开始构建.  
Then start building.


```bash
git clone https://github.com/scottransom/presto
cd presto
export PRESTO=`pwd`
export TEMPO=$TEMPO_PREFIX
meson setup build --prefix=$PRESTO_PREFIX
# 可能会要求安装更多依赖项, 比如
# may require installing more dependencies like: 
#          libx11-dev linpng-dev

python3 check_meson_build.py
# 可能要求设置环境变量, 例如
# may require setting environment variables, e.g.
#export PATH=$PRESTO_PREFIX/bin:$PATH
#export LIBRARY_PATH=$PRESTO_PREFIX/lib/riscv64-linux-gnu:$LIBRARY_PATH
#export LD_LIBRARY_PATH=$PRESTO_PREFIX/lib/riscv64-linux-gnu:$LD_LIBRARY_PATH
# 可将这些总结到 env.sh 中以备以后调用
# you may write these into env.sh to be sourced when later used

meson compile -C build
meson install -C build

cd python
# 在给 presto 使用的 Python 虚拟环境中
# in Python virtual environment used for presto
pip install --config-settings=builddir=build . 
```

## PSRCHIVE + DSPSR

* <https://psrchive.sourceforge.net/index.shtml>
* <https://dspsr.sourceforge.net/>

## PulsarX + TransientX

以安装到 `$PULSARX_PREFIX` 为例.  
Assume installing to `$PULSARX_PREFIX`.

安装依赖项:  
Install dependencies:

```bash
sudo apt install libboost-all-dev libtool liblapack-dev
```

注意 `libboost-dev` 不足以完整安装 Boost 库.  
Notice that `libboost-dev` is not enough for a full Boost installation.

另外, 还需安装 tempo2.  
Moreover, tempo2 also need to be installed.

PulsarX 和 TransientX 有可能不能找到它们的依赖项 (XLibs, PlotX 等), 故用 `CPATH` 和 `LIBRARY_PATH` 来绕过这个问题.  
Configure scripts of PulsarX & TransientX may not find its dependencies (XLibs, PlotX, etc.),
so `CPATH` and `LIBRARY_PATH` is used as a workaround:

```bash
# $PULSARX_PREFIX/env.sh
export YMW16_DIR=${YMW16_PREFIX}
export PATH=$TEMPO2_PREFIX/bin:$PATH

export PULSARX_PREFIX="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
export CPATH=$PULSARX_PREFIX/include:$CPATH
export LIBRARY_PATH=$PULSARX_PREFIX/lib:$LIBRARY_PATH
export LD_LIBRARY_PATH=$PULSARX_PREFIX/lib:$LD_LIBRARY_PATH
export PATH=$PULSARX_PREFIX/bin:$PATH
```

开始构建, 参考命令:  
Start building. Example command:

```bash
git clone https://github.com/ypmen/XLibs
git clone https://github.com/ypmen/PlotX
git clone https://github.com/ypmen/TransientX
git clone https://github.com/ypmen/PulsarX

cd XLibs
./bootstrap
./configure --prefix=$PULSARX_PREFIX
make -j`nproc`
make install -j`nproc`
cd ..

cd PlotX
./bootstrap
./configure --prefix=$PULSARX_PREFIX
make -j`nproc`
make install -j`nproc`
cd ..

cd TransientX
./bootstrap
./configure --prefix=$PULSARX_PREFIX
make -j`nproc`
make install -j`nproc`
cd ..

cd PulsarX
./bootstrap
./configure --prefix=$PULSARX_PREFIX
make -j`nproc`
make install -j`nproc`
cd ..
```

注意 configure 的报错很多情况下并不准确, 请查看 config.log 来确定真实原因.  
Notice that error from configure isn't accurate, refer to config.log for real situation.  
例如, "checking PGPLOT installation... library not found!" 的真实原因是:  
For example, "checking PGPLOT installation... library not found!" is actually caused by:

```
# in config.log
configure:16966: checking PGPLOT installation
configure:16983: gcc -o conftest -g -O2   conftest.c -lcpgplot -lpgplot -lcfitsio -lpng  -lgfortran -lm -lquadmath -lX11 >&5
/usr/bin/ld: cannot find -lquadmath: No such file or directory
collect2: error: ld returned 1 exit status
configure:16983: $? = 1
```

注意 PulsarX 中的构建配置对新版本 Python 缺少支持, 需要手动修改, 见 <https://github.com/ypmen/PulsarX/pull/5>.  
Notice that build config in PulsarX lacks support for new versions of Python, need to be manually patched. See <https://github.com/ypmen/PulsarX/pull/5>.

```diff
diff --git a/config/ax_python.m4 b/config/ax_python.m4
index 05efd1c..2ea0355 100644
--- a/config/ax_python.m4
+++ b/config/ax_python.m4
@@ -55,7 +55,7 @@
 AC_DEFUN([AX_PYTHON],
 [AC_MSG_CHECKING(for python build information)
 AC_MSG_RESULT([])
-for python in python3.8 python3.7 python3.6 python3.5 python3.4 python3.3 python3.2 python3.1 python3.0 python2.7 python2.6 python2.5 python2.4 python2.3 python2.2 python2.1 python; do
+for python in python3.13 python3.12 python3.11 python3.10 python3.9 python3.8 python3.7 python3.6 python3.5 python3.4 python3.3 python3.2 python3.1 python3.0 python2.7 python2.6 python2.5 python2.4 python2.3 python2.2 python2.1 python; do
 AC_CHECK_PROGS(PYTHON_BIN, [$python])
 ax_python_bin=$PYTHON_BIN
 if test x$ax_python_bin != x; then
```