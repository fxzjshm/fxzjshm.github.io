---
layout: post
title: 将 SNAP 配置程序移植到龙芯派 | Port SNAP configure program to Loongson Pi
date: 2025-07-18
author: Fxzjshm
category: Radio Astronomy
tags: [Linux, CASPER, SNAP]
---

## 0. 引言

SNAP (Smart Network ADC Processor) 是 CASPER 开发的低成本采样装置, 更多信息可见 <https://github.com/casper-astro/casper-hardware/blob/master/FPGA_Hosts/SNAP/README.md>. 通常情况下需要使用树莓派来驱动 SNAP, 但一些已有的镜像对应的树莓派版本已经不再销售, 可考虑其他单板计算机来控制 SNAP, 例如龙芯 2K0300 先锋派.

龙芯 2K0300 先锋派的信息参见 <https://www.loongson.cn/news/show?id=688>. 选择此板的主要原因为 GPIO 兼容, 省去了重新画电路板适配 GPIO 的过程.

## 1. 下载系统

龙芯先锋派的系统、内核等信息可从以下链接获取:

* 根文件系统、内核二进制与源码下载: <https://pan.baidu.com/s/1FZcnFcmTGd5GcyP8ayFm5A?pwd=1234>
* 文档: <https://gitee.com/open-loongarch/docs-2k0300>
* 内核源码: <https://gitee.com/open-loongarch/linux-6.12>

为了方便安装依赖, 这里使用有包管理器的、新世界 (新 ABI) 的 Alpine 作为基本系统.
在 SD 卡中创建 ext4 分区, 将根文件系统解压并复制进去即可. 为了确保功能正常, 可以将内核手动替换为新版本.

注意 Alpine 3.21 开始支持龙架构, 但 Alpine 3.22 的预编译程序运行时出现 illegal instruction 的报错,
因此似乎只能使用 Alpine 3.21.

然后卸载 SD 卡, 装入龙芯先锋派并启动.

## 2. 安装 CASPER 相关软件

### 2.1 casperfpga
<!-- more -->
```bash
git clone https://github.com/casper-astro/casperfpga
cd casperfpga
git checkout py310-dev  # or py27, py38-dev, ...
```
经验上 `py27` 分支较为稳定, 但较新的系统可能已经不提供 Python 2.7;
`py38-dev` 与 `py310-dev` 较新, 但存在变量/函数命名前后不一致、接口更改等问题,
可能需要手动修复 bug, 已有代码可能需要移植; 根据一些反馈, `py38` 分支可能存在问题.
对于 `py310-dev` 中存在的部分语法问题, 已将个人已知的更改上传到 <https://github.com/fxzjshm/casperfpga/tree/fix-syntax>.

根据习惯和需求, 进入 Python 虚拟环境用 pip 安装, 或用 pip 全局安装;
若可能对源码进行修改, 推荐使用

```bash
pip install -e .
```

注意安装 casperfpga 时自动下载的 tornado 较老,
并不适配 Python 3.12, 因此需要

```bash
pip uninstall tornado
sudo apk add py3-tornado
```

由于龙芯先锋派烧写 FPGA 固件较慢, 运行时可能出现超时, 需要手动调大超时阈值; 还需要修复一些语法错误:

```diff
diff --git a/src/snapadc.py b/src/snapadc.py
index 0a4682e..e8ad765 100644
--- a/src/snapadc.py
+++ b/src/snapadc.py
@@ -884,7 +884,7 @@ class SnapAdc(object):
                         h[chip,t] = (np.sum(d == d[:1], axis=(0,1)) == d.shape[0] * d.shape[1])
                 self.selectADC() # select all chips
                 self.adc.test('off')
-                self.setDemux(numChannel=self.numChannel)
+                self.setDemux(numChannel=self.num_channel)
                 ker = np.ones(ker_size)
                 for chip in self.adcList:
                     # identify taps that work for all lanes of chip
diff --git a/src/transport_katcp.py b/src/transport_katcp.py
index b5cd2f2..794351f 100644
--- a/src/transport_katcp.py
+++ b/src/transport_katcp.py
@@ -580,7 +580,7 @@ class KatcpTransport(Transport, katcp.CallbackClient):
                                      request_args=(version, ),
                                      require_ok=True)
 
-    def upload_to_ram_and_program(self, filename, port=-1, timeout=10,
+    def upload_to_ram_and_program(self, filename, port=-1, timeout=30,
                                   wait_complete=True,
                                   skip_verification=False, **kwargs):
         """
```


### 2.2 katcp
注意, 目前已知仅有 <https://github.com/jack-h/katcp_devel/tree/rpi-devel-casperfpga> 处的 katcp 可支持树莓派,
```bash
git clone https://github.com/jack-h/katcp_devel.git
cd katcp_devel
git checkout rpi-devel-casperfpga
```

