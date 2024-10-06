---
layout: post
title: 在 CASA 里添加台站 | Add observatory in CASA
date: 2024-10-04
author: Fxzjshm
category: Astronomy
tags: [Astronomy]
---

当导入数据到 CASA (_Common Astronomy Software Applications_) 时,
若台站不在 CASA 数据库中, 会得到如下报错:

```
RuntimeError: Exception: Telescope 21CMA is not recognized by CASA.
... thrown by casacore::MPosition casacore::MSMetaData::getObservatoryPosition(casacore::uInt) const at File: /source/casa6/casatools/casacore/ms/MSOper/MSMetaData.cc, line: 3305
```

从报错位置 [MSMetaData.cc#L3305](https://github.com/casacore/casacore/blob/bf9e411892b3816724eb11737cc32d12f86a6436/ms/MSOper/MSMetaData.cc#L3305) 可以追踪到 [MeasTable.cc#L2905](https://github.com/casacore/casacore/blob/bf9e411892b3816724eb11737cc32d12f86a6436/measures/Measures/MeasTable.cc#L2905), [MeasTable.cc#L2863](https://github.com/casacore/casacore/blob/bf9e411892b3816724eb11737cc32d12f86a6436/measures/Measures/MeasTable.cc#L2863), 得到关键词 "Observatories",
"measures.observatory.directory", "geodetic".

经过一些搜索后注意到该文件是 casacore 下载得到的, 通常位于 `~/.casa/data/geodetic/Observatories`, 并且是类似 measurement set 的结构, 那么可用 Table 相关函数编辑.

注意 CASA 启动会加载该文件, 也就是加上读锁, 然后就无法获得写锁 (会死锁, 表现为程序卡死), 可以复制一份为 `Observatories-bak` 作为备份, 再复制一份为 `Observatories-edit` 编辑.

在 CASA 中使用

```python
browsetable(tablename="/home/user/.casa/data/geodetic/Observatories-edit")
```

编辑加入台站参数即可. 记得退出 CASA 之后把 `Observatories-edit` 改回 `Observatories`.
