---
layout: post
title: Enable FSBL_DEBUG in recent versions of Petalinux
date: 2025-01-18
author: Fxzjshm
category: FPGA
tags: [Linux, FPGA, Vivado, Petalinux]
---

## Problem

For debugging RFSoC booting from SD card, the `FSBL_DEBUG` is helpful (See [official doc](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842019/Zynq+UltraScale+FSBL), search for "debug").

Some of Q&As e.g. ["How to enable FSBL customization in petalinux"](https://adaptivesupport.amd.com/s/question/0D52E00006hpcg7SAA/how-to-enable-fsbl-customization-in-petalinux?language=en_US), and even official doc ["Zynq UltraScale+ MPSoC Software Developer Guide (UG1137)"](https://docs.xilinx.com/r/en-US/ug1137-zynq-ultrascale-mpsoc-swdev/Modifying-Component-Recipes), give a method:

```bash
mkdir -p <plnx-proj-root>/project-spec/meta-user/recipes-bsp/fsbl
vim <plnx-proj-root>/project-spec/meta-user/recipes-bsp/fsbl/fsbl_%.bbappend
```

File content:

```
#Enable appropriate FSBL debug flags
YAML_COMPILER_FLAGS_append = " -DFSBL_DEBUG"
```

However this doesn't work in recent versions of Petalinux, e.g. 2024.1.

## Analyze

It is easily noticed that other .bbappend files uses a different syntax, e.g. `recipes-bsp/u-boot/u-boot-xlnx_%.bbappend` has a line like:

```
SRC_URI:append = " file://platform-top.h file://bsp.cfg"
```

So the `fsbl_%.bbappend` should be

```
# Enable appropriate FSBL debug flags
YAML_COMPILER_FLAGS_append = " -DFSBL_DEBUG "
```

But this still not working, `petalinux-build` gives warning:

```
WARNING: No recipes in default available for:
  /<plnx-proj-root>/project-spec/meta-user/recipes-bsp/fsbl/fsbl_%.bbappend
```

At this point, search engine gives me another Q&A ["PetaLinux 2022.2: Recipes modifying FSBL: doesn't seem to be taken into account during petalinux-build"](https://adaptivesupport.amd.com/s/question/0D54U00006qVXtpSAG/petalinux-20222-recipes-modifying-fsbl-doesnt-seem-to-be-taken-into-account-during-petalinuxbuild?language=en_US); and also found a similar line in `components/yocto/layers/meta-kria/recipes-bsp/embeddedsw/fsbl-firmware_%.bbappend`:

```
YAML_COMPILER_FLAGS:append:kria ?= " -DFSBL_DEBUG"
```

## Solution

```bash
mkdir -p project-spec/meta-user/recipes-bsp/embeddedsw
nano project-spec/meta-user/recipes-bsp/embeddedsw/fsbl-firmware_%.bbappend
```

File content:

```
# Enable appropriate FSBL debug flags
YAML_COMPILER_FLAGS:append = " -DFSBL_DEBUG "
```

or alternatively, set it in `petalinux-config`'s FSBL config.

Notice that

```bash
petalinux-build -x mrproper
```

seems required for the flag to work, otherwise a wired "empty variable name" error from `make` may happen:

```
Log data follows:
| DEBUG: Executing shell function do_compile
| NOTE: make -j1
| make -C zynqmp_fsbl_bsp
| make[1]: Entering directory '<plnx-proj-root>/build/tmp/work/zynqmp_generic_xczu47dr-xilinx-linux/fsbl-firmware/2024.1+gitAUTOINC+b173d24682-r0/git/fsbl-firmware/fsbl-firmware/zynqmp_fsbl_bsp'
| make --no-print-directory seq_libs
| Running Make include in psu_cortexa53_0/libsrc/avbuf_v2_6/src
| make -C psu_cortexa53_0/libsrc/avbuf_v2_6/src -s include  "SHELL=/bin/sh" "COMPILER=aarch64-none-elf-gcc" "ASSEMBLER=aarch64-none-elf-as" "ARCHIVER=aarch64-none-elf-ar" "COMPILER_FLAGS=  -c" "EXTRA_COMPILER_FLAGS=-DFSBL_DEBUG "YAML_COMPILER_FLAGS:append:pn-fsbl-firmware = "  -DFSBL_DEBUG -Os -flto -ffat-lto-objects"
| make[1]: Leaving directory '<plnx-proj-root>/build/tmp/work/zynqmp_generic_xczu47dr-xilinx-linux/fsbl-firmware/2024.1+gitAUTOINC+b173d24682-r0/git/fsbl-firmware/fsbl-firmware/zynqmp_fsbl_bsp'
| make: *** empty variable name.  Stop.
| make[2]: *** [Makefile:42: psu_cortexa53_0/libsrc/avbuf_v2_6/src/make.include] Error 2
| make[1]: *** [Makefile:18: all] Error 2
| make: *** [Makefile:32: zynqmp_fsbl_bsp/psu_cortexa53_0/lib/libxil.a] Error 2
| ERROR: oe_runmake failed
```
