---
title: 根据MAC地址获取IP地址
date: 2023-12-20 19:07:52
index_img: /img/index-0.jpg
tags:
  - linux
  - network
  - python
  - DHCP
categories:
  - [network, DHCP]
author: Tao
excerpt: 本文介绍如何利用DHCP协议来获取MAC地址对应的IP地址
---
### 背景简介
---
局域网中，通过MAC地址获取其对应的IP地址，可以通过`arp`协议，但是如果两台设备之间无交互，则本地arp表中不会存储有对应的IP条目
```bash
tao@z20:~$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
10.238.36.1              ether   54:00:75:75:60:01   C                     eno2
10.238.36.6              ether   e0:ae:6b:fc:5c:4d   C                     eno2
```

### DHCP协议
---
DHCP协议是从BOOTP（Bootstrap Protocol）协议发展而来的，主要是实现IP的自动获取和配置。DHCP协议报文采用UDP协议封装，因为DHCP通常用在局域网，网络环境可靠，UDP降低了通信的复杂性和开销，DHCP服务器侦听67端口，客户端是68端口。DHCP协议的交互流程如下：
- DHCP客户端初次接入网络时,会发送DHCP discover广播报文（DHCP Discover），用于查找和定位DHCP服务器，以获取IP。报文的源IP为`0.0.0.0`, MAC为本机mac地址；目的IP为：`255.255.255.255`，MAC为：`FF:FF:FF:FF:FF:FF`。
- DHCP server在收到DHCP discover报文后，会发送DHCP offer报文（DHCP Offer），此报文包含分配的IP地址、掩码、网关、DNS、IP过期时间等信息等配置信息。discover报文和offer报文通过事务ID：`XID`字段关联。另外DHCP协议的第一个字段`op`也可以用来区分是应答报文还是请求报文
- 在DHCP客户端收到服务器发送的DHCP offer报文后，会发送DHCP request报文（DHCP Request）另外在DHCP客户端获取IP地址并重启后，同样也会发送DHCP请求报文，用于确认分配的IP地址等配置信息。DHCP客户端的IP地址快要到期时，也会发送DHCP request报文向服务器申请延长IP地址租期。
- 收到DHCP客户端发送的DHCP请求报文后，DHCP服务器会回复DHCP ACK报文（DHCP ACK），表示IP分配成功。
- 客户端收到DHCP ACK报文后，会将获取的IP地址等信息进行配置和使用。

### 通过python的scapy模块发送DHCP报文来获取IP
这部分介绍如何通过python来实现自动获取ip，以及相关函数的实现

#### 什么是Scapy？
scapy官方的定义：
> Scapy is a Python program that enables the user to send, sniff and dissect and forge network packets. This capability allows construction of tools that can probe, scan or attack networks.
> In other words, Scapy is a powerful interactive packet manipulation program. It is able to forge or decode packets of a wide number of protocols, send them on the wire, capture them, match requests and replies, and much more. Scapy can easily handle most classical tasks like scanning, tracerouting, probing, unit tests, attacks or network discovery. It can replace hping, arpspoof, arp-sk, arping, p0f and even some parts of Nmap, tcpdump, and tshark.

#### python function实现
第一步，导入scapy模块
```python
from scapy.all import *
```

第二步，自动获取网络接口，`dev_from_index`是scapy中的函数(Return interface for a given interface index)
```python
def get_net_iface():
    iface_index = ''
    for line in conf.ifaces.show(print_result=False).split('\n'):
        if '192.168.14' in line:
            print(f"iface found: {line}.")
            iface_index = line.split()[2]
            break
    else:
        raise Exception("Is the server connect to correct network?")
    return dev_from_index(iface_index)
```

第三步，从第二部获取的网络接口发送DHCP报文, 其中：
- `bytes.fromhex`将十六进制字符串转换为`bytes`字节字符串，也可以通过`biascii.unhexkify`来实现
- `conf.checkIPaddr`设置为`False`，可以避免请求的目的地址和应答的源地址不匹配，获取不到应答
- `flags=0x8000`flags字段的最高位置1，使DHCP server也以广播形式应答

```python
def get_ip_from_mac(mac, iface):
    bmac = bytes.fromhex(mac.replace(":", ""))
    conf.checkIPaddr = False

    dhcp_discover = Ether(dst="ff:ff:ff:ff:ff:ff", src=iface.mac) \
        / IP(src="0.0.0.0", dst="255.255.255.255") \
        / UDP(sport=68, dport=67) \
        / BOOTP(
            chaddr=bmac,
            xid=random.randint(1, 1000000000),
            flags=0x8000) \
        / DHCP(options=[("message-type", "discover"), "end"])
    dhcp_offer = srp1(dhcp_discover, iface=iface, timeout=9)
    if dhcp_offer:
        ip_address = dhcp_offer[BOOTP].yiaddr
        print(f"mac {mac} ip is: {ip_address}")
        return ip_address
    else:
        raise Exception(f"No DHCP offer received for {mac}.")
```

在ubuntu22上以**root权限**运行测试如下：
```python
>>> iface = get_net_iface()
iface found: sys     2      eno2     ff:6f:8c:f9:88:22  10.38.36.58  fe80::3304:c53b:493e:2f97.
>>> iface
<NetworkInterface eno2 [UP+BROADCAST+RUNNING+SLAVE]>
>>> get_ip_from_mac("ff:6b:fc:86:f0:de", iface)
Begin emission:
Finished sending 1 packets.
.........................................................................................................................................................................................*
Received 186 packets, got 1 answers, remaining 0 packets
mac ff:6b:fc:86:f0:de ip is: 10.38.36.148
'10.38.36.148'
>>>
```

### 注意事项
这种获取IP的方式有以下局限：
- 只在同一局域网内有效，不同网络虽然能获取到ip但并不可用
- 如果指定的mac对应的主机关机，则获取的ip并不一定可用
