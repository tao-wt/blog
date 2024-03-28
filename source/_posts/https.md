---
title: HTTPS的证书校验和数据加密过程
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
[2024-03-22T07:22:00.550Z]       |   File "C:\Users\tao\jenkins\venv\Lib\site-packages\aiohttp\connector.py", line 994, in _wrap_create_connection
[2024-03-22T07:22:00.550Z]       |     raise ClientConnectorCertificateError(req.connection_key, exc) from exc
[2024-03-22T07:22:00.550Z]       | aiohttp.client_exceptions.ClientConnectorCertificateError: Cannot connect to host oss.obsv3.mo-jkr-1.tao.com:443 ssl:True [SSLCertVerificationError: (1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1006)')]
[2024-03-22T07:22:00.550Z]       +------------------------------------

```
从报错看是因为不能获取本地的CA证书导致的。查看[官方文档](https://docs.aiohttp.org/en/stable/client_advanced.html#example-verify-certificate-fingerprint "aiohttp")有这样的描述：
> By default, Python uses the system CA certificates. In rare cases, these may not be installed or Python is unable to find them, resulting in a error like ssl.SSLCertVerificationError: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate

### 解决办法：
1. 禁用SSL证书校验，官方说明如下：By default aiohttp uses strict checks for HTTPS protocol. Certification checks can be relaxed by setting `ssl` to `False`:
    ```python
    r = await session.get('https://example.com', ssl=False)
    ```
    或者在session层面禁用ssl校验:
    ```python
     ClientSession(connector=TCPConnector(ssl=False))
    ```
3. to work around this problem is to use the **certifi** package:
    ```python
    ssl_context = ssl.create_default_context(cafile=certifi.where())
    async with ClientSession(connector=TCPConnector(ssl=ssl_context)) as sess:
    ...
    ```
    > **Certifi** provides Mozilla’s carefully curated精心策划的 collection of Root Certificates for validating the trustworthiness可信度 of SSL certificates while verifying the identity身份 of TLS hosts. It has been extracted提取 from the `Requests` project.

由于解决上面上面的错误时，勾起了对https加密过程的好奇，所以就有了下面https(TLSv1.2)的加密过程的分析。

## https请求整体流程
下面截图是访问Bing时的报文交互过程，后面对每条报文做单独分析
![https0](/img/https0.png)

## Client Hello
`TCP`连接建立后，客户端首先向服务器发送一个`Client Hello`消息，其中包含SSL协议版本, 随机数`Random`和客户端支持的加密算法列表`Cipher Suites`, 如下图：
![https1](/img/https1.png)
各字段含义如下：
- `Random`：随机数，在生成`对称密钥`时会用
    > `Symmetric Encryption`: `对称加密`, 一种早期的加密技术: 在加密和解密过程中使用**同一密钥**。当发送者发送加密消息时，使用密钥来**加密**原始信息。接收者在收到加密消息后，须用同一密钥**解密**信息。对称加密因其高效性和易于实施而受到广泛的欢迎，但安全性相对较低，因为任何知道这个密钥的人都能解密被加密的信息。此外，密钥的管理和分发过程也存在潜在的危险。
    > `Asymmetric Encryption`: `非对称加密`,又称公钥密码系统，它使用**一对密钥**，包括一个公钥和一个私钥。公钥用于加密信息，而私钥则用于解密信息。公钥可以公开给任何人，而私钥则必须妥善保管。拥有私钥的人才能解密用公钥加密的信息。这种加密方式的安全性依赖于某种计算的复杂性，因此安全性较高, 但速度相对较慢。
- `Cipher Suites`：服务端会从此列表选择其支持的密码套件


## Server Hello 和 Certificate
Server Hello的报文结构如下：
![https2](/img/https2.png)
各字段含义如下：
- `Random`串：服务端生成的随机数，在生成对称密钥时会用
- `Cipher Suites`：服务端选择的密码套件，`TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
    - `TLS`：`Transport Layer Security`, 指使用的协议是TLS
    - `ECDHE`：表示椭圆曲线Diffie-Hellman密钥交换（Elliptic Curve Diffie-Hellman Ephemeral）算法，它允许双方在不共享任何秘密的情况下协商出一个共享的对称密钥。Ephemeral: 短暂的。
    - `RSA`：一种`非对称加密`算法，表示使用RSA算法进行身份验证和签名。在TLS握手过程中，服务器会提供一个RSA公钥证书，客户端验证这个证书以确认服务器的身份。
    - `AES_256_GCM`：这是对称加密算法的组合。AES（Advanced Encryption Standard）指对称加密算法，GCM（Galois/Counter Mode）AES的一种工作模式，它提供了数据认证和加密的功能。这里的“256”指的是AES使用的密钥长度，256位通常被认为是相当安全的。
    - `SHA384`：消息认证码（MAC）算法，用于确保数据的完整性和真实性。

- `Certificate`：服务端的证书，Bing有两个证书。
    这个阶段，服务器会发送自己的证书给客户端，以便客户端验证服务器的身份。这个证书可能是由一个根证书颁发机构（CA）直接签发的，也可能是由一个或多个中间CA签发的。

    可以导出上面两个证书到`.der`文件(右击，*导出分组字节流*)，查看证书信息(windows可以直接查看，linux需要先转成`.pem`文件)：
    ![https3](/img/https3.png)
    ![https4](/img/https4.png)

> `PDU`: 协议数据单元，是网络中不同层之间传递的信息单元。
> `TCP segment of a reassembled PDU`：意味着这条TCP数据包是一个重新组装的数据包单元（PDU）的一部分。`TCP`是一种面向流的协议，它并不保证每次发送或接收的数据包都严格对应应用层发送或接收的消息。数据包在传输过程中可能会被分割成多个较小的片段，多个较小的数据包也可能会在接收端被重新组合(reassembled)成一个较大的数据流。

### 服务器证书
#### 证书申请
申请证书时，申请者（通常是服务器管理员）使用某种加密算法（如`RSA`）生成一个密钥对，这个密钥对包括一个公钥和一个私钥。公钥将被放入SSL证书中，而私钥则必须安全地保存在服务器上。 

然后，申请者会提交一个证书签名请求`CSR`给CA机构审核，这个`CSR`包含了申请者的信息以及公钥。如果审核通过，CA机构会使用**自己的私钥**对申请者的公钥和相关信息进行签名，生成**SSL证书**。申请者随后将这个证书安装在服务器上，用于与浏览器或其他客户端进行安全通信。

#### 证书验证
验证过程如下：
1. 浏览器会首先检查这个证书是否由它所信任的CA颁发。它会查看内置的受信任CA列表，找到对应的CA公钥。
2. 然后浏览器用找到的CA公钥来解密证书中的某些部分（如签名），并验证其有效性。
    > **私钥加密的信息不能用公钥直接解密**。私钥加密通常用于`数字签名`：发送者使用自己的私钥对消息摘要（通常是消息的哈希值）进行加密，生成`数字签名`。
    >
    > 接收者收到消息和数字签名后，使用发送者的公钥对数字签名进行解密，然后重新计算消息的哈希值，并与解密得到的哈希值进行比较。如果两者相同，那么接收者就可以确认消息是由持有相应私钥者签发，并且消息在传输过程中没有被篡改。
    >
    > 这个过程并不涉及公钥对私钥加密信息的解密，而是**公钥对数字签名的验证**。**私钥加密的信息只能用私钥解密**。非对称加密。

3. 如果使用CA公钥解密和验证的过程成功，浏览器就会信任这个服务器证书，并认为与该服务器的通信是安全的。
4. 浏览器会检查证书吊销列表，确保该证书没有被证书颁发机构吊销。
5. 浏览器会验证SSL证书的有效期，确保证书没有过期。
6. 浏览器会检查SSL证书中的域名是否与正在访问的网站的域名一致。
7. 浏览器会验证SSL证书中的公钥，确保它们可以安全地将数据传输给服务器。
    可以用`openssl`命令来手动提取服务器公钥
    ```bash
    openssl x509 -in certificate.pem -pubkey -noout > ca_pubkey.pem
    ```

#### 查看系统已安装的CA
查看windows系统已安装的证书：`certmgr.msc`
![https5](/img/https5.png)
Linux系统已安装的证书在`/etc/ssl/certs`目录下，可以用`update-ca-certificates`来更新证书：
![https6](/img/https6.png)

> `CA证书`（Certificate Authority Certificate）: 是由受信任的证书颁发机构（CA）签发的证书，用于证明该CA的**身份**和**公钥**。`CA证书`是公钥基础设施（PKI）的核心组成部分，它允许其他实体（如服务器或客户端）验证由该CA签发的其他证书的有效性。

## Client Finish
![https7](/img/https7.png)
**Client Key Exchange**：发送`预主密钥`（Pre-Master Secret）给服务器(用服务器公钥加密，确保只有服务器知道预主密钥)。`预主密钥`是一个随机生成的密钥，用于和服务器共同计算出`会话密钥`（Session Key, 对称的）,用于后续的加密通信

**Change Cipher Spec**：一个简单的通知，告诉通信的另一方接下来的数据将使用新的加密算法和会话密钥（使用两个随机数以及第三个`Pre-master key/secret`随机数一起算出的`对称密钥` session key/secret）进行加密

**encrypted handshake message**：包含了之前握手过程中所有重要信息的加密版本，这是为了在正式传输数据之前对刚握手建立起来的加解密通道进行验证

## Server Finish
服务端对客户端发送过来的报文使用服务端私钥进行解密校验,提取出`预主密钥`，并生成相同的`对称密钥`
![https8](/img/https8.png)
**New Session Ticket**: 服务器发送一个新的会话票据`Session Ticket`给客户端，以便客户端可以在将来的连接中重用会话状态，从而避免完整的握手过程。
> 客户端利用New Session Ticket恢复加密连接时，`会话密钥`通常不会变化。

**Change Cipher Spec**: 用于告知通信的另一方，随后的报文将使用之前协商好的加密规范（Cipher Spec）进行加密, 标志着加密通信的开始

**Encrypted Handshake Message**: 目的是验证之前协商的加密参数（如密钥）的正确性。如果双方能够成功解密并验证这个报文，那说明握手过程中协商的加密参数是正确的。

## 解密https报文
经过上面的分析，要解密https报文得先获得`预主密钥`, 但是有抓取的`Client Key Exchange`报文并不能获取`预主密钥`, 因为用公钥加密的`预主密钥`只能用对应的私钥来解密; 所以要解密https报文只能在客户端(如，浏览器)生成`预主密钥`时将其保存，然后再用协商好的信息生成`会话密钥`......

### Wireshark 设置
新版的Wirshark提供了更方便的方法来解密https报文, 配置步骤如下：
![wireshark_tls1](/img/wireshark_tls1.png)
![wireshark_tls2](/img/wireshark_tls2.png)
看，现在抓取到的报文已经自动解密了，可以看到必应用的竟是**HTTP2**：
![wireshark_tls3](/img/wireshark_tls3.png)
