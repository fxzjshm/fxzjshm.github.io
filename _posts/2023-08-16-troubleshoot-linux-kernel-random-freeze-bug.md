---
layout: post
title: 排查 Linux 内核随机卡死问题 / Troubleshoot Linux kernel random freeze bug
date: 2023-08-16
author: Fxzjshm
category: Other
tags: [Linux]
---

Translation below.

### 问题描述
某 2021 年还安装 CentOS 7 的服务器近期时常出现无响应、远程无法连接, 键鼠输入有延迟, 且有内核出错记录:

```
Aug 11 12:45:31 <machine name> kernel: warn alloc falled: 1380491 callbacks suppressed
Aug 11 12:45:31 <machine name> kernel: swapper/27: page allocation failure: order 0, mode: 0x20
Aug 11 12:45:31 <machine name> kernel: CPU: 27 PID: 0 Comm: swapper/27 Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160 62.1.el7.x86 64 #1
Aug 11 12:45:31 <machine name> kernel: Hardware name: <name>
Aug 11 12:45:31 <machine name> kernel: Call Trace:
Aug 11 12:45:31 <machine name> kernel: <IRQ>  [<ffffffffbcb865a9>] dump_stack+0x19/0x1b
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c4bd0>] warn alloc failed+0x110/0x180
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc4d3383>] ? __wake up+0x13/0x20
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c976f>] __alloc_pages_nodemask+0x9df/oxbe0
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c9b78>] page_frag_alloc+0x158/0x170
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbca4808d>] __napi_alloc_skb+0x8d/0xe0
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03df895>] mlx5e_skb_from_cqe_mpwrq_nonlinear+0x65/0x2f0 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03dfdbf>] mlx5e_handle_rx_cqe_mpwrq+0xbf/0x8c0 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: swapper/57: page allocation failure: order 0, mode: 0x20
Aug 11 12:45:31 <machine name> kernel: CPU: 57 PID: 0 Comm: swapper/57 Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160 62.1.el7.x86 64 #1
Aug 11 12:45:31 <machine name> kernel: Hardware name: <name>
Aug 11 12:45:31 <machine name> kernel: Call Trace:
Aug 11 12:45:31 <machine name> kernel: <IRQ>  [<ffffffffbcb865a9>] dump_stack+0x19/0x1b
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03e14be>] mlx5e_napi_poll+0xbe/0xd10 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc78da24>] ? __radix_tree_lookup+0x84/0xf0
......
```

重启后可恢复一段时间, 但数小时到数天后仍会卡死.

### 问题排查
<!-- more -->
* `dmesg` 与 `journalctl` 日志未发现异常
* GUI 中鼠标输入缓慢, 按下键盘 Num lock 到对应灯切换状态有延迟, 怀疑与内核中断有关
* 停止所有观测和处理任务, 键鼠输入仍然卡顿, 则不是因为负载过高
* 由于内核报错与网卡有关, 断开与数据链路上游的连接, 无变化
* 想起另有一台 CentOS 7 服务器也出现过 ~10 天未使用后发现无法远程连接的问题, 继续怀疑与内核有关
* 搜索 "CentOS 7 kernel free" 得到一些链接:
  * CentOS 7 freezing -- Unix & Linux Stack Exchange <https://unix.stackexchange.com/questions/534010/centos-7-freezing/558093#558093>
  * freezes on CentOS 7 -- CentOS <https://forums.centos.org/viewtopic.php?t=51455>
    * 其中的链接 <http://bugs.centos.org/view.php?id=7414> 似乎无法访问?
* 由上述链接, 准备先尝试升级内核
* 查看内核更改记录时, 注意到一些更改似乎与上述栈展开信息有关:
  * kernel-3.10.0-1160.95.1.el7.x86_64.rpm CentOS 7 Download @ pkgs.org <https://centos.pkgs.org/7/centos-updates-x86_64/kernel-3.10.0-1160.95.1.el7.x86_64.rpm.html>
