---
layout: post
title: 一些服务器知识 / Some server tricks
date: 2023-03-04
author: Fxzjshm
category: Other
tags: [Linux]
---

Credit: Chenchen Miao and Cijie Zhang

## 1. 利用 HPE 的 iLO 功能实现远程操作服务器 / operate the server remotely using HPE's iLO
1. 记下机箱标签上的 iLO 账号与密码 / remember username and password of iLO which is on the label of server
![](/img/2023/some-server-tricks/1677858241518.avif)
2. 第一次启动需要接上屏幕, 在引导阶段会显示 iLO 的 IP 地址 / screen seems required on first boot, IP address of iLO will show when booting
3. 用其他设备以 HTTP(S) 访问该地址, 以上述账号密码登录即可使用 iLO 远程操作与监控该服务器 / visit this address via HTTP(s) protocol using another device, and then login with username and password said above to operate and monitor this server remotely
![](/img/2023/some-server-tricks/Screenshot_20230303_162619.png)


## 2. 某些 RAID 卡的设置的位置 / Location of settings of some RAID card
1. 启动时 F9 进入 System Utilities / press F9 when booting to enter System Utilities
![](/img/2023/some-server-tricks/raid_setting_0.png)
2. 其余见图 / further steps refer to these images
<div align="center">
<img src="/img/2023/some-server-tricks/raid_setting_1.png" width = "50%" />
</div>
<div align="center">
<img src="/img/2023/some-server-tricks/raid_setting_2.png" width = "50%" />
</div>
<div align="center">
<img src="/img/2023/some-server-tricks/raid_setting_3.png" width = "50%" />
</div>
