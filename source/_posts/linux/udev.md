---
title: udev 使用总结
date: 2023-12-10 17:47:11
comments: true
tags:
  - linux
---
### 简介
udev是linux系统中提供设备事件的子系统，如：当一个U盘插入电脑时触发脚本，来实现自动化操作。
### 配置
udev的配置文件一般位于`/etc/udev/rules.d/`目录下，rule文件名最前面的数字字段代表这条rule的优先级，数字越大优先级越低。例如：
``` bash
root@debian:~# ls -l /etc/udev/rules.d/
total 12
-rw-r--r-- 1 root root  96 Jun 26  2022 70-persistent-net.rules
-rw-r--r-- 1 root root 127 Jun 26  2022 80-max-sectors-blk.rules
-rw-r--r-- 1 root root 193 Jun 26  2022 80-qcloud-nic.rules
root@debian:~# cat /etc/udev/rules.d/80-qcloud-nic.rules
SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", RUN+="/usr/local/qcloud/udev_run/udev_run.sh"
SUBSYSTEM=="net", ACTION=="remove", KERNEL=="eth*", RUN+="/usr/local/qcloud/udev_run/udev_run.sh"
```
