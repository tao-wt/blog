---
title: Linux系统traffic control流量控制
date: 2024-2-2 16:00:49
index_img: /img/index-10.jpg
tags:
  - linux
  - shell
  - bash
  - tc
categories:
  - [linux, shell, bash]
  - [linux, tc]
author: tao-wt@qq.com
excerpt: 最近测试团队提了一个弱网环境的测试需求，本文描述了使用tc命令实现的流量控制脚本，用来模拟弱网环境
---
> `tc`命令的使用细节，可以参考脚本后面的内容，这些内容提取自`tc`命令的说明手册，侧重于从整体上说明`tc`命令的用法和脚本中所涉及的`qdisc`, `class`和`filter`的概念/描述；若要深入理解，可参考[tc-iproute2帮助手册](https://manpages.debian.org/unstable/iproute2/tc.8.en.html "帮助手册地址")。

## 脚本使用说明
可以通过`--help`, `-h`参数来获取脚本帮助:
```
tao@S20:~/tao$ sudo sh traffic_control.sh -h
---------traffic_control.sh usage:-------------
this script support follow paramters:
  --addr/-a: the ip address, separate by ','
  --clean/-c: boolean, remove the traffic control setting
  --devices/-d: the network interfaces, separate by ','
  --delay/-e: int, the packet delay, default 900, (900ms)
  --loss/-l: int, the packet loss, default 50, (50%)
  --rate/-r: int, the traffic rate, default 100, (100kbps)
-------------------------------------------
tao@S20:~/tao$ 
```
此脚本默认是在接口(`--devices`参数，支持指定多个接口)层面对报文的速率、延迟和丢包率进行限制；当指定`ip`(`--addr`参数)时，只会对具体`ip`的流量进行限制，即不会在接口层面对整个流量进行限制。
`--clean`或`-c`参数用来清除指定接口上的限制。

### 示例
在进行流量控制之前，`ping`和`curl`命令的输出如下：
```
tao@S20:~/tao$ ping baidu.com
PING baidu.com (39.156.66.10) 56(84) bytes of data.
64 bytes from 39.156.66.10: icmp_seq=1 ttl=52 time=26.3 ms
64 bytes from 39.156.66.10: icmp_seq=2 ttl=52 time=26.1 ms
64 bytes from 39.156.66.10: icmp_seq=3 ttl=52 time=26.0 ms
^C64 bytes from 39.156.66.10: icmp_seq=4 ttl=52 time=26.3 ms

--- baidu.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 10665ms
rtt min/avg/max/mdev = 26.045/26.178/26.347/0.125 ms
tao@S20:~/tao$ 
tao@S20:~/tao$ curl -u 'tao_test:******' -X GET http://1.*.*.*/TAO_other/temp/29105342_b.zip -o 29105342.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 2803M  100 2803M    0     0   109M      0  0:00:25  0:00:25 --:--:--  111M
tao@S20:~/tao$ 
```
可以看到上面的`ping`的延迟为26ms，没有丢包；`curl`命令的下载速度为：111MB/s。
现在运行脚本来对`enp99s0`接口的流量进行限制，设置速率为300KB/s、延迟为：200ms、丢包率为：20%
```
tao@S20:~/tao$ sudo sh traffic_control.sh --devices enp99s0 --delay 200 --rate 300 --loss 20
[INFO] >>> devices: 'enp99s0'
[INFO] >>> delay: '200ms'
[INFO] >>> loss: '20%'
[INFO] >>> rate: '300kbps'
[INFO] >>> addr: 'all'
[INFO] >>> Executing: traffic_control
[INFO] >>> start to configure device 'enp99s0'
[INFO] >>> device 'enp99s0' configure complete
[INFO] >>> all devices are configured and ready for testing
tao@S20:~/tao$ 
```
再次用`ping`和`curl`进行测试，可以看到速率、丢包率和延迟都达到了预期效果:
```
tao@S20:~/tao$ ping baidu.com
PING baidu.com (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=1 ttl=49 time=230 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=3 ttl=49 time=230 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=5 ttl=49 time=230 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=6 ttl=49 time=230 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=8 ttl=49 time=230 ms
64 bytes from 110.242.68.66 (110.242.68.66): icmp_seq=9 ttl=49 time=230 ms
64 bytes from 110.242.68.66: icmp_seq=10 ttl=49 time=230 ms
^C64 bytes from 110.242.68.66: icmp_seq=11 ttl=49 time=230 ms

--- baidu.com ping statistics ---
11 packets transmitted, 8 received, 27.2727% packet loss, time 23961ms
rtt min/avg/max/mdev = 229.778/229.910/230.175/0.138 ms
tao@S20:~/tao$ 
tao@S20:~/tao$ curl -u 'tao_test:******' -X GET http://1.*.*.*/TAO_other/temp/29105342_b.zip -o 29105342.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0 2803M    0 7887k    0     0   285k      0  2:47:29  0:00:27  2:47:02  303k^C
tao@S20:~/tao$ 
```
最后，清除上面所作的限制：
```
tao@S20:~/tao$ sudo sh traffic_control.sh --devices enp99s0 --clean
[INFO] >>> devices: 'enp99s0'
[INFO] >>> cleanup traffic control setting...
[INFO] >>> device 'enp99s0' traffic control config removed
[INFO] >>> cleanup done
tao@S20:~/tao$ 
```

## 脚本
traffic_control.sh脚本如下：
```sh
#!/usr/bin/env sh
set -e


die() {
    echo "[ERROR] >>> $*" >&2
    exit 1
}

einfo() {
    echo "[INFO] >>> $*" >&2
}

ewarn() {
    echo "[WARNING] >>> $*" >&2
}

run() {
    # It's nice to be able to print some commands without
    # enabling XTRACE for all the things.
    einfo "Executing: $*"
    "$@"
}

traffic_control() {
    for eth in $(echo "${devices}" | tr ',' ' '); do
        if ! ip a s | grep -q "${eth}"; then
            ewarn "device '${eth}' not found, skip!"
            continue
        fi
        einfo "start to configure device '${eth}'"
        configure_device "${eth}"
        einfo "device '${eth}' configure complete"
    done
    einfo "all devices are configured and ready for testing"
    # while true; do
    #     sleep 13s
    # done
}

edebug() {
    if [[ -n "${DEBUG}" ]]; then
        echo "[DEBUG] >>> $*" >&2
    fi
}

help() {
    echo "---------$(basename $0) usage:-------------"
    echo "this script support follow paramters:"
    echo "  --addr/-a: the ip address, separate by ','"
    echo "  --clean/-c: boolean, remove the traffic control setting"
    echo "  --devices/-d: the network interfaces, separate by ','"
    echo "  --delay/-e: int, the packet delay, default 900, (900ms)"
    echo "  --loss/-l: int, the packet loss, default 50, (50%)"
    echo "  --rate/-r: int, the traffic rate, default 100, (100kbps)"
    echo "-------------------------------------------"
}

verify_arguments() {
    if [ -z "${devices}" ]; then
        help
        die "the 'devices' is required!"
    fi
    einfo "devices: '${devices}'"
    if "${clean}"; then
        cleanup
        exit 0
    fi
    if [ -z "${delay}" ]; then
        delay=900
    fi
    einfo "delay: '${delay}ms'"
    if [ -z "${loss}" ]; then
        loss=50
    fi
    einfo "loss: '${loss}%'"
    if [ -z "${rate}" ]; then
        rate=100
    fi
    einfo "rate: '${rate}kbps'"
    if [ -z "${addr}" ]; then
        addr="all"
    fi
    einfo "addr: '${addr}'"
}

configure_device() {
    local device="${1}"

    tc qdisc add dev "${device}" root handle ab: htb default 10
    tc class add dev "${device}" parent ab: \
        classid ab:10 htb rate "${rate}kbps" \
        burst "${rate}k"
    tc qdisc add dev "${device}" parent ab:10 \
        netem loss "${loss}%" delay "${delay}ms"
    if [ -n "${addr}" ]; then
        configure_ip "${device}"
    fi
    config_ingress_traffic "${device}"
}

config_ingress_traffic() {
    local device="${1}"

    tc qdisc add dev "${device}" handle ffff: ingress

    for ip_addr in $(echo "${addr}" | tr ',' ' '); do
        if ! check_ip_addr "${ip_addr}"; then
            continue
        fi
        tc filter add dev "${device}" parent ffff: \
            protocol ip prio 3 \
            u32 match ip src "${ip_addr}" \
            police rate "${rate}kbps" burst "${rate}k" drop flowid :10
    done
}

get_arguments() {
    clean="false"
    while [ $# -ge 1 ]; do
        case "${1}" in
            --devices|-d)
                devices="${2}"
                shift 2
                ;;
            --delay|-e)
                delay="${2}"
                shift 2
                ;;
            --rate|-r)
                rate="${2}"
                shift 2
                ;;
            --addr|-a)
                addr="${2}"
                shift 2
                ;;
            --loss|-l)
                loss="${2}"
                shift 2
                ;;
            --help|-h)
                help
                exit 0
                ;;
            --clean|-c)
                clean="true"
                shift
                ;;
            *)
                ewarn "unknown option: ${1}! use '--help' for help info"
                shift 1
                ;;
        esac
    done
    verify_arguments
}

configure_ip() {
    local device="${1}"

    for ip_addr in $(echo "${addr}" | tr ',' ' '); do
        if ! check_ip_addr "${ip_addr}"; then
            continue
        fi
        tc filter add dev "${device}" \
            protocol ip \
            parent ab:0 prio 1 \
            u32 match ip dst "${ip_addr}" flowid ab:10

        tc filter add dev "${device}" \
            protocol ip \
            parent ab:0 prio 1 \
            u32 match ip src "${ip_addr}" flowid ab:10
    done
}

check_permission() {
    local user_id
    user_id="$(id -u)"
    if [ "${user_id}" -ne "0" ]; then
        die "please use root permission to run this script!"
    fi
}

cleanup() {
    einfo "cleanup traffic control setting..."
    for eth in $(echo "${devices}" | tr ',' ' '); do
        if ! ip a s | grep -q "${eth}"; then
            continue
        fi
        if ! tc qdisc show dev "${eth}" | grep -q "qdisc htb ab:"; then
            continue
        fi
        tc qdisc del dev "${eth}" root handle ab:
        tc qdisc del dev "${eth}" handle ffff: ingress
        einfo "device '${eth}' traffic control config removed"
    done
    einfo "cleanup done"
}

check_ip_addr() {
    local ip_addr="${1}"

    if [ "${ip_addr}" != "all" ]; then
        if ! echo "${ip_addr}" | grep -Eq \
                "^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$"; then
            ewarn "addr '${ip_addr}' format error, skip!"
            return 1
        fi
    fi
    return 0
}


if [ "${1}" != "--source-only" ]; then
    check_permission
    get_arguments "${@}"
    # trap cleanup EXIT
    run traffic_control
fi

```

## 命令简介
`tc`命令主要是用来查看/操作内核中的traffic control的设置。

### TC DESCRIPTION
Traffic Control由以下概念组成：
- **SHAPING**
    When traffic is shaped, its rate of transmission is under control. Shaping may be more than lowering the available bandwidth - it is also used to smooth out bursts in traffic for better network behaviour. Shaping occurs on egress.
- **SCHEDULING**
    By scheduling the transmission of packets it is possible to improve interactivity for traffic that needs it while still guaranteeing bandwidth to bulk大部分 transfers. Reordering is also called prioritizing, and happens only on egress.
- **POLICING**
    Whereas shaping deals with transmission of traffic, policing pertains to traffic arriving. Policing thus occurs on ingress.
- **DROPPING**
    Traffic exceeding超过 a set bandwidth may also be dropped forthwith, both on ingress and on egress.

Processing of traffic is controlled by three kinds of objects: `qdiscs`, `classes` and `filters`.
- **QDISCS**
    `qdisc` is short for 'queueing discipline' and it is elementary基础,初级 to understanding traffic control. Whenever the kernel needs to send a packet to an interface, it is enqueued to the qdisc configured for that interface. Immediately afterwards, the kernel tries to get as many packets as possible from the qdisc, for giving them to the network adaptor driver.
- **CLASSES**
    Some qdiscs can contain `classes`, which contain further qdiscs - traffic may then be enqueued in any of the inner qdiscs, which are within the classes. When the kernel tries to dequeue a packet from such a `classful qdisc` it can come from any of the classes.A qdisc may for example prioritize certain kinds of traffic by trying to dequeue from certain classes before others.
- **FILTERS**
    A `filter` is used by a classful qdisc to determine in which class a packet will be enqueued. Whenever traffic arrives at a class with subclasses, it needs to be classified. Various methods may be employed to do so, one of these are the filters. All filters attached to the class are called, until one of them returns with a verdict判决. It is important to notice that filters reside within qdiscs - they are not masters of what happens.
    > The available filters are: `basic`, `bpf`, `cgroup`, `route`, `u32`等等，本文用到的是**u32**:
    > `u32`: Generic filtering on arbitrary packet data. The Universal/Ugly 32bit filter allows one to match arbitrary bitfields in the packet. Due to breaking everything down to values, masks and offsets, It is equally powerful and hard to use. Luckily many abstracting directives are present which allow defining rules on a higher level and therefore free the user from having to fiddle不停摆弄 with bits and masks in many cases.

### QEVENTS
Qdiscs may invoke user-configured actions when certain interesting events take place in the qdisc. Each `qevent` can either be unused, or can have a block attached to it. To this block are then attached filters using the "tc block BLOCK_IDX" syntax. The block is executed when the qevent associated with the attachment point takes place.
```bash
tc qdisc add dev eth0 root handle 1: red limit 500K avpkt 1K \
qevent early_drop block 10
tc filter add block 10 matchall action mirred egress mirror dev eth1
```

### CLASSLESS QDISCS
In the absence of classful qdiscs, classless qdiscs can only be attached at the root of a device. Full syntax:
```
tc qdisc add dev DEV root QDISC_NAME QDISC-PARAMETERS
tc qdisc del dev DEV root
```
The `pfifo_fast` qdisc is the automatic default in the absence of a configured qdisc. The classless qdiscs are: `ingress`, `tbf`,`netem`,`[p|b]fifo`,`red`等等
- **[p|b]fifo**
    Simplest usable qdisc, pure First In, First Out behaviour. Limited in packets or in bytes.
- **ingress**
    This is a special qdisc as it applies to incoming traffic on an interface, allowing for it to be filtered and policed.
- **red**
    Random Early Detection simulates模拟 physical congestion拥塞 by randomly dropping packets when nearing configured bandwidth allocation分配. Well suited to very large bandwidth applications.主要用于出口网络

### CLASSFUL QDISCS
The classful qdiscs are: `HTB`,`PRIO`,`HFSC`等等
`HTB`: The Hierarchy Token Bucket implements a rich linksharing hierarchy of classes with an emphasis on conforming to existing practices.(层次结构令牌桶实现了丰富的类链接共享层次结构，重点是符合现有实践。) HTB facilitates guaranteeing bandwidth to classes, while also allowing specification of upper limits to inter-class sharing. It contains `shaping` elements, based on `TBF` and can prioritize classes.

### THEORY理论 OF OPERATION
Classes form a tree, where each class has a single parent. A class may have multiple children. Some qdiscs allow for runtime addition of classes (HTB) while others (PRIO) are created with a static number of children.
Qdiscs which allow dynamic addition of classes can have zero or more subclasses to which traffic may be enqueued.
Furthermore, each class contains a `leaf qdisc` which by default has `pfifo` behaviour, although another qdisc can be attached in place. This qdisc may again contain classes, but each class can have only one `leaf qdisc`.
When a packet enters a classful qdisc it can be classified to one of the classes within.
Each node within the tree can have its own filters but higher level filters may also point directly to lower classes.
If classification did not succeed, packets are enqueued to the leaf qdisc attached to that class. Check qdisc specific manpages for details, however.

### NAMING
All qdiscs, classes and filters have IDs, which can either be specified or be automatically assigned.
IDs consist of **major** number and a **minor** number, separated by a colon - **major:minor**. Both **major** and **minor** are hexadecimal numbers and are limited to 16 bits. There are two special values: root is signified by **major** and **minor** of all ones, and unspecified is all zeros.
- QDISCS
    A qdisc, which potentially can have children, gets assigned a **major** number, called a 'handle', leaving the **minor** number namespace available for classes. The handle is expressed as '10:'. It is customary to explicitly assign a handle to qdiscs expected to have children.
- CLASSES
    Classes residing under a qdisc share their qdisc **major** number, but each have a separate **minor** number called a 'classid' that has no relation to their parent classes, only to their parent qdisc. The same naming custom as for qdiscs applies.
- FILTERS
    Filters have a three part ID, which is only needed when using a hashed filter hierarchy.

### TC COMMANDS
The following commands are available for qdiscs, classes and filter:
- add
    Add a qdisc, class or filter to a node. For all entities, a **parent** must be passed, either by passing its ID or by attaching directly to the root of a device. When creating a qdisc or a filter, it can be named with the **handle** parameter. A class is named with the **classid** parameter.
- delete
    A qdisc can be deleted by specifying its handle, which may also be 'root'. All subclasses and their leaf qdiscs are automatically deleted, as well as any filters attached to them.
- show
    Displays all filters attached to the given interface. A valid parent ID must be passed.