```
2023-02-02 - Rado Vrbovsky <rvrbovsk@redhat.com> [3.10.0-1160.86.1.el7]
...
- mm: prevent page_frag_alloc() from corrupting the memory (Rafael Aquini) [2141062]
- mm: Use fixed constant in page_frag_alloc instead of size + 1 (Rafael Aquini) [2141062]
- mm: page_alloc: fix ref bias in page_frag_alloc() for 1-byte allocs (Rafael Aquini) [2141062]
...
```
* 搜索 "page_frag_alloc" 发现此问题的起源
  * Bug 2104445 - RHEL9.1: in low memory conditions, page_frag_alloc may corrupt the memory.  <https://bugzilla.redhat.com/show_bug.cgi?id=2104445>
* 使用其中的复现内核模块可以复现问题
```bash
sudo insmod oomk.ko memory_size_gb=300 fragsize=4097
```
```
[74442.580044] Test begins, memory size = 300 fragsize = 4097
[74448.091139] BUG: Bad page state in process insmod  pfn:3cdf8cb
[74448.091144] page:fffff701f37e32c0 count:0 mapcount:1 mapping:          (null) index:0x15937
[74448.091145] page flags: 0x6fffff00080068(uptodate|lru|active|swapbacked)
[74448.091149] page dumped because: PAGE_FLAGS_CHECK_AT_FREE flag(s) set
[74448.091150] bad because of flags:
[74448.091150] page flags: 0x60(lru|active)
[74448.091151] Modules linked in: oomk(OE+) nvidia_uvm(OE) ...
[74448.091181]  ...
[74448.091209] CPU: 59 PID: 31416 Comm: insmod Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160.62.1.el7.x86_64 #1
[74448.091210] Hardware name: <name>
[74448.091211] Call Trace:
[74448.091218]  [<ffffffffa33865a9>] dump_stack+0x19/0x1b
[74448.091222]  [<ffffffffa33818e5>] bad_page.part.75+0xdc/0xf9
[74448.091226]  [<ffffffffa2dc6f82>] free_pages_prepare+0x1f2/0x210
[74448.091228]  [<ffffffffa2dc72b9>] __free_pages_ok+0x19/0xc0
[74448.091229]  [<ffffffffa2dc7431>] page_frag_free+0x61/0x80
[74448.091231]  [<ffffffffc08d10b7>] oomk_init+0xb7/0x1000 [oomk]
[74448.091234]  [<ffffffffc08d1000>] ? 0xffffffffc08d0fff
[74448.091238]  [<ffffffffa2c0210a>] do_one_initcall+0xba/0x240
[74448.091242]  [<ffffffffa2d1ea7a>] load_module+0x271a/0x2bb0
[74448.091245]  [<ffffffffa2fb4cd0>] ? ddebug_proc_write+0x100/0x100
[74448.091248]  [<ffffffffa2e50b73>] ? fput+0x13/0x20
[74448.091249]  [<ffffffffa2d1a603>] ? copy_module_from_fd.isra.44+0x53/0x150
[74448.091251]  [<ffffffffa2d1f0f6>] SyS_finit_module+0xa6/0xd0
[74448.091253]  [<ffffffffa3399f92>] system_call_fastpath+0x25/0x2a
[74448.091255] BUG: Bad page state in process insmod  pfn:3cdf8ca
[74448.091255] page:fffff701f37e3280 count:0 mapcount:1 mapping:          (null) index:0x15957
[74448.091256] page flags: 0x6fffff00080068(uptodate|lru|active|swapbacked)
[74448.091258] page dumped because: PAGE_FLAGS_CHECK_AT_FREE flag(s) set
[74448.091258] bad because of flags:
[74448.091259] page flags: 0x60(lru|active)
[74448.091260] Modules linked in: ...
```

