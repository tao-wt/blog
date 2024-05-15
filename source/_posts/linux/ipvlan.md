"---
title: linux 虚拟网卡之 ipvlan
date: 2024-5-15 15:55:19
index_img: /img/index-15.jpg
tags:
  - linux
  - network
  - ipvlan
  - iptables
categories:
  - [linux, network, ipvlan]
  - [linux, netfilter, iptables]
author: tao-wt@qq.com
excerpt: 在分析K8s的网络的底层原理时，顺带将涉及到的 ipvlan 虚拟网卡技术也研究了下
---
# 简介
通过`ipvlan`，可以在一个父接口上创建出多个虚拟接口，这些虚拟接口的MAC地址和父接口的一样；虚拟网卡可以直接访问宿主机接口所连的物理网络，省去了Bridge的转发。以下是[官方介绍](https://www.kernel.org/doc/html/v5.8/networking/ipvlan.html#introduction "ipvlan.introduction")：
> This is conceptually very similar to the macvlan driver with one major exception of using L3 for mux-ing /demux-ing among slaves. This property makes the master device share the L2 with it’s slave devices. I have developed this driver in conjunction with network namespaces and not sure if there is use case outside of it.

ipvlan有三种操作模式：`L2`、`L3`和`L3s`，父网卡上创建ipvlan接口时，同一时刻只能选择其中一种模式；另外每种模式可以有三种`flags`选择：`bridge`、`private`和`vepa`；无论哪种模式哪种`flag`，`ipvlan`接口和父接口之间网络都不通，除非在父接口所在命名空间也创建一个`ipvlan`接口。本文只演示两种模式和bridge/vepa flag的情况。每种flag的介绍，请参考[官方文档](https://www.kernel.org/doc/html/v5.8/networking/ipvlan.html#mode-flags "ipvlan.flags")
> IPvlan has two modes of operation - L2 and L3. For a given master device, you can select one of these two modes and all slaves on that master will operate in the same (selected) mode. The RX mode is almost identical except that in L3 mode the slaves wont receive any multicast / broadcast traffic. L3 mode is more restrictive since routing is controlled from the other (mostly) default namespace.

# L2模式
在`L2`模式下，`ipvlan`接口组装好的报文直接通过父接口发送出去。另外，此模式下，`ipvlan`接口能收发组播和广播。
> In this mode TX processing happens on the stack instance attached to the slave device and packets are switched and queued to the master device to send out. In this mode the slaves will RX/TX multicast and broadcast (if applicable) as well.

## bridge
> This is the default option. To configure the IPvlan port in this mode, user can choose to either add this option on the command-line or don’t specify anything. This is the traditional mode where slaves can cross-talk among themselves apart from talking through the master device.

### 创建网络命名空间和ipvlan接口
分别创建两个网络命名空间和两个`ipvlan`接口，然后将ipvl1接口放入ns1中，ipvl2接口放入ns2中；父接口eno2的配置如下
```bash
tao@S3:~$
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$ sudo ip link add link eno2 name ipvl1 type ipvlan mode l2 bridge
tao@S3:~$ sudo ip link add link eno2 name ipvl2 type ipvlan mode l2 bridge
tao@S3:~$ sudo ip link set ipvl1 netns ns1
tao@S3:~$ sudo ip link set ipvl2 netns ns2
tao@S3:~$ ip a s eno2
3: eno2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether bb:6b:8c:89:88:22 brd ff:ff:ff:ff:ff:ff
    altname enp0s31f6
    inet 10.138.36.58/23 brd 10.138.37.255 scope global dynamic noprefixroute eno2
       valid_lft 46416sec preferred_lft 46416sec
    inet6 fe80::3304:c53b:493e:2f97/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
tao@S3:~$
```

ns1中的配置如下：
```bash
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
101: ipvl1@if3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether bb:6b:8c:89:88:22 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set ipvl1 up
root@S3:/home/tao# ip addr add 10.138.36.2/23 dev ipvl1
root@S3:/home/tao# ip route add default dev ipvl1
root@S3:/home/tao# ip r
default dev ipvl1 scope link
10.138.36.0/23 dev ipvl1 proto kernel scope link src 10.138.36.2
root@S3:/home/tao#
```
ns2的配置如下：
```bash
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set ipvl2 up
root@S3:/home/tao# ip addr add 10.138.36.3/23 dev ipvl2
root@S3:/home/tao# ip route add default via 10.138.36.1 dev ipvl2
root@S3:/home/tao# ip r
default via 10.138.36.1 dev ipvl2
10.138.36.0/23 dev ipvl2 proto kernel scope link src 10.138.36.3
root@S3:/home/tao# ip addr s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
102: ipvl2@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether bb:6b:8c:89:88:22 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.138.36.3/23 scope global ipvl2
       valid_lft forever preferred_lft forever
    inet6 fe80::bb6b:8c00:289:8822/64 scope link
       valid_lft forever preferred_lft forever
root@S3:/home/tao#
```

### 验证ipvlan接口间可以互通
在ns2中执行`ping 10.138.36.2`, 可以通。
```bash
root@S3:/home/tao# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
10.138.36.2              ether   bb:6b:8c:89:88:22   C                     ipvl2
root@S3:/home/tao# arp -d 10.138.36.2
root@S3:/home/tao#
root@S3:/home/tao# ping 10.138.36.2 -c1
PING 10.138.36.2 (10.138.36.2) 56(84) bytes of data.
64 bytes from 10.138.36.2: icmp_seq=1 ttl=64 time=0.092 ms

--- 10.138.36.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.092/0.092/0.092/0.000 ms
root@S3:/home/tao#
```
在父接口上抓包：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
14:37:06.875123 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
14:37:16.177226 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.2 tell 10.138.36.3, length 28
14:37:27.247251 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
14:37:27.386299 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
**注意**：父接口上只抓到了一个相关的`arp`报文，说明：
- 在`bridge`模式下，同一个父接口上的`ipvlan`接口可以直接通信，不需要经过父接口
- `ipvlan`接口共享父接口的广播域。如下在其它主机上抓包所示。

ns1中抓包如下：
```bash
root@S3:/home/tao# tcpdump -i ipvl1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C14:37:16.177161 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.2 tell 10.138.36.3, length 28
14:37:16.177189 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 10.138.36.2 is-at bb:6b:8c:89:88:22, length 28
14:37:16.177198 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.3 > 10.138.36.2: ICMP echo request, id 26381, seq 1, length 64
14:37:16.177215 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.2 > 10.138.36.3: ICMP echo reply, id 26381, seq 1, length 64
14:37:21.359841 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.3 tell 10.138.36.2, length 28
14:37:21.359875 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 10.138.36.3 is-at bb:6b:8c:89:88:22, length 28

6 packets captured
6 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
**注意**：当`arp`请求`10.138.36.3`的MAC地址时，发送的是**单播报文**
> 当系统收到`ARP`请求包时，通常会根据该包自动更新ARP表，以保持信息的最新和准确；不过，这也取决于具体的系统配置和策略。

在同一网络中的其它主机上抓包：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
14:37:03.358028 f4:e9:75:75:60:01 > dd:6b:8c:84:b8:c4, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
14:37:06.878972 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
14:37:16.181444 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.2 tell 10.138.36.3, length 46
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```
### 测试ipvlan和外网通信
在ns2中`ping`同一网络中其它主机和`baidu.com`：
```bash
root@S3:/home/tao# ping baidu.com
ping: baidu.com: Temporary failure in name resolution
root@S3:/home/tao# resolvectl dns ipvl2 10.3.3.3 10.3.3.4
Failed to set DNS configuration: Link 102 not known
root@S3:/home/tao#
root@S3:/home/tao# ping 10.138.36.55 -c1
PING 10.138.36.55 (10.138.36.55) 56(84) bytes of data.
64 bytes from 10.138.36.55: icmp_seq=1 ttl=64 time=0.960 ms

--- 10.138.36.55 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.960/0.960/0.960/0.000 ms
root@S3:/home/tao# ping 110.242.68.66 -c1
PING 110.242.68.66 (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66: icmp_seq=1 ttl=52 time=24.3 ms

--- 110.242.68.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 24.302/24.302/24.302/0.000 ms
root@S3:/home/tao#
```
**待解决问题**：怎么在网络命名空间设置域名解析呢？

在父接口抓包(包括`DNS`报文)：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp or port 53' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:29:47.469463 f4:e9:75:75:60:01 > 84:a9:38:c6:7b:13, ethertype IPv4 (0x0800), length 90: 10.3.3.3.53 > 10.138.36.185.56550: 8323* 1/0/0 A 10.25.8.60 (48)
16:29:48.167129 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.55 tell 10.138.36.3, length 28
16:29:48.167335 bb:6b:8c:86:76:92 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
16:29:48.167356 bb:6b:8c:89:88:22 > bb:6b:8c:86:76:92, ethertype IPv4 (0x0800), length 98: 10.138.36.3 > 10.138.36.55: ICMP echo request, id 21626, seq 1, length 64
16:29:48.167878 bb:6b:8c:86:76:92 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.55 > 10.138.36.3: ICMP echo reply, id 21626, seq 1, length 64
16:29:48.408284 f4:e9:75:75:60:01 > dd:6b:8c:84:b8:c4, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
16:29:50.403471 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
16:29:52.811073 bb:6b:8c:86:76:92 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.3 (bb:6b:8c:89:88:22) tell 10.138.36.55, length 46
16:29:52.811087 bb:6b:8c:89:88:22 > bb:6b:8c:86:76:92, ethertype ARP (0x0806), length 42: Reply 10.138.36.3 is-at bb:6b:8c:89:88:22, length 28
16:29:56.858903 bb:6b:8c:89:88:22 > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 10.138.36.3 > 110.242.68.66: ICMP echo request, id 58111, seq 1, length 64
16:29:56.883179 f4:e9:75:75:60:01 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 110.242.68.66 > 10.138.36.3: ICMP echo reply, id 58111, seq 1, length 64
^C
28 packets captured
29 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
另外，由于ns1的default路由设置问题(其作用域为`link`)，所以没法访问外网：
```bash
root@S3:/home/tao# ip r
default dev ipvl1 scope link
10.138.36.0/23 dev ipvl1 proto kernel scope link src 10.138.36.2
root@S3:/home/tao# ping 10.138.36.55 -c1
PING 10.138.36.55 (10.138.36.55) 56(84) bytes of data.
64 bytes from 10.138.36.55: icmp_seq=1 ttl=64 time=0.865 ms

--- 10.138.36.55 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.865/0.865/0.865/0.000 ms
root@S3:/home/tao# ping 110.242.68.66 -c1
PING 110.242.68.66 (110.242.68.66) 56(84) bytes of data.
From 10.138.36.2 icmp_seq=1 Destination Host Unreachable

--- 110.242.68.66 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

root@S3:/home/tao#
```

## vepa
> If this is added to the command-line, the port is set in VEPA mode. i.e. port will offload switching functionality to the external entity as described in 802.1Qbg Note: VEPA mode in IPvlan has limitations. IPvlan uses the mac-address of the master-device, so the packets which are emitted in this mode for the adjacent neighbor will have source and destination mac same. This will make the switch / router send the redirect message.

### 创建ipvlan接口和命名空间
`ipvlan`接口和父接口设置不通子网的ip(父接口是：10.138.36.0/23, ipvlan接口：192.168.9.0/24)。
36.58主机上的设置如下：
```bash
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$ sudo ip link add link eno2 name ipvl1 type ipvlan mode l2 vepa
tao@S3:~$ sudo ip link add link eno2 name ipvl2 type ipvlan mode l2 vepa
tao@S3:~$ sudo ip link set ipvl1 netns ns1
tao@S3:~$ sudo ip link set ipvl2 netns ns2
tao@S3:~$
tao@S3:~$ sudo ip netns exec ns1 ip link set lo up
tao@S3:~$ sudo ip netns exec ns1 ip link set ipvl1 up
tao@S3:~$ sudo ip netns exec ns1 ip addr add 192.168.9.2/24 dev ipvl1
tao@S3:~$ sudo ip netns exec ns1 ip route add default dev ipvl1
tao@S3:~$
tao@S3:~$ sudo ip netns exec ns2 ip link set lo up
tao@S3:~$ sudo ip netns exec ns2 ip link set ipvl2 up
tao@S3:~$ sudo ip netns exec ns2 ip addr add 192.168.9.3/24 dev ipvl2
tao@S3:~$ sudo ip netns exec ns2 ip r add default via 10.138.36.66 dev ipvl2
Error: Nexthop has invalid gateway.
tao@S3:~$ sudo ip netns exec ns2 ip r add default dev ipvl2
tao@S3:~$
```
**注意**：设置网关时报错，下一跳必须在同一网络中！

### 验证同一父接口上ipvlan接口的连通性
在没有外部交换机/路由器的情况下，在ns2中`ping`ns1，不通：
```bash
root@S3:/home/tao# ping 192.168.9.2 -c 1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.
From 192.168.9.3 icmp_seq=1 Destination Host Unreachable

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

root@S3:/home/tao#
```
父接口上抓包，可以看到有`arp`应答报文：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10:52:19.275108 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 192.168.9.2 is-at bb:6b:8c:89:88:22, length 28
10:52:19.275115 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28
10:52:20.303875 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 192.168.9.2 is-at bb:6b:8c:89:88:22, length 28
10:52:20.303882 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28
10:52:21.327875 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 192.168.9.2 is-at bb:6b:8c:89:88:22, length 28
10:52:21.327881 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28
^C
11 packets captured
11 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
在ns2中抓包如下所示，ipvl2接口上没有收到`arp`应答报文。说明在`vepa`模式下同一父接口下`ipvlan`接口间的链路是不通的， 需要通过外部路由器或交换机来转发：
```bash
root@S3:/home/tao# tcpdump -i ipvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C10:52:19.275053 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28
10:52:20.303836 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28
10:52:21.327836 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.3, length 28

7 packets captured
7 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
在其它主机上抓包，可以看到发出的`arp` 请求报文：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
10:52:19.280963 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.9.2 tell 192.168.9.3, length 46
10:52:20.309715 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.9.2 tell 192.168.9.3, length 46
10:52:21.333733 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.9.2 tell 192.168.9.3, length 46
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```

此时，在ns1中查看`arp`表, 存在`192.168.9.3` 条目，手动在ns2中添加`192.168.9.2`的mac地址信息，再次进行测试：
ns1中：
```nash
root@S3:/home/tao#  arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.9.3              ether   bb:6b:8c:89:88:22   C                     ipvl1
root@S3:/home/tao#
```
ns2中：
```bash
root@S3:/home/tao# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.9.2                      (incomplete)                              ipvl2
root@S3:/home/tao# arp -s 192.168.9.2 bb:6b:8c:89:88:22
root@S3:/home/tao# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.9.2              ether   bb:6b:8c:89:88:22   CM                    ipvl2
root@S3:/home/tao#
```
再次测试(在ns2中), 依旧不通：
```bash
root@S3:/home/tao# ping 192.168.9.2 -c 1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

root@S3:/home/tao#
```
在父接口上抓包：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:09:56.245678 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
12:10:10.462786 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 62626, seq 1, length 64
12:10:26.422511 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
12:10:31.481387 f4:e9:75:75:60:01 > dd:6b:8c:89:87:d1, ethertype ARP (0x0806), length 60: Request who-has 10.138.37.1 (dd:6b:8c:89:87:d1) tell 10.138.36.1, length 46
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
ns1中没有抓到包，父接口抓包情况，再次说明在`vepa`模式下同一父接口上的`ipvlan`接口之间的网路是不通的：
```bash
root@S3:/home/tao# tcpdump -i ipvl1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
另外，在其它主机上没有抓到对应的`icmp`包，因为，父接口发送的那条`icmp`数据包的目的MAC是它自己的，所以`icmp`报文并没有送父接口发送出去！
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
12:10:26.358807 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
12:10:31.417668 f4:e9:75:75:60:01 > dd:6b:8c:89:87:d1, ethertype ARP (0x0806), length 60: Request who-has 10.138.37.1 (dd:6b:8c:89:87:d1) tell 10.138.36.1, length 46
12:10:36.179203 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
^C
7 packets captured
7 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```

### 测试ipvlan接口和其它主机的连通性
在ns2中`ping 10.138.36.66`, 通：
```bash
root@S3:/home/tao# ping 10.138.36.66 -c 1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.
64 bytes from 10.138.36.66: icmp_seq=1 ttl=64 time=0.512 ms

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.512/0.512/0.512/0.000 ms
root@S3:/home/tao#
```
在ns1中ipvl1接口上抓包：
```bash
root@S3:/home/tao# tcpdump -i ipvl1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C13:11:46.209489 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:48.207614 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:49.209507 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:12:00.038607 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 192.168.9.3, length 28

4 packets captured
4 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
在目标主机(36.66)上抓包，并反向`ping` ns2，通。说明在`vepa` + `l2`模式下`ipvlan`接口和父接口共享同一广播域：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
13:11:46.193683 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:48.191807 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:49.193668 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:54.910861 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
13:11:59.416243 f4:e9:75:75:60:01 > bb:6b:8c:86:78:83, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
13:12:00.022803 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.66 tell 192.168.9.3, length 46
13:12:00.022844 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 28
13:12:00.022951 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 32399, seq 1, length 64
13:12:00.023010 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 32399, seq 1, length 64
^C
9 packets captured
16 packets received by filter
0 packets dropped by kernel
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.9.0/24 via 10.138.36.58 dev eno1
tao@S20:~$ ping 192.168.9.3 -c 1
PING 192.168.9.3 (192.168.9.3) 56(84) bytes of data.
64 bytes from 192.168.9.3: icmp_seq=1 ttl=64 time=0.178 ms

--- 192.168.9.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.178/0.178/0.178/0.000 ms
tao@S20:~$
```
在ns2中抓包，有对应的`arp`报文，说明在`l2`模式下`ipvlan`接口可以收到广播报文：
```bash
root@S3:/home/tao# tcpdump -i ipvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C13:11:46.209494 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:48.207619 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:11:49.209512 f4:e9:75:75:60:01 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.147 tell 10.138.36.1, length 46
13:12:00.038424 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 192.168.9.3, length 28
13:12:00.038758 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 46
13:12:00.038778 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 32399, seq 1, length 64
13:12:00.038906 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 32399, seq 1, length 64

7 packets captured
7 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```

# L3模式
在`L3`模式下，`ipvlan`接口报文链路层的处理是在父接口对应网络协议栈上，并且会经过一次父接口网络栈的路由；另外，`L3`模式下，`ipvlan`接口不能收发多播和广播报文。
> In this mode TX processing up to L3 happens on the stack instance attached to the slave device and packets are switched to the stack instance of the master device for the L2 processing and routing from that instance will be used before packets are queued on the outbound device. In this mode the slaves will not receive nor can send multicast / broadcast traffic.

## bridge
### 创建ipvlan接口和相应网络命名空间
整体配置和上面`L2`模式类似：
```bash
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$
tao@S3:~$ sudo ip link add link eno2 name ipvl1 type ipvlan mode l3 bridge
tao@S3:~$ sudo ip link add link eno2 name ipvl2 type ipvlan mode l3 bridge
tao@S3:~$ sudo ip link set ipvl1 netns ns1
tao@S3:~$ sudo ip link set ipvl2 netns ns2
tao@S3:~$ sudo ip netns exec ns1 ip link set lo up
tao@S3:~$ sudo ip netns exec ns1 ip link set ipvl1 up
tao@S3:~$ sudo ip netns exec ns1 ip addr add 192.168.9.2/24 dev ipvl1
tao@S3:~$ sudo ip netns exec ns1 ip route add default dev ipvl1
tao@S3:~$
tao@S3:~$ sudo ip netns exec ns2 ip link set lo up
tao@S3:~$ sudo ip netns exec ns2 ip link set ipvl2 up
tao@S3:~$ sudo ip netns exec ns2 ip addr add 192.168.9.3/24 dev ipvl2
tao@S3:~$ sudo ip netns exec ns2 ip r add default dev ipvl2
tao@S3:~$
```

### 检验ipvlan接口之间的连通性
在ns2中`ping 192.168.9.2`, 可以通。注意ipvl2接口的状态字段: `NOARP`。
```bash
root@S3:/home/tao# ip a s ipvl2
104: ipvl2@if3: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether bb:6b:8c:89:88:22 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.9.3/24 scope global ipvl2
       valid_lft forever preferred_lft forever
    inet6 fe80::bb6b:8c00:289:8822/64 scope link
       valid_lft forever preferred_lft forever
root@S3:/home/tao# ping 192.168.9.2 -c 1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.
64 bytes from 192.168.9.2: icmp_seq=1 ttl=64 time=0.073 ms

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.073/0.073/0.073/0.000 ms
root@S3:/home/tao#
```
在父接口上抓包， 没有`arp`报文，也没有`icmp`报文：
```bash
tao@S3:~$ sudo tcpdump -i eno2 "icmp or arp" -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:25:25.939953 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
19:25:25.975000 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Reply 10.138.36.55 is-at bb:6b:8c:86:76:92, length 46
19:25:29.387931 f4:e9:75:75:60:01 > bb:6b:8c:86:78:83, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
^C
3 packets captured
3 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
在ns1中抓包，只有`icmp`报文，没有`arp`报文，说明同一父接口`ipvlan`接口之间的通信只发生在`ip`层：
```bash
root@S3:/home/tao# tcpdump -i ipvl1 "icmp or arp" -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C19:25:33.869373 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 9857, seq 1, length 64
19:25:33.869395 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo reply, id 9857, seq 1, length 64

2 packets captured
2 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao# 
```

### ipvlan接口和外部主机的通信
在ns2中`ping 10.138.36.66`，36.66和父接口处于同一网络，但和`ipvlan`接口不是同一子网，所以正常情况下是不通的:
```bash
root@S3:/home/tao# ping 10.138.36.66 -c 1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

root@S3:/home/tao#
```
在父接口上抓包；对应`arp`报文父接口发的，说明数据包从链路层往下的处理发生在父接口！另外在ns1的ipvl1接口没有抓取到任何报文。
```bash
tao@S3:~$ sudo tcpdump -i eno2 "icmp or arp" -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
19:42:41.012348 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 10.138.36.58, length 28
19:42:41.012500 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 46
19:42:41.012522 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2304, seq 1, length 64
19:42:46.469102 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
在ns2中抓包；目的MAC和源MAC相同，说明`ipvlan`接口发送数据包时，还是会完整组包的。
```
root@S3:/home/tao# tcpdump -i ipvl2 -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C19:42:41.509829 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2304, seq 1, length 64

1 packet captured
1 packet received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
在目标主机上抓包，`arp`交互报文和`ICMP echo request`报文已收到；注意，应答报文也发送了，其目的MAC是网关36.1的MAC，说明应答报文走了默认路由！
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
19:42:41.020445 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.66 tell 10.138.36.58, length 46
19:42:41.020483 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 42: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 28
19:42:41.020571 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2304, seq 1, length 64
19:42:41.020620 bb:6b:8c:86:6d:83 > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 2304, seq 1, length 64
19:42:46.477106 bb:6b:8c:86:76:92 > dd:70:b8:09:81:39, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.163 (dd:70:b8:09:81:39) tell 10.138.36.55, length 46
^C
9 packets captured
9 packets received by filter
0 packets dropped by kernel
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
tao@S20:~$ ip neigh
10.138.36.2 dev eno1 lladdr bb:6b:8c:89:88:22 STALE
10.138.36.1 dev eno1 lladdr f4:e9:75:75:60:01 REACHABLE
fe80::18a3:398f:9a50:3d3a dev eno1 lladdr 68:da:73:aa:3d:90 STALE
tao@S20:~$
```
从上面的分析可以得出：如果`ipvlan`接口使用使用了不同的子网，要使`ipvlan`接口和父接口所在网络其它主机通信，需要在外部主机上添加路由。在36.66上添加路由：
```bash
tao@S20:~$ sudo ip route add 192.168.9.0/24 via 10.138.36.58 dev eno1
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 20100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.9.0/24 via 10.138.36.58 dev eno1
tao@S20:~$
tao@S20:~$ ping 192.168.9.2 -c 1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.
64 bytes from 192.168.9.2: icmp_seq=1 ttl=64 time=0.221 ms

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.221/0.221/0.221/0.000 ms
tao@S20:~$
```
现在在ns2中就可以`ping`通36.66了，反之亦然，如上所示。
```
root@S3:/home/tao# ping 10.138.36.66 -c 1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.
64 bytes from 10.138.36.66: icmp_seq=1 ttl=64 time=0.244 ms

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.244/0.244/0.244/0.000 ms
root@S3:/home/tao#
```

## vepa
### 创建ipvlan接口和相应网络命名空间
创建命名空间和`ipvlan`接口：
```bash
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$ sudo ip link add link eno2 name ipvl1 type ipvlan mode l3 vepa
tao@S3:~$ sudo ip link add link eno2 name ipvl2 type ipvlan mode l3 vepa
tao@S3:~$ sudo ip link set ipvl1 netns ns1
tao@S3:~$ sudo ip link set ipvl2 netns ns2
tao@S3:~$
tao@S3:~$ sudo ip netns exec ns1 ip link set lo up
tao@S3:~$ sudo ip netns exec ns1 ip link set ipvl1 up
tao@S3:~$ sudo ip netns exec ns1 ip addr add 192.168.9.2/24 dev ipvl1
tao@S3:~$ sudo ip netns exec ns1 ip r add default dev ipvl1
tao@S3:~$
tao@S3:~$ sudo ip netns exec ns2 ip link set lo up
tao@S3:~$ sudo ip netns exec ns2 ip link set ipvl2 up
tao@S3:~$ sudo ip netns exec ns2 ip addr add 192.168.9.3/24 dev ipvl2
tao@S3:~$ sudo ip netns exec ns2 ip r add default dev ipvl2
tao@S3:~$
```
### 验证ipvlan接口间的互通性
在ns2中`ping`ns1，在没有外部交换机或路由的情况下是`ping`不通的：
```bash
root@S3:/home/tao# ping 192.168.9.2 -c1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

root@S3:/home/tao#
```
ns2中抓包如下，在`l3`模式下，没有`arp`广播报文产生：
```bash
root@S3:/home/tao# tcpdump -i ipvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C15:20:33.152259 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 40524, seq 1, length 64

1 packet captured
1 packet received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
在父接口抓包如下，可以看到ipvl2发出的`icmp`报文走了父接口所在命名空间的默认路由：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'arp or icmp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:20:29.368029 f4:e9:75:75:60:01 > bb:6b:8c:86:78:83, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
15:20:33.152287 bb:6b:8c:89:88:22 > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 40524, seq 1, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
tao@S3:~$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
10.138.36.66             ether   bb:6b:8c:86:6d:83   C                     eno2
10.138.36.1              ether   f4:e9:75:75:60:01   C                     eno2
tao@S3:~$ ip r
default via 10.138.36.1 dev eno2 proto dhcp metric 100
10.138.36.0/23 dev eno2 proto kernel scope link src 10.138.36.58 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.19.0.0/16 dev br-9bf9aa158c15 proto kernel scope link src 172.19.0.1
tao@S3:~$
```
怎么让这两个`ipvlan`接口能`ping`通呢？需要在一台外部主机配置路由转发功能将36.58上父接口发来的包转发回去就可以了：
在父接口所在命名空间配置路由，将目的网络为`192.168.9.0/24`的数据包发往`10.138.36.66`：
```bash
tao@S3:~$ sudo ip r add 192.168.9.0/24 via 10.138.36.66 dev eno2
tao@S3:~$ ip r
default via 10.138.36.1 dev eno2 proto dhcp metric 100
10.138.36.0/23 dev eno2 proto kernel scope link src 10.138.36.58 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.19.0.0/16 dev br-9bf9aa158c15 proto kernel scope link src 172.19.0.1
192.168.9.0/24 via 10.138.36.66 dev eno2
tao@S3:~$
```
在36.66外部主机上上开启路由转发功能，并配置防火墙和路由规则，将`192.168.9.0/24`网段的流量转发回36.58：
```bash
tao@S20:~$ sudo iptables -I FORWARD 1 -i eno1 -o eno1 -j ACCEPT
tao@S20:~$ sudo iptables -L -nv
Chain INPUT (policy ACCEPT 242 packets, 24175 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  eno1   eno1    0.0.0.0/0            0.0.0.0/0
  138 11592 DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
  138 11592 DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.9.0/24 via 10.138.36.58 dev eno1
tao@S20:~$
```
现在`ipvlan`接口间就可以`ping`通了， 如下：
```bash
root@S3:/home/tao# ping 192.168.9.2 -c1
PING 192.168.9.2 (192.168.9.2) 56(84) bytes of data.
64 bytes from 192.168.9.2: icmp_seq=1 ttl=63 time=0.350 ms

--- 192.168.9.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.350/0.350/0.350/0.000 ms
root@S3:/home/tao#
```
在ns1和ns2中抓包的报文一样；并且都不存在`arp`广播报文；另外注意`icmp`应答报文的源MAC地址是36.66的MAC地址：
```bash
root@S3:/home/tao# tcpdump -i ipvl1 'arp or icmp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C15:45:39.231038 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 46924, seq 1, length 64
15:45:39.231084 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo reply, id 46924, seq 1, length 64

2 packets captured
2 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
在36.66上抓包可以看到，相关报文是双份的，因为报文经过了接口两次；所以在`vepa`模式下如果同一父接口的`ipvlan`接口交互频繁会对网络环境产生一定影响：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
15:45:39.210854 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 46924, seq 1, length 64
15:45:39.210898 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo request, id 46924, seq 1, length 64
15:45:39.211034 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo reply, id 46924, seq 1, length 64
15:45:39.211050 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo reply, id 46924, seq 1, length 64
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```
### 测试和同一网络内设备的连通性
在ns2中执行`ping 10.138.36.66`，通：
```bash
root@S3:/home/tao# ping 10.138.36.66 -c1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.
64 bytes from 10.138.36.66: icmp_seq=1 ttl=64 time=0.202 ms

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.202/0.202/0.202/0.000 ms
root@S3:/home/tao#
```
在ns2中抓包：
```bash
root@S3:/home/tao# tcpdump -i ipvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C16:01:07.506322 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2748, seq 1, length 64
16:01:07.506505 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 2748, seq 1, length 64

2 packets captured
2 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao# arp -n
root@S3:/home/tao#
```
在父接口抓包， 报文走的是第二条路由：
```
tao@S3:~$ ip r
default via 10.138.36.1 dev eno2 proto dhcp metric 100
10.138.36.0/23 dev eno2 proto kernel scope link src 10.138.36.58 metric 100
192.168.9.0/24 via 10.138.36.66 dev eno2
tao@S3:~$ sudo tcpdump -i eno2 'arp or icmp' -e -n
[sudo] password for tao:
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:01:07.506355 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2748, seq 1, length 64
16:01:07.506505 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 2748, seq 1, length 64
^C
12 packets captured
12 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
在目标主机36.66上抓包：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
[sudo] password for tao:
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
16:01:07.485695 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 10.138.36.66: ICMP echo request, id 2748, seq 1, length 64
16:01:07.485760 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.3: ICMP echo reply, id 2748, seq 1, length 64
^C
12 packets captured
12 packets received by filter
0 packets dropped by kernel
tao@S20:~$ ip link show eno1
2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    link/ether bb:6b:8c:86:6d:83 brd ff:ff:ff:ff:ff:ff
    altname enp4s0
tao@S20:~$
```

### 测试和公网的连通性
在ns2中执行`ping 39.156.66.10`，不通：
```bash
root@S3:/home/tao# ping 39.156.66.10 -c1
PING 39.156.66.10 (39.156.66.10) 56(84) bytes of data.

--- 39.156.66.10 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

root@S3:/home/tao#
```
父接口上抓包如下，可以看到`icmp`请求报文已经发送，但是由于网关36.1上没有配置到`192.168.9.0/24`网络的路由，所以没法收到应答报文：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'arp or icmp' -e -n
[sudo] password for tao:
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:16:13.911247 bb:6b:8c:89:88:22 > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 39.156.66.10: ICMP echo request, id 6307, seq 1, length 64
16:16:29.355347 f4:e9:75:75:60:01 > bb:6b:8c:86:78:83, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
解决：在父接口所在命名空间添加路由，将发往39.156.66.10的报文转发到36.66：
```bash
tao@S3:~$ sudo ip r add 39.156.66.10/32 via 10.138.36.66 dev eno2
tao@S3:~$ ip r
default via 10.138.36.1 dev eno2 proto dhcp metric 100
10.138.36.0/23 dev eno2 proto kernel scope link src 10.138.36.58 metric 100
39.156.66.10 via 10.138.36.66 dev eno2
192.168.9.0/24 via 10.138.36.66 dev eno2
tao@S3:~$
```
在36.66上配置防火墙规则(`nat`表)，在路由前后转换目的ip和源ip：
```bash
tao@S20:~$ sudo iptables -t nat -I POSTROUTING 1 -o eno1 -s 192.168.9.0/24 -d 39.156.66.10 -j MASQUERADE
tao@S20:~$ sudo iptables -t nat -I PREROUTING 1 -i eno1 -s 39.156.66.10 -j DNAT --to 192.168.9.3
tao@S20:~$ sudo iptables -t nat -L -nv --line-numbers
Chain PREROUTING (policy ACCEPT 2990 packets, 128K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DNAT       all  --  eno1   *       39.156.66.10         0.0.0.0/0            to:192.168.9.3
2     5002  372K DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 2978 packets, 126K bytes)
num   pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 449 packets, 38063 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1      151  9060 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 449 packets, 38063 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        1    84 MASQUERADE  all  --  *      eno1    192.168.9.0/24       39.156.66.10
2        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0

Chain DOCKER (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0
tao@S20:~$
```
现在ns2中命令`ping 39.156.66.10`就可以通了：
```bash
root@S3:/home/tao# ping 39.156.66.10 -c1
PING 39.156.66.10 (39.156.66.10) 56(84) bytes of data.
64 bytes from 39.156.66.10: icmp_seq=1 ttl=45 time=30.6 ms

--- 39.156.66.10 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.558/30.558/30.558/0.000 ms
root@S3:/home/tao#
```
在36.66上抓包，可以看到已成功的进行了地址转换，从中可以看到tcpdump抓取的报文是在`NAT`地址转换之前；另外一点是36.66其实还是走了默认网关36.1：
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
16:40:23.870620 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 39.156.66.10: ICMP echo request, id 22349, seq 1, length 64
16:40:23.870683 bb:6b:8c:86:6d:83 > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 39.156.66.10: ICMP echo request, id 22349, seq 1, length 64
16:40:23.900931 f4:e9:75:75:60:01 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 39.156.66.10 > 10.138.36.66: ICMP echo reply, id 22349, seq 1, length 64
16:40:23.900980 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 39.156.66.10 > 192.168.9.3: ICMP echo reply, id 22349, seq 1, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```
在ns2中抓包：
```bash
root@S3:/home/tao# tcpdump -i ipvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ipvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C16:40:23.896060 bb:6b:8c:89:88:22 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 39.156.66.10: ICMP echo request, id 22349, seq 1, length 64
16:40:23.926600 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 39.156.66.10 > 192.168.9.3: ICMP echo reply, id 22349, seq 1, length 64

2 packets captured
2 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```
验证结束，清理防火墙规则：
```
tao@S20:~$ sudo iptables -t nat -D PREROUTING 1
tao@S20:~$ sudo iptables -t nat -D POSTROUTING 1
tao@S20:~$ sudo iptables -t filter -D FORWARD 1
```
