---
layout: post
title: 为 SNAP 构建树莓派镜像 | Build a Raspberry Pi image for SNAP
date: 2024-05-14
author: Fxzjshm
category: Radio Astronomy
tags: [Linux, CASPER, SNAP]
---

**Large languange model translated English version available below.**

## 0. 引言
SNAP (Smart Network ADC Processor) 是 CASPER 开发的低成本采样装置, 更多信息可见 <https://github.com/casper-astro/casper-hardware/blob/master/FPGA_Hosts/SNAP/README.md>. 通常情况下需要使用树莓派来驱动 SNAP, 但一些已有的镜像可能无法在新版树莓派上使用, 或可考虑其他单板计算机来控制 SNAP, 本文可为构建过程提供参考.

## 1. 下载系统
对于树莓派, 可从 <https://www.raspberrypi.com/software/> 下载其系统; 类似地其他单板计算机可从制造商处获取.

特别地, 如果已有可在旧版本树莓派中可启动的镜像但该镜像无法在新版树莓派上启动, 可以考虑仅替换引导/固件与内核, 此时请注意原先镜像中的系统是 32 位还是 64 位, 否则可能无法启动.

下载系统后, 按照其教程烧录系统到 SD 卡上.

新版内核可能缩小了 SPI 缓冲区的默认大小, 需要手动扩大.
在 bootfs 分区的 cmdline.txt (即 Linux 内核参数) 中添加
```
spidev.bufsiz=65536
```
(参考 <https://raspberrypi.stackexchange.com/questions/65595>)

然后卸载 SD 卡, 装入树莓派并启动.

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

根据习惯和需求, 进入 Python 虚拟环境用 pip 安装, 或用 pip 全局安装;
若可能对源码进行修改, 推荐使用
```bash
pip install -e .
```

### 2.2 katcp
注意, 目前已知仅有 <https://github.com/jack-h/katcp_devel/tree/rpi-devel-casperfpga> 处的 katcp 可支持树莓派,
```bash
git clone https://github.com/jack-h/katcp_devel.git
cd katcp_devel
git checkout rpi-devel-casperfpga
```

GPIO 相关的参数可能需要调整, 比如外设的内存地址,
见 [`e95899f`](https://github.com/jack-h/katcp_devel/commit/e95899ff5dee5f855b6d795d2b6543c708c6e508);
若需要适配其他单板计算机, 可能需要修改 
tcpborphserver3/rpjtag_io.h


随后进行编译安装,
```bash
make -j`nproc`
sudo make install
cd tcpborphserver3
make -j`nproc`
sudo make install
```
这将把程序安装到 `/usr/local/bin` 和 `/usr/local/lib`;
也可以用 `PREFIX` 自行指定安装位置, 但其他程序寻找 katcp 相关程序时需要相应适配.

## 3. 设置 katcpd
其实只是让 tcpborphserver3 开机自动启动. 这里以 systemd 为例, init.d 等请参考对应文档.

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
  tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log
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

接着试验一下启动脚本能否正常上传 .fpg 文件即可.

## 4. 进一步定制
这里应该做你想要的进一步定制, 比如上传 `.fpg` 文件和启动脚本, 删掉不需要的软件之类的.


## 5. 保存此镜像
希望上面没遗漏什么重要的步骤.

将树莓派关机, 拔出 SD 卡并插回电脑, 可打开写保护开关以防止误操作.
可先用 GParted, KDE Partition Manager 之类的工具将 rootfs 分区缩小来节约空间,
然后用 `dd` 将 SD 卡内容导出为镜像, 最后用 7z 之类的压缩工具压缩镜像即可.

或许这就是全部了. 祝开发顺利!

<br/>
<br/>

---

<br/>
<br/>

(Translated by Yi-1.5-34B, manual edited)

## 0. Introduction

The SNAP (Smart Network ADC Processor) is a low-cost sampling device developed by CASPER, for more information see <https://github.com/casper-astro/casper-hardware/blob/master/FPGA_Hosts/SNAP/README.md>. Typically, it requires being driven with a Raspberry Pi, but some existing images may not work on newer models of the Raspberry Pi, or one may consider using other single board computers to control the SNAP as well, then this text can serve as guidance in building one such setup.

## 1. Downloading The System

For Raspberry Pi, you can download its OS from <https://www.raspberrypi.org/software/>; similarly, get your operating system from the manufacturer if you're using other single-board computer.

Specifically, if you have an image that boots up fine on older versions of the Raspberry Pi but does not start on new ones, try replacing only the bootloader/firmware and the kernel. Note check whether the original image was running a 32-bit or 64-bit version of the OS, otherwise booting might fail.

After downloading the OS, follow their instructions to burn it onto the SD card.

Newer kernels might shrink down the default size of the SPI buffer, which needs manual expansion. Add this line into the cmdline.txt file within the boot partition:
```
spidev.bufsiz=65536
```
Reference: <https://raspberrypi.stackexchange.com/questions/65595>

Then eject the SD card, put it back into the Raspberry Pi, and power it up.
## 2. Installing CASPER Software
### 2.1 casperfpga
```bash
git clone https://github.com/casper-astro/casperfpga
cd casperfpga
git checkout py310-dev # Or use py27, py38-dev, etc.
```

Experience has shown the `py27` branch to be stable, though newer systems may no longer support Python 2.7;
the branches like `py38-dev` and `py310-dev` are recent but come with issues like inconsistent variable naming, interface changes, etc., which require manually fixing bugs and potentially porting code; based on feedback, there were also problems reported regarding 'py38'.

According to personal preference and requirements, enter a python virtual environment to install via pip or do global installation via pip; if source modifications are anticipated, recommend using
```bash
pip install -e .
```

### 2.2 katcp

Note that currently only <https://github.com/jack-h/katcp_devel/tree/rpi-devel-casperfpga> supports the Raspberry Pi,
```bash
git clone https://github.com/jack-h/katcp_devel.git
cd katcp_devel
git checkout rpi-devel-casperfpga
```
Parameters related to GPIO may need adjustment, such as peripheral memory addresses,
see [`e95899f`](https://github.com/jack-h/katcp_devel/commit/e95899ff5dee5f855b6d795d2b6543c708c6e508). If adapting to other single-board computers, adjustments to
tcpborphserver3/rpjtag_io.h may be necessary.

Proceed to compile and install:
```bash
make -j`nproc`
sudo make install
cd tcpborphserver3
make -j`nproc`
sudo make install
```
This installs programs to `/usr/local/bin` and `/usr/local/lib`; alternatively, specify custom installation locations via `PREFIX`, keeping in mind this may require adaptations when other programs search for katcp components.

## 3. Setting Up katcpd
Essentially just ensuring tcpborphserver3 starts automatically upon reboot. Here we take systemd as example, please refer to specific documentation for alternatives like init.d.

Create /etc/systemd/system/katcpd.service
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

Note: this systemd config is converted from a script in `/etc/init.d/katcpd`

```bash
#!/usr/bin/env bash


start() {
  echo "STARTING"
  tcpborphserver3 -b /bofs -l /logs/tcpborphserver3.log
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

Then
```bash
sudo systemctl reload-daemon
sudo systemctl enable --now katcpd
```
Next, test that startup scripts successfully upload FPG files.

## 4. Further Customization
Here should perform further customization according to what you wish, including uploading .fpg files and writing startup scripts, removing unnecessary software, etc.

## 5. Preserving This Image
Hopefully haven’t missed anything important above. 

Shut off the RPi, remove the SD card then reinsert into PC, you may flip write protection switch before accidentally making unwanted changes. 

You could first reduce space usage through tools like GParted, KDE Partition Manager, reducing the size of the rootfs partition. Then, export the contents of the SD card to an image using dd. Finally, compress the resulting image using something like 7z.

Perhaps that sums everything up. Wishing you smooth sailing ahead! Good luck with your development!
