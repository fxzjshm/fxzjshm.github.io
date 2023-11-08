---
layout: post
title: Access git remote in private network
date: 2023-11-09
author: Fxzjshm
category: Linux
tags: [Linux]
---

情况: Git 远程服务器 (例如 GitLab 实例) 位于内网, 需要能从外部访问.

Situation: Git remote server (e.g. GitLab instance) is in private network but need to access from outside.

一种解决方案: 用 ssh 打开隧道, 例如

One solution: open tunnel using ssh, like

```
Host jumper-machine
  # ...
  LocalForward 3002 10.1.2.233:80
  LocalForward 3003 10.1.2.233:33322
```

在用 `ssh jumper-machine` 开启隧道后,
即可使用 `http://localhost:3002` 访问 Git 远程服务器的网页, 
`ssh://git@127.0.0.1:3003/<user>/<repo>` 可作为 Git 仓库的远程地址.

After opening tunnel using `ssh jumper-machine`,
use `http://localhost:3002` to visit webpage of Git remote server,
use `ssh://git@127.0.0.1:3003/<user>/<repo>` as Git remote url.

<!-- more -->
