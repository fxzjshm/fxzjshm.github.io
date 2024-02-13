---
layout: post
title: "Workaround Debian package dependency: fix `wine-devel` package rename on Ubuntu 22.04"
date: 2024-02-13
author: Fxzjshm
category: Linux
tags: [Linux, Ubuntu]
---

The most recent version of Wine is named `wine-development` in Ubuntu 22.04 
repository, while in WineHQ repository it is named `wine-devel` [^1].
This causes problem when installing packages depending on Wine and want the 
most recent version, e.g. `carla-bridge-wine32`.

One workaround is to use a dummy package using "equivs":
1. install equivs
2. run `equivs-control wine-devel-fixes`
3. edit control file, related lines:
```
Package: wine-devel-fixes
Depends: wine-devel
Provides: wine-development
Description: Fixes the Wine package rename on Ubuntu 22.04
```
4. run `equivs-build wine-devel-fixes`
5. install the generated `wine-devel-fixes_1.0_all.deb`

Reference: workaround ROCm installation on Ubuntu 22.04 (when it didn't support): <https://github.com/ROCm/ROCm/issues/1730#issuecomment-1205109513>

<!-- more -->

[^1]: not RHEL's `-devel` suffix