### 问题解决
* 升级内核到 `sudo insmod oomk.ko memory_size_gb=3 fragsize=4097` 并重启, 用此模块确认这个问题已经修复
* 目前尚无进一步的异常

---

### Description
Some server with CentOS 7 installed in 2021 sometimes cannot respond and cannot be connected remotely, mouse & keyboard inputs are laggy, with kernel error log:

```
Aug 11 12:45:31 <machine name> kernel: warn alloc falled: 1380491 callbacks suppressed
Aug 11 12:45:31 <machine name> kernel: swapper/27: page allocation failure: order 0, mode: 0x20
Aug 11 12:45:31 <machine name> kernel: CPU: 27 PID: 0 Comm: swapper/27 Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160 62.1.el7.x86 64 #1
Aug 11 12:45:31 <machine name> kernel: Hardware name: <name>
Aug 11 12:45:31 <machine name> kernel: Call Trace:
Aug 11 12:45:31 <machine name> kernel: <IRQ>  [<ffffffffbcb865a9>] dump_stack+0x19/0x1b
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c4bd0>] warn alloc failed+0x110/0x180
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc4d3383>] ? __wake up+0x13/0x20
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c976f>] __alloc_pages_nodemask+0x9df/oxbe0
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc5c9b78>] page_frag_alloc+0x158/0x170
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbca4808d>] __napi_alloc_skb+0x8d/0xe0
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03df895>] mlx5e_skb_from_cqe_mpwrq_nonlinear+0x65/0x2f0 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03dfdbf>] mlx5e_handle_rx_cqe_mpwrq+0xbf/0x8c0 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: swapper/57: page allocation failure: order 0, mode: 0x20
Aug 11 12:45:31 <machine name> kernel: CPU: 57 PID: 0 Comm: swapper/57 Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160 62.1.el7.x86 64 #1
Aug 11 12:45:31 <machine name> kernel: Hardware name: <name>
Aug 11 12:45:31 <machine name> kernel: Call Trace:
Aug 11 12:45:31 <machine name> kernel: <IRQ>  [<ffffffffbcb865a9>] dump_stack+0x19/0x1b
Aug 11 12:45:31 <machine name> kernel: [<ffffffffc03e14be>] mlx5e_napi_poll+0xbe/0xd10 [mlx5_core]
Aug 11 12:45:31 <machine name> kernel: [<ffffffffbc78da24>] ? __radix_tree_lookup+0x84/0xf0
......
```

It can operate normally after reboot, but still freeze after several hours or days.

### Investigation
<!-- more -->
* No unexpented things found in `dmesg` and `journalctl`
* Mouse input in GUI is laggy; there's delay from pressing Num Lock in keyboard to correspond LED switch state, so maybe related to kernel interrupt
* All observation and data processing are terminated but mouse & keyboard inputs are still laggy, so not related to high load of server
* As kernel error log related to network interface, disconnect from upstream data source, no change
* Realized that another CentOS 7 server was also unresponsive (cannot remotely connect) after not used for ~10 days, still suspect related to kernel
* Search "CentOS 7 kernel free" and get some links:
  * CentOS 7 freezing -- Unix & Linux Stack Exchange <https://unix.stackexchange.com/questions/534010/centos-7-freezing/558093#558093>
  * freezes on CentOS 7 -- CentOS <https://forums.centos.org/viewtopic.php?t=51455>
    * The link <http://bugs.centos.org/view.php?id=7414> in that page seems unable to connect?
* Based on these links, going to upgrade kernel
* When reading kernel change log, notice some information that related to kernel error stack trace above :
  * kernel-3.10.0-1160.95.1.el7.x86_64.rpm CentOS 7 Download @ pkgs.org <https://centos.pkgs.org/7/centos-updates-x86_64/kernel-3.10.0-1160.95.1.el7.x86_64.rpm.html>
