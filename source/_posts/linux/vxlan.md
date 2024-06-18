---
title: 理解并实践vxlan
date: 2024-6-18 13:57:30
index_img: /img/index-18.jpg
tags:
  - linux
  - network
  - vxlan
  - overlay
categories:
  - [linux, network, overlay]
  - [linux, network, vxlan]
author: tao-wt@qq.com
excerpt: 之前在看docker和k8s时经常看到overlay和vxlan相关概念，利用这个机会深入了解下。
---
# Vxlan
The VXLAN protocol is a tunnelling protocol designed to solve the problem of limited VLAN IDs (4096) in `IEEE 802.1q`. With VXLAN the size of the identifier is expanded to 24 bits (16777216).

Unlike most tunnels, a VXLAN is a 1 to N network, not just point to point. A VXLAN device can learn the IP address of the other endpoint either dynamically in a manner similar to a learning bridge, or make use of statically-configured forwarding entries.

## 原理
`VXLAN`是在底层物理网络`underlay`之上使用隧道技术，借助`UDP`层构建的`Overlay`的逻辑网络，使逻辑网络与物理网络解耦，实现灵活的组网需求。它对原有的网络架构几乎没有影响，不需要对原网络做任何改动，即可架设一层新的网络。也正是因为这个特性，很多`CNI`插件才会选择`VXLAN`作为通信手段。

`VXLAN`报文的转发过程：原始报文经过`VTEP`，被Linux内核添加上`VXLAN`头部以及外层的`UDP`头部，再发送出去，对端`VTEP`接收到`VXLAN`报文后拆除外层`UDP`头部，并根据`VXLAN`头部的`VNI`把原始报文发送到目的服务器。

VTEP 转发表的学习可以通过以下两种方式：
- 多播
- 外部控制中心（如`Flannel`等`CNI`插件）

