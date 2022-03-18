---
layout: post
title: Blackmoon 僵尸网络病毒处理的反馈报告
date: 2022-03-18
author: Fxzjshm
category: Other
tags: [virus, Blackmoon]
---

* 搜索关键词： Blackmoon Nidispla2.exe ssAup.exe BD17FBAC.tmp 23AD3B33.sys

## 事件核验情况
* 该同学自述几周前曾下载盗版翻译软件安装，后因觉不好用卸载
* 经任务管理器进程排查、360 附带的网速管理工具查看，确定 "Nidispla2.exe", "ssAup.exe" 两未知进程在以较小流量进行网络通讯，结合已有公告确定被感染。

## 排查过程
* 利用“打开文件所在位置”功能进入上述两进程所在文件夹 "C:\Users\\\${USERNAME}\AppData\Local\Temp\BD17FBAC.tmp"（其中 "${USERNAME}" 为用户名），发现两上述进程、一伪装成桌面配置文件的程序配置文件 "desktop.ini" 、程序日志 "~1.log"。
* 利用 Process Explorer 发现 Nidispla2.exe 是 ssAup.exe 子进程，ssAup.exe 为 explorer.exe 的子进程，遂重启 explorer.exe，但发现 ssAup.exe 自行启动，遂开始查找可能自启的位置
* 在任务管理器内启动选项卡未发现异常
* 在注册表内已知的可能自动加载程序的路径查找可疑程序，无果
* 由于计划任务无法打开，利用命令行工具 "schtasks" 对计划任务进行清理，无果
* 利用 Autologon 查找可能自行加载的位置，未发现与本次事件有关的异常
* 上述过程中程序日志 "~1.log" 一直在更新，根据日志内容使用 WinDbg 附加到 explorer.exe, 360se.exe 中，发现有无函数符号的未知代码在独立线程运行，由于汇编知识有限无法确认其功能，但通过断点发现此函数会调用 “WriteFile” 函数，怀疑是在进行日志的写入
* 上述步骤中后台使用 360 、火绒专杀先后进行扫描，未发现威胁
* 查看 "C:\Windows\System32\Drivers"，发现显示“文件夹为空”
* 利用火绒剑查看情况，发现驱动部分无法正常读取信息
* 利用火绒剑的文件功能强行进入 "C:\Windows\System32\Drivers"，发现名为 "23AD3B33.sys" 的未知驱动，但详细信息却显示为 "acpi.sys" 的信息（包括数字签名、原文件名等，但文件大小不同）
* 使用管理员权限的 cmd 删除 "23AD3B33.sys"，被拒绝访问
* 利用火绒剑强行粉碎 "23AD3B33.sys"，却发现 "acpi.sys" 被删除，遂紧急运行 "sfc /scannow" 修复但修复失败，查看修复日志发现 "acpi.sys" 的 Hash 值返回为一类似 "d45d45d45d45..." 的有规律字符串，判断其中一定有蹊跷；恰好舍友有一版本完全相同的 Windows 10 副本，从其系统中复制并再次利用火绒剑复制回原有目录；由于存在系统损坏重装的可能性，出于数据可恢复性的考虑关闭了 Bitlocker
* 在 Windows Defender 中打开“内核隔离”功能，在此过程中删除了影响此功能打开的金山公司的某一驱动
* 重启，发现仍可进入系统，且 "Nidispla2.exe" 、 "ssAup.exe" 不在运行；检查日志文件发现日志不再更新；
* 此时可正常进入 "C:\Windows\System32\Drivers" 并找到 23AD3B33.sys 本体，其数字签名显示为国内某公司签发且已于 2020 年过期，尚不清楚为何可被系统载入
* 备份后删除上述文件，静置，（期间曾短暂尝试关闭“内核隔离”功能重启），直到睡觉前未发现日志更新与上述程序运行，初步判断病毒已清除
* 由于处理完成时时间较晚且第二天有课，后续仍需进一步观察

## 损失与危害情况
* 处置过程中未发现较大流量活动；尚不清楚此主机前段时间是否已参加攻击
* "C:\Windows\System32\Drivers" 无法访问
* "C:\Users\\\${USERNAME}\AppData\Local\Temp\BD17FBAC.tmp" 内的文件以及 "C:\Windows\System32\Drivers\23AD3B33.sys"
* 未发现除此以外的影响
* **注意强行删除 "23AD3B33.sys" 会导致 "acpi.sys" 被删除进而破坏系统** （通过 retdec 反编译的代码确认 "23AD3B33.sys" 与 "acpi.sys" 有关，具体关联受限于汇编与 Windows 编程的知识有限尚不清楚）

## 协调处置过程
* 见上文，在 Windows Defender 中打开“内核隔离”功能，并在重启后删除 "C:\Windows\System32\Drivers\23AD3B33.sys" (注意 "acpi.sys" 的备份) 与 "C:\Users\\\${USERNAME}\AppData\Local\Temp\BD17FBAC.tmp"（其中 "${USERNAME}" 为用户名）

## 涉事系统运营单位
* (Todo)

## 整改情况
* 该同学承诺不再下载来路不明的软件
* (Todo)