```
2023-02-02 - Rado Vrbovsky <rvrbovsk@redhat.com> [3.10.0-1160.86.1.el7]
...
- mm: prevent page_frag_alloc() from corrupting the memory (Rafael Aquini) [2141062]
- mm: Use fixed constant in page_frag_alloc instead of size + 1 (Rafael Aquini) [2141062]
- mm: page_alloc: fix ref bias in page_frag_alloc() for 1-byte allocs (Rafael Aquini) [2141062]
...
```
* Search "page_frag_alloc" then find original bug report
  * Bug 2104445 - RHEL9.1: in low memory conditions, page_frag_alloc may corrupt the memory.  <https://bugzilla.redhat.com/show_bug.cgi?id=2104445>
* Reproduce this bug using attached kernel mod:
```bash
sudo insmod oomk.ko memory_size_gb=300 fragsize=4097
```
```
[74442.580044] Test begins, memory size = 300 fragsize = 4097
[74448.091139] BUG: Bad page state in process insmod  pfn:3cdf8cb
[74448.091144] page:fffff701f37e32c0 count:0 mapcount:1 mapping:          (null) index:0x15937
[74448.091145] page flags: 0x6fffff00080068(uptodate|lru|active|swapbacked)
[74448.091149] page dumped because: PAGE_FLAGS_CHECK_AT_FREE flag(s) set
[74448.091150] bad because of flags:
[74448.091150] page flags: 0x60(lru|active)
[74448.091151] Modules linked in: oomk(OE+) nvidia_uvm(OE) ...
[74448.091181]  ...
[74448.091209] CPU: 59 PID: 31416 Comm: insmod Kdump: loaded Tainted: P           OE  ------------   3.10.0-1160.62.1.el7.x86_64 #1
[74448.091210] Hardware name: <name>
[74448.091211] Call Trace:
[74448.091218]  [<ffffffffa33865a9>] dump_stack+0x19/0x1b
[74448.091222]  [<ffffffffa33818e5>] bad_page.part.75+0xdc/0xf9
[74448.091226]  [<ffffffffa2dc6f82>] free_pages_prepare+0x1f2/0x210
[74448.091228]  [<ffffffffa2dc72b9>] __free_pages_ok+0x19/0xc0
[74448.091229]  [<ffffffffa2dc7431>] page_frag_free+0x61/0x80
[74448.091231]  [<ffffffffc08d10b7>] oomk_init+0xb7/0x1000 [oomk]
[74448.091234]  [<ffffffffc08d1000>] ? 0xffffffffc08d0fff
[74448.091238]  [<ffffffffa2c0210a>] do_one_initcall+0xba/0x240
[74448.091242]  [<ffffffffa2d1ea7a>] load_module+0x271a/0x2bb0
[74448.091245]  [<ffffffffa2fb4cd0>] ? ddebug_proc_write+0x100/0x100
[74448.091248]  [<ffffffffa2e50b73>] ? fput+0x13/0x20
[74448.091249]  [<ffffffffa2d1a603>] ? copy_module_from_fd.isra.44+0x53/0x150
[74448.091251]  [<ffffffffa2d1f0f6>] SyS_finit_module+0xa6/0xd0
[74448.091253]  [<ffffffffa3399f92>] system_call_fastpath+0x25/0x2a
[74448.091255] BUG: Bad page state in process insmod  pfn:3cdf8ca
[74448.091255] page:fffff701f37e3280 count:0 mapcount:1 mapping:          (null) index:0x15957
[74448.091256] page flags: 0x6fffff00080068(uptodate|lru|active|swapbacked)
[74448.091258] page dumped because: PAGE_FLAGS_CHECK_AT_FREE flag(s) set
[74448.091258] bad because of flags:
[74448.091259] page flags: 0x60(lru|active)
[74448.091260] Modules linked in: ...
```

### Solution
* Upgrade kernel to `sudo insmod oomk.ko memory_size_gb=3 fragsize=4097` and reboot, confirm this bug is fixed using test kernel module
* No further error noticed till now
