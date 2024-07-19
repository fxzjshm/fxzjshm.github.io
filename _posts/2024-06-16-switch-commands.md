---
layout: post
title: Switch commands
date: 2024-06-16
author: Fxzjshm
category: Linux
tags: [Linux]
---

Device: Dell Powerswitch w/ BMC OS10

help:
```
?
```

enter config mode:
```
config
```

enter certain port group:
```
port-group 1/1/11
```

change port mode:
```
port 1/1/21 mode Eth 100g-1x
```
```
port 1/1/21 mode Eth 25g-4x
```

show current config:
```
do show interface status
```

change MTU of multiple ports
```
interface range ethernet 1/1/1-1/1/32
mtu 9216
```

save config:
```
do write memory
```