GPIO 相关的参数可能需要调整, 比如外设的内存地址,
树莓派的参见 [`e95899f`](https://github.com/jack-h/katcp_devel/commit/e95899ff5dee5f855b6d795d2b6543c708c6e508).
龙芯派的 GPIO 芯片基地址可从 device tree 看到, 从
<https://gitee.com/open-loongarch/linux-6.12/blob/master/arch/loongarch/boot/dts/loongson-2k0300.dtsi#L683>
能看到基地址为 0x16104000;
GPIO 编号可从原理图和芯片说明对比得到, 并注意到对应的设备是 spidev2;
GPIO 的操作方式可参考其驱动 <https://gitee.com/open-loongarch/linux-6.12/blob/master/drivers/gpio/gpio-loongson-irq.c>.

实际测试中发现 SPI 一次传输仅能获得至多 2 个回复, 并不清楚是什么导致的, 暂且使用 MAX_BURST_SIZE = 1 来规避.

```diff
diff --git a/tcpborphserver3/rpjtag_io.c b/tcpborphserver3/rpjtag_io.c
index f8b7302..cd82b4c 100644
--- a/tcpborphserver3/rpjtag_io.c
+++ b/tcpborphserver3/rpjtag_io.c
@@ -13,7 +13,8 @@ int  mem_fd;
 void *gpio_map;
 
 // I/O access
-volatile unsigned *gpio;
+// volatile unsigned *gpio;
+volatile unsigned char *gpio;
 
 void tick_clk()
 {
@@ -54,7 +55,7 @@ int setup_io()
     }
     
     // Always use volatile pointer!
-    gpio = (volatile unsigned *)gpio_map;
+    gpio = (volatile unsigned char *)gpio_map;
     
     INP_GPIO(JTAG_TCK);
     INP_GPIO(JTAG_TMS);
diff --git a/tcpborphserver3/rpjtag_io.h b/tcpborphserver3/rpjtag_io.h
index 1bac3cf..9acd618 100644
--- a/tcpborphserver3/rpjtag_io.h
+++ b/tcpborphserver3/rpjtag_io.h
@@ -1,6 +1,7 @@
 #ifndef RPJTAG_IO_H
 #define RPJTAG_IO_H
 
+#include <stdint.h>
 #include <stdlib.h>
 #include <sys/mman.h>
 #include <stdio.h>
@@ -11,31 +12,94 @@
 #define PAGE_SIZE (4*1024)
 #define BLOCK_SIZE (4*1024)
 
-//#define RPI_VERSION 4
-#if RPI_VERSION >= 4
-    #define BCM2708_PERI_BASE 0xFE000000
-#elif RPI_VERSION >= 2
-    #define BCM2708_PERI_BASE 0x3F000000
-#else
-    #define BCM2708_PERI_BASE 0x20000000
-#endif
-#define GPIO_BASE                (BCM2708_PERI_BASE + 0x200000) /* GPIO controller */
+// //#define RPI_VERSION 4
+// #if RPI_VERSION >= 4
+//     #define BCM2708_PERI_BASE 0xFE000000
+// #elif RPI_VERSION >= 2
+//     #define BCM2708_PERI_BASE 0x3F000000
+// #else
+//     #define BCM2708_PERI_BASE 0x20000000
+// #endif
+// #define GPIO_BASE                (BCM2708_PERI_BASE + 0x200000) /* GPIO controller */
+
+// // GPIO setup macros. Always use INP_GPIO(x) before using OUT_GPIO(x) or SET_GPIO_ALT(x,y)
+// #define INP_GPIO(g) *(gpio+((g)/10)) &= ~(7<<(((g)%10)*3))
+// #define OUT_GPIO(g) *(gpio+((g)/10)) |=  (1<<(((g)%10)*3))
+// #define SET_GPIO_ALT(g,a) *(gpio+(((g)/10))) |= (((a)<=3?(a)+4:(a)==4?3:2)<<(((g)%10)*3))
+
+// #define GPIO_SET(g) *(gpio+7) = 1<<(g) // sets   bits which are 1 ignores bits which are 0
+// #define GPIO_CLR(g) *(gpio+10) = 1<<(g) // clears bits which are 1 ignores bits which are 0
+// #define GPIO_READ(g) (*(gpio+13) >> (g)) & 0x00000001
+// #define GPIO_READRAW *(gpio+13)
+
+// for ls2k0300
+#define GPIO_BASE 0x16104000
+
+// from kernel source, drivers/gpio/gpio-loongson-irq.c
+typedef struct lsirq_gpio_regtable_s {
+       uint32_t dir;
+       uint32_t out;
+       uint32_t in;
+       uint32_t irq;
+       uint32_t irqpol;
+       uint32_t irqedg;
+       uint32_t irqclr;
+       uint32_t irqsta;
+       uint32_t irqdul;
+} lsirq_gpio_regtable_t;
+
+static lsirq_gpio_regtable_t generic_reg_table = {
+       .dir    = 0x800,
+       .out    = 0x900,
+       .in     = 0xa00,
+       .irq    = 0xb00,
+       .irqpol = 0xc00,
+       .irqedg = 0xd00,
+       .irqclr = 0xe00,
+       .irqsta = 0xf00,
+       .irqdul = 0xf80,
+};
+
+#define REG_GPIO_DIR (generic_reg_table.dir)
+#define REG_GPIO_OUT (generic_reg_table.out)
+#define REG_GPIO_IN  (generic_reg_table.in)
+
+extern volatile unsigned char *gpio;
+
+static void lsirq_gpio_set_reg_dir(int p, unsigned char val) {
+    __sync_synchronize();
+    *(gpio + REG_GPIO_DIR + p) = val;
+}
+
+static void lsirq_gpio_set_reg_out(int p, unsigned char val) {
+    __sync_synchronize();
+    *(gpio + REG_GPIO_OUT + p) = val;
+}
+
+static unsigned char lsirq_gpio_get_reg_in(int p) {
+    unsigned char ret;
+    __sync_synchronize();
+    ret = *(gpio + REG_GPIO_IN + p);
+    __sync_synchronize();
+    return ret;
+}
 
-// GPIO setup macros. Always use INP_GPIO(x) before using OUT_GPIO(x) or SET_GPIO_ALT(x,y)
-#define INP_GPIO(g) *(gpio+((g)/10)) &= ~(7<<(((g)%10)*3))
-#define OUT_GPIO(g) *(gpio+((g)/10)) |=  (1<<(((g)%10)*3))
-#define SET_GPIO_ALT(g,a) *(gpio+(((g)/10))) |= (((a)<=3?(a)+4:(a)==4?3:2)<<(((g)%10)*3))
+#define INP_GPIO(g) lsirq_gpio_set_reg_dir(g, 1)
+#define OUT_GPIO(g) lsirq_gpio_set_reg_out(g, 0); lsirq_gpio_set_reg_dir(g, 0)
 
-#define GPIO_SET(g) *(gpio+7) = 1<<(g) // sets   bits which are 1 ignores bits which are 0
-#define GPIO_CLR(g) *(gpio+10) = 1<<(g) // clears bits which are 1 ignores bits which are 0
-#define GPIO_READ(g) (*(gpio+13) >> (g)) & 0x00000001
-#define GPIO_READRAW *(gpio+13)
+#define GPIO_SET(g) lsirq_gpio_set_reg_out(g, 1)
+#define GPIO_CLR(g) lsirq_gpio_set_reg_out(g, 0)
+#define GPIO_READ(g) lsirq_gpio_get_reg_in(g) & 1
 
 //Perspective is from Device connected, so TDO is output from device to input into rpi
-#define JTAG_TMS 27 //PI ---> JTAG
-#define JTAG_TDI 22 //PI ---> JTAG
-#define JTAG_TDO 23 //PI <--- JTAG
-#define JTAG_TCK 24 //PI ---> JTAG
+// #define JTAG_TMS 27 //PI ---> JTAG
+// #define JTAG_TDI 22 //PI ---> JTAG
+// #define JTAG_TDO 23 //PI <--- JTAG
+// #define JTAG_TCK 24 //PI ---> JTAG
+#define JTAG_TMS 69 //PI ---> JTAG
+#define JTAG_TDI 81 //PI ---> JTAG
+#define JTAG_TDO 70 //PI <--- JTAG
+#define JTAG_TCK 71 //PI ---> JTAG
 
 //-D DEBUG when compiling, will make all sleeps last 0.5 second, this can be used to test with LED on ports, or pushbuttons
 //Else sleeps can be reduced to increase speed
diff --git a/tcpborphserver3/spifpga_user.h b/tcpborphserver3/spifpga_user.h
index dc20023..b2c4aa3 100644
--- a/tcpborphserver3/spifpga_user.h
+++ b/tcpborphserver3/spifpga_user.h
@@ -1,11 +1,12 @@
 #ifndef SPIFPGA_USER_H_
 #define SPIFPGA_USER_H_
 
-#define DEVICE "/dev/spidev0.0"
+// #define DEVICE "/dev/spidev0.0"
+#define DEVICE "/dev/spidev2.0"
 #define MAX_SPEED 4000000
 #define DELAY 1
 #define BITS 8
-#define MAX_BURST_SIZE 256
+#define MAX_BURST_SIZE 1
 #define BYTES_PER_WORD 4
 
 struct fpga_spi_cmd {
diff --git a/tcpborphserver3/tcpborphserver3.h b/tcpborphserver3/tcpborphserver3.h
index 8ec8253..4ca8722 100644
--- a/tcpborphserver3/tcpborphserver3.h
+++ b/tcpborphserver3/tcpborphserver3.h
@@ -18,7 +18,8 @@
 #endif
 
 #define TBS_FPGA_CONFIG    "/dev/shm/fpga-config"
-#define TBS_FPGA_MEM       "/dev/spidev0.0"
+// #define TBS_FPGA_MEM       "/dev/spidev0.0"
+#define TBS_FPGA_MEM       "/dev/spidev2.0"
 
 #define TBS_KCPFPG_PATH    "/bin/kcpfpg"
 #define TBS_RAMFILE_PATH   "/dev/shm/gateware"
```

另外, 由于 Alpine 基于 musl 而不是 glibc, 并且 gcc 版本较高 (检查更严格), 需要进行额外的适配.

```diff
diff --git a/katcp/dispatch.c b/katcp/dispatch.c
index 87be231..42bda81 100644
--- a/katcp/dispatch.c
+++ b/katcp/dispatch.c
@@ -2251,6 +2251,10 @@ int append_args_katcp(struct katcp_dispatch *d, int flags, char *fmt, ...)
   return result;
 }
 
+#ifdef KATCP_EXPERIMENTAL
+int append_end_flat_katcp(struct katcp_dispatch *d);
+#endif
+
 int append_end_katcp(struct katcp_dispatch *d)
 {
   sane_katcp(d);
diff --git a/katcp/job.c b/katcp/job.c
index 4ddf491..0b646e2 100644
--- a/katcp/job.c
+++ b/katcp/job.c
@@ -13,6 +13,9 @@
 #include <sysexits.h>
 
 #include <sys/wait.h>
+#ifndef WAIT_ANY
+# define WAIT_ANY      (-1)
+#endif
 #include <sys/types.h>
 #include <sys/socket.h>

```

随后进行编译安装,

```bash
make -j`nproc`
sudo make install
cd tcpborphserver3
make -j`nproc`
sudo make install
```

这将把程序安装到 `/usr/local/bin` 和 `/usr/local/lib`.

## 3. 设置 katcpd

其实只是让 tcpborphserver3 开机自动启动.

由于 Alpine 似乎使用 init.d, 使用以下配置文件作为 `/etc/init.d/katcpd`;
注意由于未知原因, tcpborphserver3 在后台模式下无法正常工作, 因此使用 `-f` 指定前台模式并手动置于后台.

```bash
#!/usr/bin/env bash


start() {
  echo "STARTING"
  #tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log
  sh -c "tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log -f" &
}

stop() {
  echo "STOPPING"
  pkill -9 tcpborphserver3
}

case "$1" in
        start)
                start
                ;;
        stop)
                stop
                ;;
        restart)
                stop
                start
                ;;
        status)
                echo "HI"
                ;;
        *)
                echo "USAGE: $0 (start|stop|status|restart)"
esac

exit 0
```

<details>
<summary>备份: systemd 服务, 以及原版 katcpd</summary>

systemd service

创建 `/etc/systemd/system/katcpd.service`
```
[Unit]
Description=Start katcpd
After=network.target

[Service]
ExecStart=tcpborphserver3 -b /bofs
Restart=always
RestartSec=60
StartLimitInterval=0

[Install]
WantedBy=multi-user.target
```

注: 此 systemd 配置由以下位于 `/etc/init.d/katcpd` 的脚本转换而来

```bash
#!/usr/bin/env bash


start() {
  echo "STARTING"
  #tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log
  sh -c "tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log -f" &
}

stop() {
  echo "STOPPING"
  pkill -9 tcpborphserver3
}

case "$1" in
        start)
                start
                ;;
        stop)
                stop
                ;;
        restart)
                stop
                start
                ;;
        status)
                echo "HI"
                ;;
        *)
                echo "USAGE: $0 (start|stop|status|restart)"
esac

exit 0
```

然后
```bash
sudo systemctl reload-daemon
sudo systemctl enable --now katcpd
```

</details>

设定开机自启动: `/etc/local.d/katcp.start`:

```bash
#!/bin/sh

sh -c "sleep 10; /etc/init.d/katcpd start" &
```

随后将 local 加入开机自启动: `sudo rc-update add local default`


接着重启试验一下启动脚本能否正常上传 .fpg 文件即可.

## 4. 进一步定制

这里应该做你想要的进一步定制, 比如上传 `.fpg` 文件和启动脚本, 安装网络时间同步软件 (比如 chrony), 删掉不需要的软件之类的.


## 5. 保存此镜像

希望上面没遗漏什么重要的步骤.

将树莓派关机, 拔出 SD 卡并插回电脑, 可打开写保护开关以防止误操作.
将分区内容打包并用 7z 之类的压缩工具压缩镜像即可, 不需要用 dd 之类的维持分区表.

或许这就是全部了. 祝开发顺利!
