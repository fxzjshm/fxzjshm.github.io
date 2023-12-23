---
layout: post
title: Build KDE 6 on Ubuntu 22.04
date: 2023-12-09
author: Fxzjshm
category: Linux
tags: [Linux, KDE]
---


To build KDE 6 on Ubuntu 22.04, other than steps on <https://community.kde.org/Get_Involved/development/Build_software_with_kdesrc-build>,
there are other things need to be noticed:
* Qt6 in apt repo is not new enough, should let kdesrc-build build Qt6; follow instruction at <https://community.kde.org/Get_Involved/development/More#Build_Qt6_using_kdesrc-build>, and notice some packages are required to build some parts (see below)
* gpgme fail to build, check <https://community.kde.org/Get_Involved/development/More#kdesrc-build_issues>, or use environment variable to help pkg-config find Qt 6. Notice this should point to installed location, like

```bash
export PKG_CONFIG_PATH=/opt/kde/usr/lib/pkgconfig:$PKG_CONFIG_PATH
```

<!-- more -->

* KWin requires `libdisplay-info`, install it from newer repo, e.g. Debian Sid; or check the link above
* KWin requires Qt6 built with `atspi_accessibility`, which requires `ATSPI2`, so need to install `libatspi2.0-dev` before build Qt6
* plasma-desktop: SDL2 in Ubuntu 22.04 not new enough to have `SDL_JoystickPath`, disable game controller support in `kcms/CMakeLists.txt`
* xdg-desktop-portal-kde requires `QtPrintSupport/private/qcups_p.h`, so install `libcups2-dev` before building Qt6