具体vxlan的介绍介绍请参考[这里](https://www.cnblogs.com/ryanyangcs/p/12696837.html)

## 本地测试
### 基于多播的vxlan
> This creates a new device named vxlan0. The device uses the multicast group 239.1.1.1 over eth1 to handle traffic for which there is no entry in the forwarding table. The destination port number is set to the IANA-assigned value of 4789.

36.66上配置如下：
```bash
tao@S20:~$ sudo ip link add name vxlan0 type vxlan id 6000 group 239.1.1.1 dstport 4789 dev eno1
tao@S20:~$ sudo ip addr add 192.168.5.10/24 dev vxlan0
tao@S20:~$ sudo ip link set vxlan0 up
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
169.254.0.0/16 dev eno1 scope link metric 1000
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.5.0/24 dev vxlan0 proto kernel scope link src 192.168.5.10
tao@S20:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 239.1.1.1 via eno1 self permanent
tao@S20:~$ ip a s vxlan0
2914: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 4a:bd:99:0c:09:b2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.10/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::48bd:99ff:fe0c:9b2/64 scope link
       valid_lft forever preferred_lft forever
tao@S20:~$ ip a s eno1
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether bb:6b:8c:86:6d:83 brd ff:ff:ff:ff:ff:ff
    altname enp4s0
    inet 10.138.36.66/23 brd 10.138.37.255 scope global dynamic noprefixroute eno1
       valid_lft 84560sec preferred_lft 84560sec
    inet6 fe80::61a2:dfa0:368c:60c2/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
tao@S20:~$
```

10.161配置如下：
```bash
tao@S8:~$ sudo ip link add name vxlan0 type vxlan id 6000 group 239.1.1.1 dstport 4789 dev eno2
tao@S8:~$ sudo ip addr add 192.168.5.11/24 dev vxlan0
tao@S8:~$ sudo ip link set vxlan0 up
tao@S8:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 239.1.1.1 via eno2 self permanent
tao@S8:~$
```

36.58配置如下：
```bash
tao@S3:~$ sudo ip link add name vxlan0 type vxlan id 6000 group 239.1.1.1 dstport 4789 dev eno2
tao@S3:~$ sudo ip addr add 192.168.5.12/24 dev vxlan0
tao@S3:~$ sudo ip link set vxlan0 up
tao@S3:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 239.1.1.1 via eno2 self permanent
tao@S3:~$
tao@S3:~$ ip a s vxlan0
24: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 62:7a:e6:8f:bc:5c brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.12/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::607a:e6ff:fe8f:bc5c/64 scope link
       valid_lft forever preferred_lft forever
tao@S3:~$ ip a s eno2
3: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bb:6b:8c:89:88:22 brd ff:ff:ff:ff:ff:ff
    altname enp0s31f6
    inet 10.138.36.58/23 brd 10.138.37.255 scope global dynamic noprefixroute eno2
       valid_lft 44839sec preferred_lft 44839sec
    inet6 fe80::3304:c53b:493e:2f97/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
tao@S3:~$
```

在36.66上执行`ping 192.168.5.12 -c1`：
```bash
tao@S20:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 239.1.1.1 via eno1 self permanent
tao@S20:~$ ping 192.168.5.12 -c1
PING 192.168.5.12 (192.168.5.12) 56(84) bytes of data.
64 bytes from 192.168.5.12: icmp_seq=1 ttl=64 time=0.447 ms

--- 192.168.5.12 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.447/0.447/0.447/0.000 ms
tao@S20:~$
tao@S20:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 239.1.1.1 via eno1 self permanent
62:7a:e6:8f:bc:5c dst 10.138.36.58 self
tao@S20:~$
```

在36.58的eno2接口上抓包，可以看到只有第一个报文是多播报文，后续的都是单播的`UDP`报文:
```bash
tao@S3:~$ sudo tcpdump -i eno2 'host 239.1.1.1 or host 10.138.36.66' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:18:00.613299 bb:6b:8c:86:6d:83 > 01:00:5e:01:01:01, ethertype IPv4 (0x0800), length 92: 10.138.36.66.40103 > 239.1.1.1.4789: VXLAN, flags [I] (0x08), vni 6000
4a:bd:99:0c:09:b2 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.12 tell 192.168.5.10, length 28
15:18:00.613401 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 92: 10.138.36.58.34917 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 6000
62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype ARP (0x0806), length 42: Reply 192.168.5.12 is-at 62:7a:e6:8f:bc:5c, length 28
15:18:00.613525 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 148: 10.138.36.66.57039 > 10.138.36.58.4789: VXLAN, flags [I] (0x08), vni 6000
4a:bd:99:0c:09:b2 > 62:7a:e6:8f:bc:5c, ethertype IPv4 (0x0800), length 98: 192.168.5.10 > 192.168.5.12: ICMP echo request, id 61, seq 1, length 64
15:18:00.613571 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 148: 10.138.36.58.56258 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 6000
62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype IPv4 (0x0800), length 98: 192.168.5.12 > 192.168.5.10: ICMP echo reply, id 61, seq 1, length 64
15:18:05.658993 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.58 tell 10.138.36.66, length 46
15:18:05.659005 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype ARP (0x0806), length 42: Reply 10.138.36.58 is-at bb:6b:8c:89:88:22, length 28
15:18:05.665788 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 92: 10.138.36.58.34917 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 6000
62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.10 tell 192.168.5.12, length 28
15:18:05.665814 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 10.138.36.58, length 28
15:18:05.665935 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 46
15:18:05.665943 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 92: 10.138.36.66.40103 > 10.138.36.58.4789: VXLAN, flags [I] (0x08), vni 6000
4a:bd:99:0c:09:b2 > 62:7a:e6:8f:bc:5c, ethertype ARP (0x0806), length 42: Reply 192.168.5.10 is-at 4a:bd:99:0c:09:b2, length 28
^C
10 packets captured
10 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
> 多播的以太网`MAC`地址高位为：`01:00:5e`，低23bit位是通过将`多播`组号`239.1.1.1`的低23bit映射过来实现的，所以`239.1.1.1`对应的以太网地址为`01:00:5e:01:01:01`

在36.58的vxlan0接口上抓包，可以看到正常的真实的交互报文:
```bash
tao@S3:~$ sudo tcpdump -i vxlan0 'arp or icmp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on vxlan0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:18:00.613355 4a:bd:99:0c:09:b2 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.12 tell 192.168.5.10, length 28
15:18:00.613372 62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype ARP (0x0806), length 42: Reply 192.168.5.12 is-at 62:7a:e6:8f:bc:5c, length 28
15:18:00.613539 4a:bd:99:0c:09:b2 > 62:7a:e6:8f:bc:5c, ethertype IPv4 (0x0800), length 98: 192.168.5.10 > 192.168.5.12: ICMP echo request, id 61, seq 1, length 64
15:18:00.613561 62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype IPv4 (0x0800), length 98: 192.168.5.12 > 192.168.5.10: ICMP echo reply, id 61, seq 1, length 64
15:18:05.665758 62:7a:e6:8f:bc:5c > 4a:bd:99:0c:09:b2, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.10 tell 192.168.5.12, length 28
15:18:05.665981 4a:bd:99:0c:09:b2 > 62:7a:e6:8f:bc:5c, ethertype ARP (0x0806), length 42: Reply 192.168.5.10 is-at 4a:bd:99:0c:09:b2, length 28
^C
6 packets captured
6 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

在36.66上`ping 192.168.5.11`就不测试了，网关对多播报文的转发需要额外`IGMP`配置，暂时跳过。

### 点对点的vxlan
36.66上配置如下：
```bash
tao@S20:~$ sudo ip link del vxlan0
tao@S20:~$ sudo ip link add name vxlan0 type vxlan id 80000 remote 10.138.10.161 dstport 4789 dev eno1
tao@S20:~$ sudo ip addr add 192.168.5.121/24 dev vxlan0
tao@S20:~$ sudo ip link set vxlan0 up
tao@S20:~$ ip a s vxlan0
2923: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 4a:13:be:ec:2f:59 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.121/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::4813:beff:feec:2f59/64 scope link
       valid_lft forever preferred_lft forever
tao@S20:~$ ip a s eno1
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether bb:6b:8c:86:6d:83 brd ff:ff:ff:ff:ff:ff
    altname enp4s0
    inet 10.138.36.66/23 brd 10.138.37.255 scope global dynamic noprefixroute eno1
       valid_lft 83191sec preferred_lft 83191sec
    inet6 fe80::61a2:dfa0:368c:60c2/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
tao@S20:~$ sudo bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 10.138.10.161 via eno1 self permanent
tao@S20:~$
```

10.161上配置如下：
```bash
tao@S8:~$
tao@S8:~$ sudo ip link add name vxlan0 type vxlan id 80000 remote 10.138.36.66 dstport 4789 dev eno1
tao@S8:~$ sudo ip addr add 192.168.5.123/24 dev vxlan0
tao@S8:~$ sudo ip link set vxlan0 up
tao@S8:~$ ip a s vxlan0
11: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether 16:51:8c:32:2f:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.123/24 scope global vxlan0
       valid_lft forever preferred_lft forever
    inet6 fe80::1451:8cff:fe32:2fe4/64 scope link
       valid_lft forever preferred_lft forever
tao@S8:~$ ip a s eno2
3: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bb:6b:8c:86:70:de brd ff:ff:ff:ff:ff:ff
    altname enp0s31f6
    inet 10.138.10.161/23 brd 10.138.11.255 scope global dynamic noprefixroute eno2
       valid_lft 66462sec preferred_lft 66462sec
    inet6 fe80::490:635c:bd87:17b2/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
tao@S8:~$ bridge fdb show dev vxlan0
00:00:00:00:00:00 dst 10.138.36.66 via eno1 self permanent
4a:13:be:ec:2f:59 dst 10.138.36.66 self
tao@S8:~$
```

**注意**：当配置完10.161的vxlan0接口时，转发表同样已经自动添加了一条36.66的`vxlan0`口的转发条目。这是由于`MDNS`和`NETBIOS`协议引起的：
```bash
tao@S8:~$ sudo tcpdump -i eno2 host 10.138.36.66 -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:15:12.615694 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 142: 10.138.36.66.47688 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 92: 192.168.5.121.137 > 192.168.5.255.137: UDP, length 50
16:15:13.372972 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 142: 10.138.36.66.47688 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 92: 192.168.5.121.137 > 192.168.5.255.137: UDP, length 50
16:15:14.126408 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 142: 10.138.36.66.47688 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 92: 192.168.5.121.137 > 192.168.5.255.137: UDP, length 50
16:15:16.128505 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 142: 10.138.36.66.47688 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 92: 192.168.5.121.137 > 192.168.5.255.137: UDP, length 50
16:15:19.132755 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 297: 10.138.36.66.33419 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 247: 192.168.5.121.138 > 192.168.5.255.138: UDP, length 205
16:15:21.135685 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 297: 10.138.36.66.33419 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 247: 192.168.5.121.138 > 192.168.5.255.138: UDP, length 205
16:15:23.138527 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 297: 10.138.36.66.33419 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 247: 192.168.5.121.138 > 192.168.5.255.138: UDP, length 205
16:15:25.140570 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 297: 10.138.36.66.33419 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 247: 192.168.5.121.138 > 192.168.5.255.138: UDP, length 205
16:15:27.143426 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 297: 10.138.36.66.33419 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 247: 192.168.5.121.138 > 192.168.5.255.138: UDP, length 205
16:16:08.539240 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 120: 10.138.36.66.39339 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > 33:33:00:00:00:02, ethertype IPv6 (0x86dd), length 70: fe80::a0c7:a1ff:fea5:e5ec > ff02::2: ICMP6, router solicitation, length 16
16:17:39.603861 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 137: 10.138.36.66.42587 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > 01:00:5e:00:00:fb, ethertype IPv4 (0x0800), length 87: 192.168.5.121.5353 > 224.0.0.251.5353: 0 [2q] PTR (QM)? _ipps._tcp.local. PTR (QM)? _ipp._tcp.local. (45)
16:17:41.221918 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 157: 10.138.36.66.58507 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
a2:c7:a1:a5:e5:ec > 33:33:00:00:00:fb, ethertype IPv6 (0x86dd), length 107: fe80::a0c7:a1ff:fea5:e5ec.5353 > ff02::fb.5353: 0 [2q] PTR (QM)? _ipps._tcp.local. PTR (QM)? _ipp._tcp.local. (45)
^C
12 packets captured
13 packets received by filter
0 packets dropped by kernel
tao@S8:~$
```
> `MDNS`协议: mDNS multicast DNS , 使用5353端口，组播地址 224.0.0.251。在一个没有常规DNS服务器的小型网络内，可以使用`mDNS`来实现类似DNS的编程接口、包格式和操作语义。MDNS协议的报文与DNS的报文结构相同，但有些字段对于MDNS来说有新的含义。每个进入局域网的主机，如果开启了`mDNS`服务的话，都会向局域网内的所有主机组播一个消息，我是谁，和我的IP地址是多少。然后其他也有该服务的主机就会响应，也会告诉你，它是谁，它的IP地址是多少。
> `NETBIOS`协议: 137和138端口在Linux系统中主要用于`NETBIOS`协议，其中137端口提供名称服务, 当使用者向局域网或互联网上的某台计算机的137端口发送一个请求时，可以获取该计算机的名称、注册用户名，以及是否安装主域控制器、IIS是否正在运行等信息。而138端口则用于通过网上邻居传输文件时。


在36.66上执行`ping 192.168.5.123 -c1`:
```bash
tao@S20:~$ ping 192.168.5.123 -c1
PING 192.168.5.123 (192.168.5.123) 56(84) bytes of data.
64 bytes from 192.168.5.123: icmp_seq=1 ttl=64 time=0.500 ms

--- 192.168.5.123 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.500/0.500/0.500/0.000 ms
tao@S20:~$
```

在10.161的eno2接口上抓包如下：
```bash
tao@S8:~$ sudo tcpdump -i eno2 host 10.138.36.66 -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:55:42.674055 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 92: 10.138.36.66.40103 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
4a:13:be:ec:2f:59 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.123 tell 192.168.5.121, length 28
15:55:42.674148 bb:6b:8c:86:70:de > bb:e9:75:75:60:01, ethertype IPv4 (0x0800), length 92: 10.138.10.161.57129 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 80000
16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype ARP (0x0806), length 42: Reply 192.168.5.123 is-at 16:51:8c:32:2f:e4, length 28
15:55:42.674316 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 148: 10.138.36.66.40030 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
4a:13:be:ec:2f:59 > 16:51:8c:32:2f:e4, ethertype IPv4 (0x0800), length 98: 192.168.5.121 > 192.168.5.123: ICMP echo request, id 62, seq 1, length 64
15:55:42.674368 bb:6b:8c:86:70:de > bb:e9:75:75:60:01, ethertype IPv4 (0x0800), length 148: 10.138.10.161.42413 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 80000
16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype IPv4 (0x0800), length 98: 192.168.5.123 > 192.168.5.121: ICMP echo reply, id 62, seq 1, length 64
15:55:47.687330 bb:6b:8c:86:70:de > bb:e9:75:75:60:01, ethertype IPv4 (0x0800), length 92: 10.138.10.161.57129 > 10.138.36.66.4789: VXLAN, flags [I] (0x08), vni 80000
16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.121 tell 192.168.5.123, length 28
15:55:47.687511 bb:e9:75:75:60:01 > bb:6b:8c:86:70:de, ethertype IPv4 (0x0800), length 92: 10.138.36.66.40103 > 10.138.10.161.4789: VXLAN, flags [I] (0x08), vni 80000
4a:13:be:ec:2f:59 > 16:51:8c:32:2f:e4, ethertype ARP (0x0806), length 42: Reply 192.168.5.121 is-at 4a:13:be:ec:2f:59, length 28
^C
6 packets captured
6 packets received by filter
0 packets dropped by kernel
tao@S8:~$
```

10.161的vxlan0接口上抓包如下：
```bash
tao@S8:~$ sudo tcpdump -i vxlan0 'arp or icmp' -e -n
[sudo] tao 的密码：
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on vxlan0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:55:42.674055 4a:13:be:ec:2f:59 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.123 tell 192.168.5.121, length 28
15:55:42.674124 16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype ARP (0x0806), length 42: Reply 192.168.5.123 is-at 16:51:8c:32:2f:e4, length 28
15:55:42.674316 4a:13:be:ec:2f:59 > 16:51:8c:32:2f:e4, ethertype IPv4 (0x0800), length 98: 192.168.5.121 > 192.168.5.123: ICMP echo request, id 62, seq 1, length 64
15:55:42.674357 16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype IPv4 (0x0800), length 98: 192.168.5.123 > 192.168.5.121: ICMP echo reply, id 62, seq 1, length 64
15:55:47.687308 16:51:8c:32:2f:e4 > 4a:13:be:ec:2f:59, ethertype ARP (0x0806), length 42: Request who-has 192.168.5.121 tell 192.168.5.123, length 28
15:55:47.687511 4a:13:be:ec:2f:59 > 16:51:8c:32:2f:e4, ethertype ARP (0x0806), length 42: Reply 192.168.5.121 is-at 4a:13:be:ec:2f:59, length 28
^C
6 packets captured
6 packets received by filter
0 packets dropped by kernel
tao@S8:~$
```

# Overlay
Overlay网络是通过网络虚拟化技术，在同一张Underlay网络上构建出的一张或者多张虚拟的逻辑网络。不同的Overlay网络虽然共享Underlay网络中的设备和线路，但是Overlay网络中的业务与Underlay网络中的物理组网和互联技术相互解耦。详情请参考[这里](https://info.support.huawei.com/info-finder/encyclopedia/en/Overlay+network.html "Overlay Network")
