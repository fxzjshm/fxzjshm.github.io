---
layout: post
title: Build upstream kernel on RHEL
date: 2023-12-23
author: Fxzjshm
category: Linux
tags: [Linux, RHEL]
---

It is known that there's `make rpm-pkg -j${nproc}` / `make deb-pkg -j${nproc}` that helps build and package Linux kernel, 
but on RHEL-like distros the default package name is still `kernel` and `kernel-devel`, causing conflict with existing kernels from repository.

So renaming the output package seems required, and here is one kind of patch:

<!-- more -->

```patch
--- a/linux-5.15.144/scripts/package/mkspec
+++ b/linux-5.15.144/scripts/package/mkspec
@@ -39,7 +39,7 @@
 #  $S: this line is enabled only when building source package
 #  $M: this line is enabled only when CONFIG_MODULES is enabled
 sed -e '/^DEL/d' -e 's/^\t*//' <<EOF
-       Name: kernel
+       Name: kernel-upstream
        Summary: The Linux Kernel
        Version: $__KERNELRELEASE
        Release: $(cat .version 2>/dev/null || echo 1)
@@ -71,12 +71,12 @@
 $S$M   Summary: Development package for building kernel modules to match the $__KERNELRELEASE kernel
 $S$M   Group: System Environment/Kernel
 $S$M   AutoReqProv: no
-$S$M   %description -n kernel-devel
+$S$M   %description devel
 $S$M   This package provides kernel headers and makefiles sufficient to build modules
 $S$M   against the $__KERNELRELEASE kernel package.
 $S$M
-$S     %prep
-$S     %setup -q
+$S     %prep -n kernel
+$S     %setup -q -n kernel-%{version}
 $S
 $S     %build
 $S     $MAKE %{?_smp_mflags} KBUILD_BUILD_VERSION=%{release}
```
