---
title: https的校验和加密过程
date: 2024-3-23 20:20:52
index_img: /img/index-8.jpg
tags:
  - HTTP
  - network
  - tls
categories:
  - [network, HTTP, tls]
author: tao-wt@qq.com
excerpt: 通过分析wireshark抓取的报文，来理解https的校验和加密过程
---
## 起因
最近在用python的aiohttp模块发送https请求时遇到了`ClientConnectorCertificateError`错误：
```
[2024-03-22T07:22:00.550Z]       |                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[2024-03-22T07:22:00.550Z]       |   File "C:\Users\tao\jenkins\venv\Lib\site-packages\aiohttp\connector.py", line 1235, in _create_direct_connection
[2024-03-22T07:22:00.550Z]       |     raise last_exc
[2024-03-22T07:22:00.550Z]       |   File "C:\Users\tao\jenkins\venv\Lib\site-packages\aiohttp\connector.py", line 1204, in _create_direct_connection
[2024-03-22T07:22:00.550Z]       |     transp, proto = await self._wrap_create_connection(
[2024-03-22T07:22:00.550Z]       |                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
[2024-03-22T07:22:00.550Z]       |   File "C:\Users\zetaoekr\jenkins\venv\Lib\site-packages\aiohttp\connector.py", line 994, in _wrap_create_connection
[2024-03-22T07:22:00.550Z]       |     raise ClientConnectorCertificateError(req.connection_key, exc) from exc
[2024-03-22T07:22:00.550Z]       | aiohttp.client_exceptions.ClientConnectorCertificateError: Cannot connect to host oss.obsv3.mo-jkr-1.tao.com:443 ssl:True [SSLCertVerificationError: (1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1006)')]
[2024-03-22T07:22:00.550Z]       +------------------------------------

```
从报错看是因为不能获取本地的CA证书导致的。查看[官方文档](https://docs.aiohttp.org/en/stable/client_advanced.html#example-verify-certificate-fingerprint "aiohttp")有这样的描述：
> By default, Python uses the system CA certificates. In rare cases, these may not be installed or Python is unable to find them, resulting in a error like ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate

解决办法：
1. 禁用SSL证书校验，官方说明如下：By default aiohttp uses strict checks for HTTPS protocol. Certification checks can be relaxed by setting ssl to False: `r = await session.get('https://example.com', ssl=False)` 或者在session层面禁用ssl校验`ClientSession(connector=TCPConnector(ssl=False))`
2. to work around this problem is to use the certifi package:
    ```python
    ssl_context = ssl.create_default_context(cafile=certifi.where())
    async with ClientSession(connector=TCPConnector(ssl=ssl_context)) as sess:
    ...
    ```
    > **Certifi** provides Mozilla’s carefully curated collection of Root Certificates for validating the trustworthiness可信度 of SSL certificates while verifying the identity身份 of TLS hosts. It has been extracted提取 from the `Requests` project.

由于解决上面上面的错误时，勾起了对https加密过程的好奇，所以让我们开始一步一步剖析https的加密过程。

## https请求整体流程
下面截图是访问Bing时抓取的报文，后面对每条报文做单独分析
![https0](/img/https0.png)

## Client Hello
`TCP`连接建立后，客户端首先向服务器发送一个`Client Hello`消息，其中包含SSL协议版本, 随机字符串`Random`和客户端支持的加密算法列表`Cipher Suites`, 如下图：
![https1](/img/https1.png)
各字段含义如下：
- `Random`：随机数，在生成对称密钥时会用
    > `Symmetric Encryption`: `对称加密`, 一种早期的加密技术: 在加密和解密过程中使用**同一密钥**。当发送者需要发送加密消息时，他们使用密钥来**加密**原始信息。接收者在收到加密消息后，须用同一密钥**解密**信息，获取原始的明文内容。对称加密因其高效性和易于实施而受到广泛的欢迎，但安全性相对较低，因为任何知道这个密钥的人都能解密被加密的信息。此外，密钥的管理和分发过程也存在潜在的危险。
    > `Asymmetric Encryption`: `非对称加密`,又称公钥密码系统，它使用**一对密钥**，包括一个公钥和一个私钥。公钥用于加密信息，而私钥则用于解密信息。公钥可以公开给任何人，而私钥则必须妥善保管。只有拥有相应私钥的人才能解密用公钥加密的信息。这种加密方式的安全性依赖于某种计算的复杂性，因此安全性较高, 但速度相对较慢。
- `Cipher Suites`：服务端会从此列表选择其支持的密码套件


## Server Hello 和 Certificate
> `PDU`: 协议数据单元，是网络中不同层之间传递的信息单元。
> `TCP segment of a reassembled PDU`：意味着这条TCP数据包是一个重新组装的数据包单元（PDU）的一部分。`TCP`是一种面向流的协议，它并不保证每次发送或接收的数据包都严格对应应用层发送或接收的消息。数据包在传输过程中可能会被分割成多个较小的片段，多个较小的数据包也可能会在接收端被重新组合(reassembled)成一个较大的数据流。
Server Hello的报文结构如下：
![https2](/img/https2.png)
各字段含义如下：
- `Random`串：服务端生成的随机数，在生成对称密钥时会用
- `Cipher Suites`：服务端选择的加密密码套件，```Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)```
    > `TLS`：指使用的协议是TLS
    > `ECDHE`：密钥交换算法, ECDHE（Elliptic Curve Diffie-Hellman Ephemeral）是基于椭圆曲线密码学的Diffie-Hellman密钥交换协议的变体。它允许双方在不共享任何秘密的情况下协商出一个共享的密钥。Ephemeral: 短暂的。使用`Diffie Hellman`算法进行TLS密钥交换具有优势：客户端和服务器都为每个新会话生成一个新密钥对。一旦计算出`预主密钥`，将立即删除客户端和服务器的私钥。这意味着私钥永远不会被窃取，确保完美的前向保密。
    > `RSA`：一种非对称加密算法，用于在TLS握手阶段交换密钥和其他参数。RSA在这里被用来进行服务器的身份验证。
    > `AES_256_GCM`：这是对称加密算法的组合。AES（Advanced Encryption Standard）指对称加密算法，GCM（Galois/Counter Mode）AES的一种工作模式，它提供了数据认证和加密的功能。这里的“256”指的是AES使用的密钥长度，256位通常被认为是相当安全的。
    > `SHA384`：这是一个消息认证码（MAC）算法，用于确保数据的完整性和真实性。
- `Certificate`：服务端证书，包含两个证书
    > 返回多个证书，这通常是因为服务器返回的证书链中包含多个证书。在HTTPS通信中，服务器会发送自己的证书给客户端，以便客户端验证服务器的身份。这个证书可能是由一个根证书颁发机构（CA）直接签发的，也可能是由一个或多个中间CA签发的。在这种情况下，服务器会发送一个证书链，包括服务器的证书以及所有必要的中间证书，最终可以追溯到根证书。
    > 分别导出上面两个证书到`.der`文件，进行查看：
    > ![https3](/img/https3.png)
    > ![https4](/img/https4.png)

### 服务器证书
#### 证书申请
申请证书时，申请者（通常是服务器管理员）使用某种加密算法（如`RSA`）生成一个密钥对，这个密钥对包括一个公钥和一个私钥。公钥将被放入SSL证书中，而私钥则必须安全地保存在服务器上。 

然后，申请者会创建一个证书签名请求`CSR`，这个`CSR`包含了申请者的信息以及公钥。这个`CSR`会提交给CA机构进行审核。如果审核通过，CA机构会使用自己的私钥对申请者的公钥和相关信息进行签名，生成SSL证书。这个证书会返回给申请者，申请者随后将其安装在服务器上，用于与浏览器或其他客户端进行安全通信。

#### 证书验证
验证过程如下：
1. 浏览器会首先检查这个证书是否由它所信任的CA颁发。为了进行这个检查，浏览器会查看其内置的受信任CA列表，找到对应的CA公钥。
2. 浏览器使用找到的CA公钥来解密证书中的某些部分（如签名），并验证其有效性。这个过程可以确保证书在传输过程中没有被篡改，且确实是由该CA签发的。
3. 如果使用CA公钥解密和验证的过程成功，浏览器就会信任这个服务器证书，并认为与该服务器的通信是安全的。
4. 浏览器会检查证书吊销列表，确保该证书没有被证书颁发机构吊销。
5. 浏览器会验证SSL证书的有效期，确保证书没有过期。
6. 浏览器会检查SSL证书中的域名是否与正在访问的网站的域名一致。
7. 浏览器会验证SSL证书中的公钥，确保它们可以安全地将数据传输给服务器。
    可以用下面`openssl`命令来手动提取服务器公钥
    ```openssl x509 -in certificate.pem -pubkey -noout > ca_pubkey.pem```

#### 查看系统已安装的CA
查看windows系统已安装的证书：`certmgr.msc`
![https5](/img/https5.png)
Linux系统已安装的证书在`/etc/ssl/certs`目录下，可以用`update-ca-certificates`来更新证书：
![https6](/img/https6.png)

> `CA证书`（Certificate Authority Certificate）是由受信任的证书颁发机构（CA）签发的证书，用于证明该CA的**身份**和**公钥**。`CA证书`是公钥基础设施（PKI）的核心组成部分，它允许其他实体（如服务器或客户端）验证由该CA签发的其他证书的有效性。
> 服务器的`SSL证书`（通常称为服务器证书或TLS证书）是由CA签发给特定服务器的证书。它包含了服务器的公钥信息，以及关于服务器身份的其他数据（如域名、组织信息等）。
> 当客户端（如浏览器）与服务器建立HTTPS连接时，服务器会将其`SSL证书`发送给客户端。客户端使用其信任的`CA证书`来验证服务器证书的有效性，从而确保与服务器之间的通信是加密和安全的。

## Client Key Exchange、Change Cipher Spec、Encrypted Handshake Message
![https7](/img/https7.png)
`Client Key Exchange`：用于发送`预主密钥`（Pre-Master Secret）给服务器(使用服务器公钥加密，这样确保了只有服务器能够知道预主密钥)。`预主密钥`是一个随机生成的密钥，用于和服务器共同计算出`会话密钥`（Session Key, 对称的）,用于后续的加密通信
`Client Cipher Spec`：一个简单的通知，告诉通信的另一方接下来的数据将使用新的加密算法和会话密钥（使用两个随机数以及第三个`Pre-master key/secret`随机数一起算出的`对称密钥` session key/secret）进行加密
`encrypted handshake message`：包含了之前握手过程中所有重要信息的加密版本，此报文是为了在正式传输数据之前对刚刚握手建立起来的加解密通道进行验证

## New Session Ticket、Change Cipher Spec、Encrypted Handshake Message
服务端对客户端发送过来的报文使用服务端私钥进行解密校验,提取出`预主密钥`，并生成相同的`对称密钥`
![https8](/img/https8.png)
`New Session Ticket`: 服务器发送一个新的会话票据`Session Ticket`给客户端，以便客户端可以在将来的连接中重用会话状态(主密钥、证书信息等)，从而避免完整的握手过程。
`Change Cipher Spec`: 用于告知通信的另一方，随后的报文将使用之前协商好的加密规范（Cipher Spec）进行加密, 标志着加密通信的开始
`Encrypted Handshake Message`: 主要目的是验证之前协商的加密参数（如密钥）的正确性。如果双方能够成功解密并验证这个报文，那么说明握手过程中协商的加密参数是正确的。
