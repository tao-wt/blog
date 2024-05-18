"---
title: linux 虚拟网卡之 macvlan
date: 2024-5-17 15:21:30
index_img: /img/index-16.jpg
tags:
  - linux
  - network
  - macvlan
  - vlan
  - veth
  - iptables
categories:
  - [linux, network, macvlan]
  - [linux, network, vlan]
  - [linux, network, bridge]
  - [linux, network, veth]
author: tao-wt@qq.com
excerpt: 由于上一篇对 ipvlan 的认识，这篇主要是研究和尝试另一种相似的虚拟网卡技术：macvlan 
---
# 原理说明
通过`Macvlan`可以在一个网络接口上创建多个虚拟接口，这些虚拟接口有自己的`MAC`地址，可以配置 IP 进行通信。虚拟接口和父接口共享同一个**广播域**。

`Macvlan`和`Bridge`相似，但省去了`Bridge`的存在，所以配置和调试起来比较简单，效率也相对高。此外，`Macvlan`完美支持`VLAN`。
> Before MACVLAN, if you wanted to connect to physical network from a VM or namespace, you would have needed to create TAP/VETH devices and attach one side to a bridge and attach a physical interface to the bridge on the host at the same time...

物理网卡相当于一个交换机，记录着对应的虚拟网卡的MAC 地址，当收到数据包时，它会根据目的 MAC 判断这个包属于哪一个虚拟网卡。这就意味着，只要是从 Macvlan 子接口发来的数据包（或者是发往 Macvlan 子接口的数据包），物理网卡只转发数据包，而不处理数据包；这导致：本机 Macvlan 网卡上的 IP 无法和物理网卡上面的 IP 直接通信！这和`ipvlan`一样。
> Mavlan sub-interfaces are not able to directly communicate with the parent interface , i.e. VMs cannot directly communicate with the host. If you require VM-host communication, you should add another macvlan sub-interface and assign it to the host.

