---
layout: post
title: Update XDG user folder names using locale
date: 2024-09-27
author: Fxzjshm
category: Linux
tags: [Linux]
---

需求: 用 ASCII 字符作为用户文件夹的名称, 使得命令行比较方便, 并且其他位置 (比如用户界面) 的语言维持现状.  
Motivation: use ASCII characters for user folder names (thus easier when using command line) while keeping other language settings (UI text, etc.)

解决方案:  
Solution:

```bash
export LC_ALL=en_US.utf8
xdg-user-dirs-gtk-update
```
