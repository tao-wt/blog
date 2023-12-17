---
title: udev 使用总结
date: 2023-12-10 17:47:11
index_img: /img/udev-index.jpg
tags:
  - linux
  - 系统管理
categories:
  - [linux, 系统管理]
author: Tao
excerpt: udev是linux系统中提供设备事件的子系统，如：当一个U盘插入电脑时触发脚本，来实现自动化操作。
---
### 简介
---
udev是linux系统中提供设备事件的子系统，如：当一个U盘插入电脑时触发脚本，来实现自动化操作。

### 配置
---
udev的配置文件一般位于`/etc/udev/rules.d/`目录下，rule文件名最前面的数字字段代表这条rule的优先级，数字越大优先级越低。例如：
``` bash
root@debian:~# ls -l /etc/udev/rules.d/
total 12
-rw-r--r-- 1 root root  96 Jun 26  2022 70-persistent-net.rules
-rw-r--r-- 1 root root 127 Jun 26  2022 80-max-sectors-blk.rules
-rw-r--r-- 1 root root 193 Jun 26  2022 80-qcloud-nic.rules
tao@debian:~# cat /etc/udev/rules.d/80-qcloud-nic.rules
SUBSYSTEM=="net", ACTION=="add", KERNEL=="eth*", RUN+="/usr/local/qcloud/udev_run/udev_run.sh"
SUBSYSTEM=="net", ACTION=="remove", KERNEL=="eth*", RUN+="/usr/local/qcloud/udev_run/udev_run.sh"
```

### 各字段说明及其信息的获取
---
udev的配置文件中各字段含义如下：
#### 各属性key的说明
---
以下是常用属性字段的简单说明，详细信息可查看`man udev`说明文档
- `SUBSYSTEM`: 匹配事件设备的系统设备类型，例如，若是磁盘则为`block`
- `ACTION`: 指对应设备(如U盘)的插入或移除
- `KERNEL`: 匹配事件设备的名字
- `ATTR{filename}`: 匹配事件设备的属性值，如可以指定设备的序列号
- `ENV{key}`: 匹配设备的特性值
- `RUN{type}`: 指定要运行的程序，针对匹配的事件，用空格分割程序参数，用单引号来指定带空格的参数
    `type`有以下两种类型
    `program`: 运行外部程序，建议使用绝对路径
    `builtin`: 使用built-in程序

#### 相关信息获取
---
以上属性值可用过下列命令获取
- `lsblk`命令用来列出块设备信息, 如uuid,label,type等等
    ```bash
    tao@z20:~$ lsblk -la -o NAME,LABEL,UUID,TYPE,MOUNTPOINTS /dev/sdb1 /dev/sda1
    NAME LABEL  UUID                                 TYPE MOUNTPOINTS
    sda1        d5cad8b3-b682-4204-a4d2-5bc9a1e5ebf4 part /project
    sdb1 RTC_LE 7478-FC48                            part
    ```
- `udevadm info`可以用来列出设备所有相关信息
    ```bash
    tao@z20:~$ udevadm info /dev/sdb1
    P: /devices/pci0000:00/0000:00:14.0/usb1/1-4/1-4:1.0/host12/target12:0:0/12:0:0:0/block/sdb/sdb1
    N: sdb1
    L: 0
    E: DEVNAME=/dev/sdb1
    E: DEVTYPE=partition
    E: DISKSEQ=68
    E: PARTN=1
    E: MAJOR=8
    E: MINOR=17
    E: SUBSYSTEM=block
    E: ID_SERIAL=Taorfvnj_dsacdarwgree5uy76-0:0
    E: ID_SERIAL_SHORT=davfvfsb4grwgt43
    E: ID_TYPE=disk
    E: ID_INSTANCE=0:0
    E: ID_BUS=usb
    E: ID_USB_DRIVER=usb-storage
    E: ID_PART_TABLE_UUID=23727e29
    E: ID_PART_TABLE_TYPE=dos
    E: ID_FS_LABEL=OTA_LE
    E: ID_FS_LABEL_ENC=OTA_LE
    E: ID_FS_UUID=7478-FC48
    E: ID_FS_UUID_ENC=7478-FC48
    E: ID_FS_VERSION=FAT32
    E: ID_FS_TYPE=vfat
    ...

    ```
- `udevadm monitor`常用来实时输出设备事件信息

#### 常用操作符说明
---
- `==`: 比较key是否等于给定的值或匹配模式
- `!=`: 判断不等于
- `+=`: 将一个值添加到一个key，即一个key的值可能是列表，如上面`run+=`
- `=`: 将值赋给key，列表会被覆盖

#### 常用匹配模式说明
---
- `*`: 匹配零个或多个字符
- `?`: 匹配任意单个字符
- `[]`: 匹配位于中括号中的单个的字符，如"tty[SR]"
- `|`: 可用来指定多个匹配模式，如，"abc|x*"将匹配"abc"或是"x*"

### 加载配置文件
---
经过以上信息查找，当指定的label的U盘插入电脑时触发自动化操作的rule如下；其中会将device的label当作参数传给脚本，在脚本运行前`%E{ID_FS_LABEL}`会进行变量替换，变量含义请参考上面属性字段说明
```bash
tao@z20:~$ cat /etc/udev/rules.d/80-local.rules
SUBSYSTEM=="block", ACTION=="add", ENV{ID_FS_LABEL}=="OUR_EP|OUR_CJ|OUR_SNM", RUN+="/home/tao/trigger.sh %E{ID_FS_LABEL}"
```
配置确认无误后通过`udevadm control --reload`命令重新加载配置文件