# 模式
`macvlan`有四种模式：`private`、`VEPA`(default mode)、`Bridge`和`Passthru`。具体每种模式的说明请[参考这里](https://yxj-books.readthedocs.io/zh-cn/latest/%E7%BD%91%E7%BB%9C%E5%86%85%E6%A0%B8/net_device/macvlan/macvlan-info.html)。

> Macvlan sub-interfaces use a **mac0@eth0** notation, to clearly identify the sub-interface and it’s parent interface. Sub-interface state is bound to its parent’s state – if eth0 is down, so is the **mac0@eth0**.

## vepa
`vepa`模式的测试拓扑如下：
![vepa测试拓扑图](/img/vepa_topo.png)
> 在这种拓扑下`bridge`上的`veth`接口其实就相当于物理`交换机`的`trunk`类型接口!

### 环境配置
首先创建`bridge`：test_br，两个`veth`设备对，并将`veth`设备对的一端加入 test_br，打开`bridge`上两个接口的`hairpin`配置。
```sh
tao@S3:~$ sudo ip link add test_br type bridge
tao@S3:~$ sudo ip link add name veth1 type veth peer name veth1_br
tao@S3:~$ sudo ip link add name veth2 type veth peer name veth2_br
tao@S3:~$ sudo ip link set veth1_br master test_br
tao@S3:~$ sudo ip link set veth2_br master test_br
tao@S3:~$ sudo ip link set test_br up
tao@S3:~$ sudo ip link set veth1_br up
tao@S3:~$ sudo ip link set veth2_br up
tao@S3:~$ sudo brctl hairpin test_br veth1_br on
tao@S3:~$ sudo brctl hairpin test_br veth2_br on
tao@S3:~$ sudo ip -d link show master test_br veth1_br
18: veth1_br@veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master test_br state UP mode DEFAULT group default qlen 1000
    link/ether 06:0f:29:dd:c3:60 brd ff:ff:ff:ff:ff:ff promiscuity 1 minmtu 68 maxmtu 65535
    veth
    bridge_slave state forwarding priority 32 cost 2 hairpin on guard off root_block off fastleave off......
tao@S3:~$ 
```

在两个`veth`设备对的另一端各自创建一个`vlan`子接口，并打开其混杂模式；注意,`vlan`子接口和父接口的MAC地址一样:
```sh
tao@S3:~$ sudo ip link set veth1 up
tao@S3:~$ sudo ip link set veth2 up
tao@S3:~$ sudo ip link add link veth1 name veth1.11 type vlan id 11
tao@S3:~$ sudo ip link add link veth2 name veth2.12 type vlan id 12
tao@S3:~$ sudo ip link set veth1.11 up
tao@S3:~$ sudo ip link set veth2.12 up
tao@S3:~$ sudo ip link set veth1.11 promisc on
tao@S3:~$ sudo ip link set veth2.12 promisc on
tao@S3:~$ ip link show veth1
19: veth1@veth1_br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 22:77:b6:04:77:d2 brd ff:ff:ff:ff:ff:ff
tao@S3:~$ ip link show veth1.11
22: veth1.11@veth1: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 22:77:b6:04:77:d2 brd ff:ff:ff:ff:ff:ff
tao@S3:~$ ip link show veth2
21: veth2@veth2_br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f2:b0:b8:d0:ef:b3 brd ff:ff:ff:ff:ff:ff
tao@S3:~$ ip link show veth2.12
23: veth2.12@veth2: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f2:b0:b8:d0:ef:b3 brd ff:ff:ff:ff:ff:ff
tao@S3:~$
```
配置防火墙规则允许bridge test_br转发数据：
```bash
tao@S3:~$ sudo iptables -I FORWARD 1 -i test_br -j ACCEPT
tao@S3:~$ sudo iptables -I FORWARD 1 -o test_br -j ACCEPT
tao@S3:~$ sudo iptables -L FORWARD -nv
Chain FORWARD (policy DROP 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     all  --  *      test_br  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  test_br *       0.0.0.0/0            0.0.0.0/0
37731 4269K DOCKER-USER  all  --  *      *       0.0.0.0/0            0.0.0.0/0
37731 4269K DOCKER-ISOLATION-STAGE-1  all  --  *      *       0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
    0     0 DOCKER     all  --  *      docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 !docker0  0.0.0.0/0            0.0.0.0/0
    0     0 ACCEPT     all  --  docker0 docker0  0.0.0.0/0            0.0.0.0/0
tao@S3:~$
```

创建4个网络命名空间，并分别基于veth1.11和veth2.12各创建两个`macvlan`接口，放入对应命名空间:
```bash
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$ sudo ip netns add ns3
tao@S3:~$ sudo ip netns add ns4
tao@S3:~$ sudo ip link add link veth1.11 name macvl1 type macvlan mode vepa
tao@S3:~$ sudo ip link add link veth1.11 name macvl2 type macvlan mode vepa
tao@S3:~$ sudo ip link set macvl1 netns ns1
tao@S3:~$ sudo ip link set macvl2 netns ns2
tao@S3:~$
tao@S3:~$ sudo ip link add link veth2.12 name macvl3 type macvlan mode vepa
tao@S3:~$ sudo ip link add link veth2.12 name macvl4 type macvlan mode vepa
tao@S3:~$ sudo ip link set macvl3 netns ns3
tao@S3:~$ sudo ip link set macvl4 netns ns4
tao@S3:~$
```

ns1的配置如下：
```sh
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
24: macvl1@if22: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 9a:e4:69:98:8f:e3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl1 up
root@S3:/home/tao# ip addr add 192.168.9.2/24 dev macvl1
root@S3:/home/tao# ip r add default dev macvl1
root@S3:/home/tao# ip r
default dev macvl1 scope link
192.168.9.0/24 dev macvl1 proto kernel scope link src 192.168.9.2
root@S3:/home/tao#
```
ns2配置如下：
```sh
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
25: macvl2@if22: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 82:a3:b5:0f:fc:0c brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl2 up
root@S3:/home/tao# ip addr add 192.168.9.3/24 dev macvl2
root@S3:/home/tao# ip route add default dev macvl2
root@S3:/home/tao# ip r
default dev macvl2 scope link
192.168.9.0/24 dev macvl2 proto kernel scope link src 192.168.9.3
root@S3:/home/tao#
```
ns3的配置如下：
```sh
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
26: macvl3@if23: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f6:e7:8e:3f:54:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl3 up
root@S3:/home/tao# ip a add 192.168.9.2/24 dev macvl3
root@S3:/home/tao# ip r add default dev macvl3
root@S3:/home/tao# ip r
default dev macvl3 scope link
192.168.9.0/24 dev macvl3 proto kernel scope link src 192.168.9.2
root@S3:/home/tao#
```
ns4配置如下：
```sh
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
27: macvl4@if23: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether d6:2b:b6:7a:33:4f brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl4 up
root@S3:/home/tao# ip a add 192.168.9.3/24 dev macvl4
root@S3:/home/tao# ip r add default dev macvl4
root@S3:/home/tao# ip r
default dev macvl4 scope link
192.168.9.0/24 dev macvl4 proto kernel scope link src 192.168.9.3
root@S3:/home/tao#
```
### 验证同一父接口下macvlan子接口的连通性
在ns1中执行：`ping 192.168.9.3`，可以看到在bridge的作用下同一父接口的macvlan接口之间是通的。可以看到`192.168.9.3`的mac地址正是是ns2中macvl2接口的mac地址，而不是ns4中macvl4的。
```sh
root@S3:/home/tao# ping 192.168.9.3 -c1
PING 192.168.9.3 (192.168.9.3) 56(84) bytes of data.
64 bytes from 192.168.9.3: icmp_seq=1 ttl=64 time=0.221 ms

--- 192.168.9.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.221/0.221/0.221/0.000 ms
root@S3:/home/tao# ip neigh
192.168.9.3 dev macvl1 lladdr 82:a3:b5:0f:fc:0c STALE
root@S3:/home/tao#
```

验证一下，在网桥中的veth1_br接口上抓包，可以看到数据帧都带有`vlan tag`，并且又都发回veth1_br接口，所以数据包是双份的：
> **注意**：`bridge`等桥接设备在转发数据包时源MAC和目的MAC不会改变！
```bash
tao@S3:~$ sudo tcpdump -i veth1_br 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth1_br, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:50:00.478919 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Request who-has 192.168.9.3 tell 192.168.9.2, length 28
16:50:00.478950 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Request who-has 192.168.9.3 tell 192.168.9.2, length 28
16:50:00.479013 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Reply 192.168.9.3 is-at 82:a3:b5:0f:fc:0c, length 28
16:50:00.479033 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Reply 192.168.9.3 is-at 82:a3:b5:0f:fc:0c, length 28
16:50:00.479048 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype 802.1Q (0x8100), length 102: vlan 11, p 0, ethertype IPv4 (0x0800), 192.168.9.2 > 192.168.9.3: ICMP echo request, id 16079, seq 1, length 64
16:50:00.479057 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype 802.1Q (0x8100), length 102: vlan 11, p 0, ethertype IPv4 (0x0800), 192.168.9.2 > 192.168.9.3: ICMP echo request, id 16079, seq 1, length 64
16:50:00.479087 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 102: vlan 11, p 0, ethertype IPv4 (0x0800), 192.168.9.3 > 192.168.9.2: ICMP echo reply, id 16079, seq 1, length 64
16:50:00.479094 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 102: vlan 11, p 0, ethertype IPv4 (0x0800), 192.168.9.3 > 192.168.9.2: ICMP echo reply, id 16079, seq 1, length 64
16:50:05.709259 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Request who-has 192.168.9.2 tell 192.168.9.3, length 28
16:50:05.709272 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Request who-has 192.168.9.2 tell 192.168.9.3, length 28
16:50:05.709297 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Reply 192.168.9.2 is-at 9a:e4:69:98:8f:e3, length 28
16:50:05.709299 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype 802.1Q (0x8100), length 46: vlan 11, p 0, ethertype ARP (0x0806), Reply 192.168.9.2 is-at 9a:e4:69:98:8f:e3, length 28
^C
12 packets captured
12 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

在veth2.12和ns4中的macvl4接口上都没有抓到数据包：
> 说明，对于`vlan`子接口来说，非此vlan的数据包已经在父接口被丢弃！
```sh
tao@S3:~$ sudo tcpdump -i veth2.12 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth2.12, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

### 验证跨父接口macvlan接口的连通性
在veth2接口上再创建一个`vlan id`为11的`vlan`子接口：veth2.11，在其上创建`macvlan`子接口，并将其移动到ns5命名空间：
```bash
tao@S3:~$ sudo ip link add link veth2 name veth2.11 type vlan id 11
tao@S3:~$ sudo ip link set veth2.11 up
tao@S3:~$ sudo ip link add link veth2.11 name macvl5 type macvlan mode vepa
tao@S3:~$ sudo ip netns add ns5
tao@S3:~$ sudo ip link set macvl5 netns ns5
tao@S3:~$
```

ns5配置如下：
```bash
root@S3:/home/tao# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
29: macvl5@if28: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 06:d9:4d:a5:a1:1e brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl5 up
root@S3:/home/tao# ip addr add 192.168.9.5/24 dev macvl5
root@S3:/home/tao# ip route add default dev macvl5
root@S3:/home/tao# ip r
default dev macvl5 scope link
192.168.9.0/24 dev macvl5 proto kernel scope link src 192.168.9.5
root@S3:/home/tao#
```

在ns1中执行```ping 192.168.9.5```，通:
```sh
root@S3:/home/tao# ping 192.168.9.5 -c1
PING 192.168.9.5 (192.168.9.5) 56(84) bytes of data.
64 bytes from 192.168.9.5: icmp_seq=1 ttl=64 time=0.208 ms

--- 192.168.9.5 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.208/0.208/0.208/0.000 ms
root@S3:/home/tao#
```

在veth2.11上抓包，可以看到`vlan子接口`上的`vlan tag`信息已经被去掉了！
> 当父接口接收报文时，如果是802.1Q报文，则会根据VLAN ID将报文发到对应的子接口。
> 当子接口发送报文时，内核会在报文中添加802.1Q头，然后交由父接口完成发送。接收报文时，Linux内核在将帧从父接口转发到VLAN子接口时，会自动移除VLAN标签。

```bash
tao@S3:~$ sudo tcpdump -i veth2.11 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth2.11, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:21:48.904283 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.5 tell 192.168.9.2, length 28
18:21:48.904334 06:d9:4d:a5:a1:1e > 9a:e4:69:98:8f:e3, ethertype ARP (0x0806), length 42: Reply 192.168.9.5 is-at 06:d9:4d:a5:a1:1e, length 28
18:21:48.904383 9a:e4:69:98:8f:e3 > 06:d9:4d:a5:a1:1e, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.5: ICMP echo request, id 34593, seq 1, length 64
18:21:48.904413 06:d9:4d:a5:a1:1e > 9a:e4:69:98:8f:e3, ethertype IPv4 (0x0800), length 98: 192.168.9.5 > 192.168.9.2: ICMP echo reply, id 34593, seq 1, length 64
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

## private
`private`模式的测试拓扑如下：
![private测试拓扑图](/img/private_topo.png)

### 环境配置
首先清理`vepa`模式下创建的环境：
```sh
tao@S3:~$ sudo ip netns del ns1
tao@S3:~$ sudo ip netns del ns2
tao@S3:~$ sudo ip netns del ns3
tao@S3:~$ sudo ip netns del ns4
tao@S3:~$ sudo ip netns del ns5
tao@S3:~$ sudo ip link del veth2.11
tao@S3:~$ sudo ip link del veth2.12
tao@S3:~$ sudo ip link del veth1.11
tao@S3:~$
```
按上面拓扑创建环境，基于veth1和veth2创建3个macvlan接口放入三个命名空间，并打开两个父接口的混杂模式：
```bash
tao@S3:~$ sudo ip link add link veth1 name macvl1 type macvlan mode private
tao@S3:~$ sudo ip link add link veth1 name macvl2 type macvlan mode private
tao@S3:~$ sudo ip link add link veth2 name macvl3 type macvlan mode private
tao@S3:~$ sudo ip netns add ns1
tao@S3:~$ sudo ip netns add ns2
tao@S3:~$ sudo ip netns add ns3
tao@S3:~$ sudo ip link set macvl1 netns ns1
tao@S3:~$ sudo ip link set macvl2 netns ns2
tao@S3:~$ sudo ip link set macvl3 netns ns3
tao@S3:~$ sudo ip link set veth1 promisc on
tao@S3:~$ sudo ip link set veth2 promisc on
tao@S3:~$ ip link show veth1
19: veth1@veth1_br: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 22:77:b6:04:77:d2 brd ff:ff:ff:ff:ff:ff
tao@S3:~$ ip link show veth2
21: veth2@veth2_br: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether f2:b0:b8:d0:ef:b3 brd ff:ff:ff:ff:ff:ff
tao@S3:~$
```
ns1的配置如下：
```sh
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl1 up
root@S3:/home/tao# ip addr add 192.168.9.2/24 dev macvl1
root@S3:/home/tao# ip r
192.168.9.0/24 dev macvl1 proto kernel scope link src 192.168.9.2
root@S3:/home/tao# ip link show macvl1
30: macvl1@if19: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 9a:e4:69:98:8f:e3 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao#
```
ns2配置如下：
```sh
root@S3:/home/tao# ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
31: macvl2@if19: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 82:a3:b5:0f:fc:0c brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl2 up
root@S3:/home/tao# ip addr add 192.168.9.3/24 dev macvl2
root@S3:/home/tao# ip r
192.168.9.0/24 dev macvl2 proto kernel scope link src 192.168.9.3
root@S3:/home/tao#
```
ns3配置如下：
```sh
root@S3:/home/tao# ip link show macvl3
32: macvl3@if21: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether f6:e7:8e:3f:54:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl3 up
root@S3:/home/tao# ip addr add 192.168.9.5/24 dev macvl3
root@S3:/home/tao# ip r
192.168.9.0/24 dev macvl3 proto kernel scope link src 192.168.9.5
root@S3:/home/tao#
```

### 测试同一父接口下macvlan接口连通性
在ns1中执行`ping 192.168.9.3`，不通！
```sh
root@S3:/home/tao# ping 192.168.9.3 -c1
PING 192.168.9.3 (192.168.9.3) 56(84) bytes of data.
From 192.168.9.2 icmp_seq=1 Destination Host Unreachable

--- 192.168.9.3 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

root@S3:/home/tao#
```
在ns2中没有抓到任何报文：
```sh
root@S3:/home/tao# tcpdump -i macvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on macvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```

在ns3上抓包，可以看到对应的`arp`报文：
```sh
root@S3:/home/tao# tcpdump -i macvl3 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on macvl3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C09:52:10.476659 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:11.501269 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:12.525274 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28

3 packets captured
3 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```

在接口veth1_br上抓包，结合ns3中抓到的报文可以得出对应的报文已经发回父接口，但是被父接口丢弃！
```sh
tao@S3:~$ sudo tcpdump -i veth1_br 'icmp or arp' -e -n
[sudo] password for tao:
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth1_br, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:52:10.476612 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:10.476629 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:11.501235 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:11.501271 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:12.525247 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
09:52:12.525276 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.3 tell 192.168.9.2, length 28
^C
6 packets captured
6 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

手动在ns1和ns2中添加对方的mac地址信息后测试：
在ns1中：
```sh
root@S3:/home/tao# ip neigh add 192.168.9.3 lladdr 82:a3:b5:0f:fc:0c dev macvl1
RTNETLINK answers: File exists
root@S3:/home/tao# ip neigh
192.168.9.3 dev macvl1  FAILED
root@S3:/home/tao# ip neigh del 192.168.9.3
Device and destination are required arguments.
root@S3:/home/tao# ip neigh del 192.168.9.3 dev macvl1
root@S3:/home/tao# ip neigh add 192.168.9.3 lladdr 82:a3:b5:0f:fc:0c dev macvl1
root@S3:/home/tao# ip neigh
192.168.9.3 dev macvl1 lladdr 82:a3:b5:0f:fc:0c PERMANENT
root@S3:/home/tao#
```
在ns2中：
```sh
root@S3:/home/tao# ip neigh add 192.168.9.2 lladdr 9a:e4:69:98:8f:e3 dev macvl2
root@S3:/home/tao# ip neigh
192.168.9.2 dev macvl2 lladdr 9a:e4:69:98:8f:e3 PERMANENT
root@S3:/home/tao#
```
再次在ns1中测试，通了(`ping`了两次)！
> 这说明`macvlan`的`private`模式只是屏蔽了同一父接口下`macvlan`子接口之间的`arp`广播报文，要能相互通信须手动更新`arp`表！
```sh
root@S3:/home/tao# ping 192.168.9.3 -c1
PING 192.168.9.3 (192.168.9.3) 56(84) bytes of data.
64 bytes from 192.168.9.3: icmp_seq=1 ttl=64 time=0.131 ms

--- 192.168.9.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.131/0.131/0.131/0.000 ms
root@S3:/home/tao# ping 192.168.9.3 -c1
PING 192.168.9.3 (192.168.9.3) 56(84) bytes of data.
64 bytes from 192.168.9.3: icmp_seq=1 ttl=64 time=0.089 ms

--- 192.168.9.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.089/0.089/0.089/0.000 ms
root@S3:/home/tao#
```
在ns2中抓包，可以看到两次`ping`的报文交互：
```sh
root@S3:/home/tao# tcpdump -i macvl2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on macvl2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C10:48:36.044430 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo request, id 177, seq 1, length 64
10:48:36.044446 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo reply, id 177, seq 1, length 64
10:48:36.978716 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo request, id 53460, seq 1, length 64
10:48:36.978731 82:a3:b5:0f:fc:0c > 9a:e4:69:98:8f:e3, ethertype IPv4 (0x0800), length 98: 192.168.9.3 > 192.168.9.2: ICMP echo reply, id 53460, seq 1, length 64

4 packets captured
4 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```

在veth2上抓包，可以看到了一条`icmp`请求报文，这是`bridge`泛洪的报文，在`bridge`的转发表相应条目过期前只会收到这一次！
```bash
tao@S3:~$ sudo tcpdump -i veth2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on veth2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
10:48:36.044421 9a:e4:69:98:8f:e3 > 82:a3:b5:0f:fc:0c, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 192.168.9.3: ICMP echo request, id 177, seq 1, length 64
^C
1 packet captured
1 packet received by filter
0 packets dropped by kernel
tao@S3:~$
```

注意此时在ns3中并未抓到任何报文，因为虽然第一次`bridge`会泛洪，但此时是因为父接口并没有转发`icmp`请求报文到macvl3接口，报文并不是macvl3接口丢弃的！
> 正常情况下当网卡接收到一个数据帧时，它会检查该数据帧的目的MAC地址是否与其自身的MAC地址匹配。如果目的MAC地址不是网卡自身的MAC地址，并且网卡没有处于混杂模式（Promiscuous Mode），那么网卡会丢弃这个数据帧，不会将其传递给上层协议栈或任何网络分析工具（如tcpdump）。
```sh
root@S3:/home/tao# tcpdump -i macvl3 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on macvl3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
^C
0 packets captured
0 packets received by filter
0 packets dropped by kernel
root@S3:/home/tao#
```

### 验证和外部网络的连通性
在`bridge` test_br上配置ip：192.168.9.1，`bridge`配置ip后就具备了三层路由功能，但这并不影响其二层转发功能：
> 如果要和公网的交互，则还需要配置`iptables`的`nat`转发规则，可以参考[linux 虚拟网卡之 ipvlan](https://www.tao-wt.fun/linux/ipvlan/#%E6%B5%8B%E8%AF%95%E5%92%8C%E5%85%AC%E7%BD%91%E7%9A%84%E8%BF%9E%E9%80%9A%E6%80%A7 "ipvlan 公网连通性")

```bash
tao@S3:~$ sudo ip addr add 192.168.9.1/24 dev test_br
tao@S3:~$ ip addr show test_br
17: test_br: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 06:84:97:f4:94:01 brd ff:ff:ff:ff:ff:ff
    inet 192.168.9.1/24 scope global test_br
       valid_lft forever preferred_lft forever
    inet6 fe80::484:97ff:fef4:9401/64 scope link
       valid_lft forever preferred_lft forever
tao@S3:~$ ip r
default via 10.138.36.1 dev eno2 proto dhcp metric 100
10.138.36.0/23 dev eno2 proto kernel scope link src 10.138.36.58 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.19.0.0/16 dev br-9bf9aa158c15 proto kernel scope link src 172.19.0.1
192.168.9.0/24 dev test_br proto kernel scope link src 192.168.9.1
tao@S3:~$
```
在ns1中添加到test_br的默认路由 ，然后`ping`宿主机ip(36.58)和宿主机所在物理网络host(36.66)，通！
```sh
root@S3:/home/tao# ip r add default via 192.168.9.1 dev macvl1
root@S3:/home/tao# ip r
default via 192.168.9.1 dev macvl1
192.168.9.0/24 dev macvl1 proto kernel scope link src 192.168.9.2
root@S3:/home/tao#
root@S3:/home/tao# ping 10.138.36.58 -c 1
PING 10.138.36.58 (10.138.36.58) 56(84) bytes of data.
64 bytes from 10.138.36.58: icmp_seq=1 ttl=64 time=0.157 ms

--- 10.138.36.58 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.157/0.157/0.157/0.000 ms
root@S3:/home/tao# ping 10.138.36.66 -c 1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.
64 bytes from 10.138.36.66: icmp_seq=1 ttl=63 time=0.416 ms

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.416/0.416/0.416/0.000 ms
root@S3:/home/tao#
```
在`bridge`test_br上抓包：
```sh
tao@S3:~$ sudo tcpdump -i test_br 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on test_br, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:06:12.842385 9a:e4:69:98:8f:e3 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.1 tell 192.168.9.2, length 28
11:06:12.842402 06:84:97:f4:94:01 > 9a:e4:69:98:8f:e3, ethertype ARP (0x0806), length 42: Reply 192.168.9.1 is-at 06:84:97:f4:94:01, length 28
11:06:12.842441 9a:e4:69:98:8f:e3 > 06:84:97:f4:94:01, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 10.138.36.58: ICMP echo request, id 45436, seq 1, length 64
11:06:12.842461 06:84:97:f4:94:01 > 9a:e4:69:98:8f:e3, ethertype IPv4 (0x0800), length 98: 10.138.36.58 > 192.168.9.2: ICMP echo reply, id 45436, seq 1, length 64
11:06:16.536000 9a:e4:69:98:8f:e3 > 06:84:97:f4:94:01, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 10.138.36.66: ICMP echo request, id 32673, seq 1, length 64
11:06:16.536358 06:84:97:f4:94:01 > 9a:e4:69:98:8f:e3, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.2: ICMP echo reply, id 32673, seq 1, length 64
11:06:17.997264 06:84:97:f4:94:01 > 9a:e4:69:98:8f:e3, ethertype ARP (0x0806), length 42: Request who-has 192.168.9.2 tell 192.168.9.1, length 28
11:06:17.997339 9a:e4:69:98:8f:e3 > 06:84:97:f4:94:01, ethertype ARP (0x0806), length 42: Reply 192.168.9.2 is-at 9a:e4:69:98:8f:e3, length 28
^C
8 packets captured
8 packets received by filter
0 packets dropped by kernel
tao@S3:~$
tao@S3:~$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.9.2              ether   9a:e4:69:98:8f:e3   C                     test_br
172.19.0.5               ether   02:42:ac:13:00:05   C                     br-9bf9aa158c15
172.19.0.4               ether   02:42:ac:13:00:04   C                     br-9bf9aa158c15
10.138.36.1              ether   f4:e9:75:75:60:01   C                     eno2
10.138.36.66             ether   bb:6b:8c:86:6d:83   C                     eno2
tao@S3:~$
```
在eno2(36.58)上抓包，可以看到36.66相关的报文已经从eno2(36.58)接口转发出去！注意并没有和36.58相关的交互报文！
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:06:16.536037 bb:6b:8c:89:88:22 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 10.138.36.58, length 28
11:06:16.536200 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype ARP (0x0806), length 60: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 46
11:06:16.536219 bb:6b:8c:89:88:22 > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 192.168.9.2 > 10.138.36.66: ICMP echo request, id 32673, seq 1, length 64
11:06:16.536336 bb:6b:8c:86:6d:83 > bb:6b:8c:89:88:22, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 192.168.9.2: ICMP echo reply, id 32673, seq 1, length 64
^C
13 packets captured
13 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
36.66上到网络`192.168.9.0/24`路由配置如下:
```bash
tao@S20:~$ ip r
default via 10.138.36.1 dev eno1 proto dhcp metric 100
10.138.36.0/23 dev eno1 proto kernel scope link src 10.138.36.66 metric 100
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1
192.168.9.0/24 via 10.138.36.58 dev eno1
tao@S20:~$
```

## bridge
换种方式，直接在物理网卡eno2上创建两个`macvlan`子接口，子接口配置和父接口同一网络下的ip。

### 环境搭建
创建命名空间ns4和ns5，并将创建的两个`macvlan`接口分别放入两个命名空间：
```bash
tao@S3:~$ sudo ip netns add ns4
tao@S3:~$ sudo ip netns add ns5
tao@S3:~$ sudo ip link add link eno2 name macvl4 type macvlan mode bridge
tao@S3:~$ sudo ip link add link eno2 name macvl5 type macvlan mode bridge
tao@S3:~$ sudo ip link set macvl4 netns ns4
tao@S3:~$ sudo ip link set macvl5 netns ns5
tao@S3:~$
```
ns4的配置如下：
```sh
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl4 up
root@S3:/home/tao# ip addr add 10.138.36.2/23 dev macvl4
root@S3:/home/tao# ip r add default via 10.138.36.1 dev macvl4
root@S3:/home/tao# ip r
default via 10.138.36.1 dev macvl4
10.138.36.0/23 dev macvl4 proto kernel scope link src 10.138.36.2
root@S3:/home/tao# ip link show macvl4
33: macvl4@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether d6:2b:b6:7a:33:4f brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao#
```

ns5中配置：
```sh
root@S3:/home/tao# ip link set lo up
root@S3:/home/tao# ip link set macvl5 up
root@S3:/home/tao# ip addr add 10.138.36.3/23 dev macvl5
root@S3:/home/tao# ip r add default via 10.138.36.1 dev macvl5
root@S3:/home/tao# ip link show macvl5
34: macvl5@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 06:d9:4d:a5:a1:1e brd ff:ff:ff:ff:ff:ff link-netnsid 0
root@S3:/home/tao# ip r
default via 10.138.36.1 dev macvl5
10.138.36.0/23 dev macvl5 proto kernel scope link src 10.138.36.3
root@S3:/home/tao#
```
### 测试macvlan接口间的连通性
在ns4中`ping`ns5，通：
```sh
root@S3:/home/tao# ip neigh
10.138.36.3 dev macvl4 lladdr 06:d9:4d:a5:a1:1e STALE
root@S3:/home/tao# ip neigh del 10.138.36.3 dev macvl4
root@S3:/home/tao# ping 10.138.36.3 -c1
PING 10.138.36.3 (10.138.36.3) 56(84) bytes of data.
64 bytes from 10.138.36.3: icmp_seq=1 ttl=64 time=0.107 ms

--- 10.138.36.3 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.107/0.107/0.107/0.000 ms
root@S3:/home/tao#
```
在宿主机的父接口抓包如下，父接口相当于一个`bridge`转发其上`macvlan`接口之间的单播报文：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:02:25.119500 d6:2b:b6:7a:33:4f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.3 tell 10.138.36.2, length 28
16:02:25.119532 06:d9:4d:a5:a1:1e > d6:2b:b6:7a:33:4f, ethertype ARP (0x0806), length 42: Reply 10.138.36.3 is-at 06:d9:4d:a5:a1:1e, length 28
16:02:25.119541 d6:2b:b6:7a:33:4f > 06:d9:4d:a5:a1:1e, ethertype IPv4 (0x0800), length 98: 10.138.36.2 > 10.138.36.3: ICMP echo request, id 24286, seq 1, length 64
16:02:25.119558 06:d9:4d:a5:a1:1e > d6:2b:b6:7a:33:4f, ethertype IPv4 (0x0800), length 98: 10.138.36.3 > 10.138.36.2: ICMP echo reply, id 24286, seq 1, length 64
^C
11 packets captured
11 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```
在和宿主机同一物理网络的其它主机(36.66)抓包，抓到了`arp`广播报文，也说明`macvlan`接口和父接口共享同一广播域！
```bash
tao@S20:~$ sudo tcpdump -i eno1 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eno1, link-type EN10MB (Ethernet), capture size 262144 bytes
16:02:25.116210 d6:2b:b6:7a:33:4f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.3 tell 10.138.36.2, length 46
^C
1 packet captured
2 packets received by filter
0 packets dropped by kernel
tao@S20:~$
```

### 测试和外部网络的连通性
在ns4中`ping`和父接口同一物理网络的其它主机36.66以及其它网络的主机10.138.10.161，通：
```sh
root@S3:/home/tao# ping 10.138.36.66 -c1
PING 10.138.36.66 (10.138.36.66) 56(84) bytes of data.
64 bytes from 10.138.36.66: icmp_seq=1 ttl=64 time=0.335 ms

--- 10.138.36.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.335/0.335/0.335/0.000 ms
root@S3:/home/tao# ping 10.138.10.161 -c1
PING 10.138.10.161 (10.138.10.161) 56(84) bytes of data.
64 bytes from 10.138.10.161: icmp_seq=1 ttl=63 time=1.63 ms

--- 10.138.10.161 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.628/1.628/1.628/0.000 ms
root@S3:/home/tao#
```
在父接口上抓包如下，可以看到`ping`不同子网ip时，流量是经过网关36.1的：
```bash
tao@S3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:14:45.336947 d6:2b:b6:7a:33:4f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.66 tell 10.138.36.2, length 28
16:14:45.337109 bb:6b:8c:86:6d:83 > d6:2b:b6:7a:33:4f, ethertype ARP (0x0806), length 60: Reply 10.138.36.66 is-at bb:6b:8c:86:6d:83, length 46
16:14:45.337134 d6:2b:b6:7a:33:4f > bb:6b:8c:86:6d:83, ethertype IPv4 (0x0800), length 98: 10.138.36.2 > 10.138.36.66: ICMP echo request, id 11377, seq 1, length 64
16:14:45.337247 bb:6b:8c:86:6d:83 > d6:2b:b6:7a:33:4f, ethertype IPv4 (0x0800), length 98: 10.138.36.66 > 10.138.36.2: ICMP echo reply, id 11377, seq 1, length 64
16:14:50.445419 bb:6b:8c:86:6d:83 > d6:2b:b6:7a:33:4f, ethertype ARP (0x0806), length 60: Request who-has 10.138.36.2 tell 10.138.36.66, length 46
16:14:50.445435 d6:2b:b6:7a:33:4f > bb:6b:8c:86:6d:83, ethertype ARP (0x0806), length 42: Reply 10.138.36.2 is-at d6:2b:b6:7a:33:4f, length 28
16:14:51.958764 d6:2b:b6:7a:33:4f > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 10.138.36.1 tell 10.138.36.2, length 28
16:14:51.960122 f4:e9:75:75:60:01 > d6:2b:b6:7a:33:4f, ethertype ARP (0x0806), length 60: Reply 10.138.36.1 is-at f4:e9:75:75:60:01, length 46
16:14:51.960141 d6:2b:b6:7a:33:4f > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 10.138.36.2 > 10.138.10.161: ICMP echo request, id 57533, seq 1, length 64
16:14:51.960360 f4:e9:75:75:60:01 > d6:2b:b6:7a:33:4f, ethertype IPv4 (0x0800), length 98: 10.138.10.161 > 10.138.36.2: ICMP echo reply, id 57533, seq 1, length 64
^C
23 packets captured
24 packets received by filter
0 packets dropped by kernel
tao@S3:~$
```

### 测试与公网的连通性
当然`ping 110.242.68.66`公网也是通的，因为报文也是通过网关36.1转发出去了(默认路由)。**这种情况，因为创建的`macvlan`接口和父接口是同一网络，所以省去了额外在防火墙`iptables`中配置 `NAT`规则的麻烦。**
```sh
root@S3:/home/tao# ping 110.242.68.66 -c1
PING 110.242.68.66 (110.242.68.66) 56(84) bytes of data.
64 bytes from 110.242.68.66: icmp_seq=1 ttl=48 time=30.8 ms

--- 110.242.68.66 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 30.768/30.768/30.768/0.000 ms
root@S3:/home/tao#
```
父接口上抓包：
```bash
zeekr@SH-E3:~$ sudo tcpdump -i eno2 'icmp or arp' -e -n
[sudo] password for zeekr:
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eno2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
16:32:26.458315 d6:2b:b6:7a:33:4f > f4:e9:75:75:60:01, ethertype IPv4 (0x0800), length 98: 10.138.36.2 > 110.242.68.66: ICMP echo request, id 57597, seq 1, length 64
16:32:26.489057 f4:e9:75:75:60:01 > d6:2b:b6:7a:33:4f, ethertype IPv4 (0x0800), length 98: 110.242.68.66 > 10.138.36.2: ICMP echo reply, id 57597, seq 1, length 64
2 packets captured
2 packets received by filter
0 packets dropped by kernel
zeekr@SH-E3:~$
```
