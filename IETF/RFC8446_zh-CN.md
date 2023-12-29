# The Transport Layer Security (TLS) Protocol Version 1.3

> 原文 [https://datatracker.ietf.org/doc/html/rfc8446](https://datatracker.ietf.org/doc/html/rfc8446)

## 摘要

本文档指定了传输层安全性（TLS）协议的 1.3 版本。TLS 允许客户端/服务器应用程序以防止窃听、篡改和消息伪造的方式通过 Internet 进行通信。

本文档更新了 [RFC5705] 和 [RFC6066]，并废弃了 [RFC5077]、[RFC5246] 和 [RFC6961]。本文档还规定了 TLS 1.2 实现的新要求。

## 1. Introduction

TLS 的主要目标是为通信双方提供一个安全的通道；对下层传输的唯一要求是提供可靠保序的数据流。安全通道尤其应该提供如下属性：

- 认证（Authentication）：server 端应该总是需要认证的；client 端可以选择性的认证。认证可以通过非对称算法完成（如RSA、椭圆曲线数字签名算法(ECDSA)、或 Edwards曲线数字签名算法(EdDSA)）完成，或通过对称预共享密钥（PSK）。
- 保密（Confidentiality）：在建立好的通道上发送的数据只能终端可见。TLS不隐藏传输数据的长度，但终端可以填充TLS记录（pad）来隐藏真实长度，从而提升安全性。
- 完整性（Integrity）：在建立好的通道上发送的数据不能被攻击者修改（如果修改了一定会被发现）。

即使出现像 [RFC3552] 那样已经完全控制网络的攻击者，这些属性也应该保证。更完整的相关安全属性清单详见附录E。

TLS由两个主要部分构成：

- 握手协议（第4节），认证通信双方，协商加密模式和参数，并确定共享密钥。握手协议可以防止篡改，如果连接没有受到攻击，攻击者就不能强制两端协商不同的参数。
- 记录协议（第5节），使用由握手协议协商出的参数来保护通信双方的流量。记录协议将流量分为一系列记录，每个记录独立地使用流量密钥保护。

TLS是一个独立的应用协议；上层协议可以透明地运行于 TLS 之上。然而，TLS 标准并未指定怎样使用 TLS 保证安全、怎样发起 TLS 握手以及怎样理解认证证书交换，这些留给运行在 TLS 之上的协议的设计者和实现者来判断。

本文定义了 TLS 1.3. 虽然 TLS 1.3 与以前的版本不直接兼容，但所有 TLS 版本都包含一个版本控制机制，可以在客户端和服务器都支持的情况下协商出一个公共版本。

本文取代和废除了以前版本的 TLS，包括 1.2 版本 [RFC5246]。也废除了在 [RFC5077] 里面定义的 TLS ticket 机制，并用定义在第 2.2 节中的机制取代它。由于 TLS 1.3 改变了密钥的产生方式，它更新了 [RFC5705]。正如 7.5 节描述的那样。它也改变了在线证书状态协议（OCSP）消息的传输方式，因此更新了 [RFC6066]，废除了 [RFC6961]，如第 4.4.2.1 节所述。

### 1.1. Conventions and Terminology

使用了以下术语:

- 客户端（client）: 发起 TLS 连接的端点。
- 连接（connection）: 两端之间的传输层连接。
- 端点（endpoint）: 连接的客户端或者服务器。
- 握手（handshake）: 客户端和服务器之间协商 TLS 交互的后续参数的初始协商。
- 对端（peer）: 一个端点，当说到一个特定端点，对端指的是非当前讨论的端点。
- 接收端（receiver）: 接收记录的端点。
- 发送者（sender）: 发送记录的端点。
- 服务器（server）: 没有发起TLS连接的端点。

### 1.2. Major Differences from TLS 1.2

以下是 TLS 1.2 和 TLS 1.3 的主要差异。这并不是全部差异，还有很多次要的差别。

- 支持的对称算法列表已经删除了所有被认为是遗留问题的算法。列表保留了所有使用 "关联数据的认证加密"（AEAD）算法。密码套件的概念已经改变，将认证和密钥交换机制与记录保护算法（包括密钥长度）和 Hash（用于秘钥导出函数和握手消息认证码 MAC）分离。
- 增加 0-RTT 模式，为一些应用数据在连接建立阶段节省了一次往返，这是以牺牲一定的安全特性为代价的。
- 静态 RSA 和 Diffie-Hellman 密码套件已经被删除；所有基于公钥的密钥交换算法现在都能提供前向安全。
- 所有 ServerHello 之后的握手消息现在都已经加密。扩展之前在 ServerHello 中以明文发送，新引入的 EncryptedExtensions 消息可以保证扩展以加密方式传输。
- 密钥导出函数被重新设计。新的设计使得密码学家能够通过改进的密钥分离特性进行更容易的分析。基于 HMAC 的提取-扩展密钥导出函数（HKDF）被用作一个基础的原始组件。
- Handshake 状态机进行了重大重构，以便更具一致性和删除多余的消息如 ChangeCipherSpec (除了中间件兼容性需要)。
- 椭圆曲线算法已经属于基本的规范，且包含了新的签名算法，如 EdDSA。TLS 1.3 删除了点格式协商以便于每个曲线使用单点格式。
- 其它的密码学改进包括改变 RSA 填充以使用 RSA 概率签名方案（RSASSA-PSS），删除压缩，数字签名算法 DSA，和定制 DHE 组（Ephemeral Diffie-Hellman）。
- 废弃了 TLS 1.2 的版本协商机制，以便在扩展中添加版本列表。这增加了不支持版本协商的 server 的兼容性。
- 之前版本中会话恢复（根据或不根据 server 端状态）和基于 PSK 的密码族已经被一个单独的新 PSK 交换所取代。
- 引用已更新至最新版本的 RFC（例如，[RFC5280] 而不是 [RFC3280]）。

### 1.3. Updates Affecting TLS 1.2

本文提出了几个可能影响 TLS 1.2 实现的变化，包括那些不支持 TLS 1.3 的实现：

- 4.1.3 节中的版本降级保护机制。
- 4.2.3 节中的 RSASSA-PSS 签名方案。
- ClientHello 中 "supported_versions" 扩展可以用于协商 TLS 使用的版本，优先于 ClientHello 中的 legacy_version 字段。
- "signature_algorithms_cert" 扩展允许一个 client 声明它使用哪种签名算法验证 X.509 证书。

此外，本文提出了支持 TLS 的早期版本需要实现的部分；见 9.3 节。

## 2. Protocol Overview

安全通道使用的密码参数由 TLS 握手协议生成。这个 TLS 的子协议在 client 和 server 第一次通信时使用。握手协议使两端协商协议版本、选择密码算法、选择性互相认证，并得到共享的密钥数据。一旦握手完成，双方就会使用得出的密钥保护应用层流量。

握手失败或其它协议错误会触发连接中止，在这之前可以有选择地发送一个 alert 消息（第6章）。

TLS支持3种基本的密钥交换模式：
- (EC)DHE (基于有限域或椭圆曲线的Diffie-Hellman)
- PSK-only
- PSK 结合 (EC)DHE

图 1 显示了基本 TLS 握手全过程：

```
       Client                                           Server

Key  ^ ClientHello
Exch | + key_share*
     | + signature_algorithms*
     | + psk_key_exchange_modes*
     v + pre_shared_key*       -------->
                                                  ServerHello  ^ Key
                                                 + key_share*  | Exch
                                            + pre_shared_key*  v
                                        {EncryptedExtensions}  ^  Server
                                        {CertificateRequest*}  v  Params
                                               {Certificate*}  ^
                                         {CertificateVerify*}  | Auth
                                                   {Finished}  v
                               <--------  [Application Data*]
     ^ {Certificate*}
Auth | {CertificateVerify*}
     v {Finished}              -------->
       [Application Data]      <------->  [Application Data]

              +  Indicates noteworthy extensions sent in the
                 previously noted message.

              *  Indicates optional or situation-dependent
                 messages/extensions that are not always sent.

              {} Indicates messages protected using keys
                 derived from a [sender]_handshake_traffic_secret.

              [] Indicates messages protected using keys
                 derived from [sender]_application_traffic_secret_N.

               Figure 1: Message Flow for Full TLS Handshake
```

握手可以分为三个阶段（上图中已表明）：

- 密钥交换：确定共享密钥材料并选择加密参数，在这个阶段之后所有的数据都会被加密。
- Server 参数：确定其它的握手参数（client是否被认证，应用层协议支持等）。
- 认证：认证 server（并且选择性认证 client），提供密钥确认和握手完整性。

在密钥交换阶段，client 会发送 ClientHello (4.1.1节) 消息，其中包含了一个随机 nonce (ClientHello.random)、协议版本、对称密码或 HKDF hash 对的列表、Diffie-Hellman 共享密钥列表（在4.2.8中的 "key_share" 扩展中）或预共享密钥标签列表（4.2.11 中的 "pre_shared_key" 扩展中），或二者都有、和其它额外扩展。还可能存在其他字段或消息，以实现中间件的兼容性。

Server 处理 ClientHello 并为连接确定合适的加密参数，然后以 ServerHello（4.1.3节）响应（其中携带了协商好的连接参数）。ClientHello 和 ServerHello 一起确定共享密钥。如果使用的是 (EC)DHE 密钥，则 ServerHello 中会包含一个携带临时 Diffie-Hellman 共享参数的 "key_share" 扩展，这个共享参数必须与 client 的在相同的组里。如果使用的是 PSK 密钥，则 ServerHello 中会包含一个 "pre_shared_key" 扩展以表明 client 提供的哪一个 PSK 被选中。需要注意的是实现上可以将 (EC)DHE 和 PSK 一起使用，这种情况下两种扩展都需要提供。

随后 Server 会发送两个消息来确定 Server 参数：

- EncryptedExtensions：用来响应不用于确定密码参数的 ClientHello 扩展，除了针对用户证书的扩展。[4.3.1]
- CertificateRequest: 如果需要基于证书的 client 认证，则包含与证书相关的参数。如果不需要 client 认证则此消息会被省略。[4.3.2]

最后，client 和 server 交换认证消息。TLS 在每次认证时使用相同的消息集，（基于 PSK 的认证随密钥交换进行）特别是：

- Certificate: 终端和任何每证书扩展的证书。如果不带证书认证则此消息会被 server 忽略；如果 server 没有发送 CertificateRequest（这表明 client 不使用证书认证），此消息会被 client 忽略。需要注意的是如果原始公钥[RFC7250] 或缓存信息扩展 [RFC7924] 正在被使用，则此消息不会包含证书而是包含一些与 server 的长期密钥相关的其它值。[4.4.2]
- CertificateVerify: 使用与证书消息中的公钥配对的私钥对整个握手消息进行签名。如果终端没有使用证书进行验证则此消息会被忽略。
- Finished: 对整个握手消息的 MAC(消息认证码)。这个消息提供了密钥确认，将终端身份与交换的密钥绑定在一起，这样在 PSK 模式下也能认证握手。[4.4.4]

接收到 server 的消息之后，client 会响应认证消息，即 Certificate，CertificateVerify (如果需要), 和 Finished。

这时握手已经完成，client 和 server 会提取出密钥材料用于记录层交换应用层数据，这些数据需要通过认证的加密来保护。应用层数据一定不能在 Finished 消息之前发送，必须等到记录层开始使用加密密钥之后才可以发送。需要注意的是server 可以在收到 client 的认证消息之前发送应用数据，任何在这个时间点发送的数据，当然都是在发送给一个未被认证的对端。

### 2.1. Incorrect DHE Share

如果 client 没有提供足够的 "key_share" 扩展（例如，只包含 server 不接受或不支持的 DHE 或 ECDHE 组），server 会使用 HelloRetryRequest 来纠正这个不匹配问题，client 需要使用一个合适的 "key_share" 扩展来重启握手，如图 2 所示。如果没有通用的密码参数能够协商，server 必须使用一个适当的 alert 来中止握手。

```
        Client                                               Server

        ClientHello
        + key_share             -------->
                                                  HelloRetryRequest
                                <--------               + key_share
        ClientHello
        + key_share             -------->
                                                        ServerHello
                                                        + key_share
                                              {EncryptedExtensions}
                                              {CertificateRequest*}
                                                     {Certificate*}
                                               {CertificateVerify*}
                                                         {Finished}
                                <--------       [Application Data*]
        {Certificate*}
        {CertificateVerify*}
        {Finished}              -------->
        [Application Data]      <------->        [Application Data]

             Figure 2: Message Flow for a Full Handshake with
                           Mismatched Parameters
```

注：这个握手过程包含初始的 ClientHello/HelloRetryRequest 交换，不能被新的 ClientHello 重置。

TLS 也支持几个基本握手中的优化变体，下面的章节将描述。

### 2.2. Resumption and Pre-Shared Key (PSK)

虽然 TLS 预共享密钥（PSK）能够在带外建立，PSK 也能在之前的连接中确定然后用来建立新连接（会话恢复或使用 PSK 恢复）。一旦握手完成，server 就能给 client 发送一个与来自初次握手的唯一密钥对应的 PSK 身份（见 4.6.1）。然后 client 能够使用这个 PSK 身份在将来的握手中协商相关 PSK 的使用。如果 server 接受 PSK，新连接的安全上下文在密码学上就与初始连接关联在一起，从初次握手中得到的密钥就会用于装载密码状态来替代完整的握手。在 TLS 1.2 以及更低的版本中，这个功能由 "session IDs" 和 "session tickets" [RFC5077] 来提供。这两个机制在 TLS 1.3 中都被废除。

PSK 可以与 (EC)DHE 密钥交换算法一同使用以便使共享密钥具备前向安全，PSK 也可以单独使用，这样以丢失了应用数据的前向安全为代价。

图3显示了两次握手，第一次建立了一个 PSK，第二次时使用它：

```
          Client                                               Server

   Initial Handshake:
          ClientHello
          + key_share               -------->
                                                          ServerHello
                                                          + key_share
                                                {EncryptedExtensions}
                                                {CertificateRequest*}
                                                       {Certificate*}
                                                 {CertificateVerify*}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Certificate*}
          {CertificateVerify*}
          {Finished}                -------->
                                    <--------      [NewSessionTicket]
          [Application Data]        <------->      [Application Data]


   Subsequent Handshake:
          ClientHello
          + key_share*
          + pre_shared_key          -------->
                                                          ServerHello
                                                     + pre_shared_key
                                                         + key_share*
                                                {EncryptedExtensions}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Finished}                -------->
          [Application Data]        <------->      [Application Data]

               Figure 3: Message Flow for Resumption and PSK
```

当 server 通过 PSK 进行认证时，它不会发送 Certificate 或 CertificateVerify 消息。当 client 通过 PSK 恢复时，它也应当提供一个 "key_share" 给server，以允许 server 在需要的情况下拒绝恢复并回退到完整的握手。Server 响应一个 "pre_shared_key" 扩展来协商确定 PSK 密钥的使用方法，并响应一个 "key_share" 扩展（如图所示）来进行 (EC)DHE 密钥确定，由此提供前向安全。

当 PSK 在带外提供时，必须一起提供 PSK 身份和与 PSK 一起使用的 KDF hash 算法。

注：当使用带外提供的预共享密钥时，一个关键的考虑是在密钥生成时使用足够的熵，就像 [RFC4086] 中讨论的那样。从口令或其它低熵源导出的共享密钥并不安全。一个低熵密码或口令，容易受到基于PSK绑定器的字典攻击。就算使用了 Diffie-Hellman 密钥建立方法，这种 PSK 认证也不是强口令认证的密钥交换。它不能防止可以观察到握手的攻击者对 密码/PSK 执行暴力攻击。

### 2.3. 0-RTT Data

当 client 和 server 共享一个 PSK（从外部获得或通过以前的握手获得）时，TLS 1.3 允许 client 在第一个发送出去的消息（"early data"）中携带数据。Client 使用这个 PSK 来认证 server 并加密 early data。

如图 4 所示，0-RTT 数据在第一个发送的消息中被加入到 1-RTT 握手里。握手的其余消息与带 PSK 恢复的 1-RTT 握手消息相同。

```
         Client                                               Server

         ClientHello
         + early_data
         + key_share*
         + psk_key_exchange_modes
         + pre_shared_key
         (Application Data*)     -------->
                                                         ServerHello
                                                    + pre_shared_key
                                                        + key_share*
                                               {EncryptedExtensions}
                                                       + early_data*
                                                          {Finished}
                                 <--------       [Application Data*]
         (EndOfEarlyData)
         {Finished}              -------->
         [Application Data]      <------->        [Application Data]

               +  Indicates noteworthy extensions sent in the
                  previously noted message.

               *  Indicates optional or situation-dependent
                  messages/extensions that are not always sent.

               () Indicates messages protected using keys
                  derived from a client_early_traffic_secret.

               {} Indicates messages protected using keys
                  derived from a [sender]_handshake_traffic_secret.

               [] Indicates messages protected using keys
                  derived from [sender]_application_traffic_secret_N.

               Figure 4: Message Flow for a 0-RTT Handshake
```

注意：0-RTT 数据的安全属性比其它类型的 TLS 数据弱，特别是：

1. 这类数据没有前向安全，它只使用了从提供的 PSK 中导出的密钥进行加密。
2. 不能保证在多条连接之间不会重放。为普通的 TLS 1.3 1-RTT 数据提供抗重放的保护方法是使用 server 的随机数据，但 0-RTT 不依赖于 ServerHello，因此只能得到更弱的保护。如果数据使用 TLS client 认证或在应用协议认证，这一点尤其重要。这个警告适用于任何使用 early_exporter_master_secret 的情况。

0-RTT 数据不能在一个连接内复制（如 server 不会处理同一连接内相同数据两次），攻击者不能使 0-RTT 数据看起来像 1-RTT 数据（因为它们是用不同秘钥保护的）。附录 E.5 包含了潜在攻击的描述，第 8 章描述了 server 可以用来限制重放影响的机制。

## 3. Presentation Language

本文使用另外的表示方法处理数据格式。下面会用到非常基础甚至是有些随便定义的表示语法。

### 3.1. Basic Block Size

所有数据条目的描述都是被显示指定的。基本数据块大小是 1 字节（即 8 位）。多字节数据条目是字节的串联，从左到右，从上到下。从字节流的角度看，一个多字节条目（在例子中是一个数值）的组织方式（使用 C 语言记法）如下：

```
      value = (byte[0] << 8*(n-1)) | (byte[1] << 8*(n-2)) |
              ... | byte[n-1];
```

多字节数值字节序是普通的网络字节序或大端格式。

### 3.2. Miscellaneous

注释以 /*" and end with "*/ 开始。

可选组件使用 [[ ]] 括起来(两个中括号).

包含未解释数据的单字节实体属于 opaque 类型。

可以为一个现有类型 T 定义一个别名 T':

```
      T T';
```

### 3.3. Numbers

基本数字数据类型是一个无符号字节（uint8）.所有更大的数据类型都是由固定长度的字节序列构成的，如第 3.1 节所述，这些字节串接在一起，并且也是无符号的。以下数字类型是预定义的：

```
      uint8 uint16[2];
      uint8 uint24[3];
      uint8 uint32[4];
      uint8 uint64[8];
```

规范中的所有值都以网络字节（大端序）顺序传输，由十六进制字节 01 02 03 04 表示的 uint32 相当于十进制值 16909060

### 3.4. Vectors

一个向量（一维数组）是同类数据元素的流。向量的大小可能在编写文档时指定或留待运行时确定。在任何情况下，向量的长度都是指字节数而非元素数。定义一个新类型 T' (是一个固定长度的类型T的向量)的语法是：

```
      T T'[n];
```

这里 T' 在数据流中占据了 n 个字节，而 n 是多个 T 的大小。向量的长度并不包含在编码流中。

在下面的例子中，Datum 被定义为协议不能理解的 3 个连续字节， 而 Data 是三个连续的 Datum，共占据 9 个字节。

```
      opaque Datum[3];      /* three uninterpreted bytes */
      Datum Data[9];        /* three consecutive 3-byte vectors */
```

变长向量的定义是通过指定一个合法长度的子范围来实现（使用符号 <floor..ceiling> ）。**当变长向量被编码时，在字节流中实际长度是在向量的内容之前。这个长度会以一个数字的形式出现，并使用足够多的字节以便表示向量的最大长度（ceiling）。**一个实际长度是 0 的变长向量会被当做一个空向量。

```
      T T'<floor..ceiling>;
```

在下面的例子中会强制要求一个向量必须包含 300-400 个字节的 opaque 类型数据，不能为空。实际长度字段占用两个字节，为一个 uint16，这足以代表数值400（见3.4节）。相似地，"longer" 可以描述多达 800 字节的数据，或 400 个 uint16 类型的元素，可以为空。它的编码会在向量前包含一个两字节的实际长度字段。一个编码向量的长度必须是单个元素长度的偶数倍（例如，一个 17 字节长的 uint16 类型的向量是非法的）。

```
      opaque mandatory<300..400>;
            /* length field is two bytes, cannot be empty */
      uint16 longer<0..800>;
            /* zero to 400 16-bit unsigned integers */
```

### 3.5. Enumerateds

另外一种简单的数据类型是枚举。每个定义都是一个不同的类型。只有相同类型的枚举能被赋值或比较。枚举的每个元素都必须指定一个值，如下所示。因为枚举类型的元素并不是有序的，所以它们能够以任意顺序指定任意唯一值。

```
      enum { e1(v1), e2(v2), ... , en(vn) [[, (n)]] } Te;
```

将来对协议的扩展或添加会定义新的值。实现需要能够解析或者忽略未知的值，除非字段的定义另有说明。

一个枚举在字节流中占据的空间需要足够存储其定义的最大有序数值。下面的定义会使用 1 个字节来表示 Color 类型的字段。

```
      enum { red(3), blue(5), white(7) } Color;
```

一个选择是指定一个值但不关联标记以强制定义枚举的大小，这样无需定义一个多余的元素。

在下面这个例子中，Taste 在字节流中会消耗 2 个字节， 但在当前的协议版本中只能表示数值 1, 2, 4。

```
      enum { sweet(1), sour(2), bitter(4), (32000) } Taste;
```

一个枚举类型的元素名称被局限于定义的类型。在第一个例子中，对枚举的第二个元素的完全合格的引用是 Color.blue，如果能很好的定义赋值目标，就不需要这样的格式。

```
      Color color = Color.blue;     /* overspecified, legal */
      Color color = blue;           /* correct, type implicit */
```

指定给枚举的名字不需要是唯一的。数字可以描述使用相同名字的一个范围。这个值包括取值范围中最小和最大值，由两个点分隔开。主要用于保留空间区域。

```
      enum { sad(0), meh(1..254), happy(255) } Mood;
```

### 3.6. Constructed Types

为了方便，结构体类型可以由原始类型组成。每个规范声明了一个新的、唯一的类型。定义的语法很像 C 语言：

```
      struct {
          T1 f1;
          T2 f2;
          ...
          Tn fn;
      } T;
```

定长和变长向量字段可以使用标准的向量语法。下文中变量示例（3.8）中的结构体 V1 和 V2 表明了这一点。

结构体内的字段可以用类型的名字来描述，用类似于枚举的语法。例如，T.f2 引用了前面定义的结构的第二个字段。

### 3.7. Constants

字段和常量可以使用 "=" 来赋一个固定的值，像下面那样：

```
      struct {
          T1 f1 = 8;  /* T.f1 must always be 8 */
          T2 f2;
      } T;
```

### 3.8. Variants

考虑到从环境中得到的一些信息是变化的，定义的结构体可以有一些变量。选择符必须是一个定义了结构体中变量取值的枚举类型。Select 语句的每个分支指定了变量字段的类型和一个可选的字段标签。通过这个机制变量可以在运行时被选择，而不是被表示语言描述。

```
      struct {
          T1 f1;
          T2 f2;
          ....
          Tn fn;
          select (E) {
              case e1: Te1 [[fe1]];
              case e2: Te2 [[fe2]];
              ....
              case en: Ten [[fen]];
          };
      } Tv;
```

例如：

```
      enum { apple(0), orange(1) };

      struct {
          uint16 number;
          opaque string<0..10>; /* variable length */
      } V1;

      struct {
          uint32 number;
          opaque string[10];    /* fixed length */
      } V2;

      struct {
          VariantTag type;
          select (VariantRecord.type) {
              case apple:  V1;
              case orange: V2;
          };
      } VariantRecord;
```

## 4. Handshake Protocol

握手协议用于协商连接的安全参数。把握手消息传递给 TLS 记录层，TLS 记录层把它们封装到一个或多个 TLSPlaintext 或 TLSCiphertext 中，然后按照当前活动连接状态的规定进行处理和传输。

```
      enum {
          client_hello(1),
          server_hello(2),
          new_session_ticket(4),
          end_of_early_data(5),
          encrypted_extensions(8),
          certificate(11),
          certificate_request(13),
          certificate_verify(15),
          finished(20),
          key_update(24),
          message_hash(254),
          (255)
      } HandshakeType;

      struct {
          HandshakeType msg_type;    /* handshake type */
          uint24 length;             /* remaining bytes in message */
          select (Handshake.msg_type) {
              case client_hello:          ClientHello;
              case server_hello:          ServerHello;
              case end_of_early_data:     EndOfEarlyData;
              case encrypted_extensions:  EncryptedExtensions;
              case certificate_request:   CertificateRequest;
              case certificate:           Certificate;
              case certificate_verify:    CertificateVerify;
              case finished:              Finished;
              case new_session_ticket:    NewSessionTicket;
              case key_update:            KeyUpdate;
          };
      } Handshake;
```

协议消息必须以 4.4.1 中定义的顺序发送，这也已经在第 2 章的图中展示了。对端如果收到了不按顺序发送的握手消息必须使用一个 "unexpected_message" alert 来中止握手。

新的握手消息类型已经由 IANA 指定并在第 11 章中描述。

### 4.1. Key Exchange Messages

密钥交换消息用于确定 client 和 server 之间的安全能力，并确定包括流量密钥在内的共享密钥来保护其余的握手消息和数据。

#### 4.1.1. Cryptographic Negotiation

在TLS中，密码协商通过 client 在 ClientHello 中提供下面 4 个选项集合来实现：

- 一个密码族列表，指的是 client 所支持的 AEAD 算法或 HKDF hash 对。
- 一个 "supported_groups" (4.2.7) 扩展，指的是 client 支持的 (EC)DHE 组；一个 "key_share" (4.2.8) 扩展，包含了一些或全部组所共享的 (EC)DHE 秘钥。
- 一个 "signature_algorithms" (4.2.3) 扩展，指的是 client 能接受的签名算法；可能会添加一个 "signature_algorithms_cert" 扩展（4.2.3）来指明证书指定的签名算法。
- 一个 "pre_shared_key" (4.2.11) 扩展，包含了一个 client 知晓的对称密钥；一个 "psk_key_exchange_modes" (4.2.9) 扩展，表明与 PSK 一起使用的密钥交换模式。

如果 server 没有选择PSK，则这些选项的前 3 个是完全正交的：server 独立地选择一个密码族，一个 (EC)DHE 组和用于确定密钥的密钥共享，一个签名算法或证书对用于认证自己和 client。如果接收到的 "supported_groups" 和 server 所支持的组之间没有重叠，则 server 必须用一个 "handshake_failure" 或一个 "insufficient_security" alert 中止握手。

如果 server 选择了一个 PSK，则必须从 client 的 "psk_key_exchange_modes" 扩展（目前是仅 PSK 或带 (EC)DHE）所携带的集合中选择一个密钥建立模式。需要注意的是如果可以不带 (EC)DHE 就使用 PSK，则 "supported_groups" 参数不重叠不一定是致命的，就像之前讨论过的非 PSK 场景一样。

如果 server 选择了一个 (EC)DHE 组并且 client 没有在初始 ClientHello 中提供兼容的 "key_share" 扩展，server 必须响应 HelloRetryRequest (4.1.4) 消息。

如果 server 成功地选择了参数且没有发送 HelloRetryRequest，表明 ServerHello 中所选的参数如下：

- 如果使用 PSK，则 server 会发送一个 "pre_shared_key" 扩展表明所选的密钥。
- 使用 (EC)DHE 时，server 也会提供 "key_share" 扩展。如果没有使用 PSK，则一直使用 (EC)DHE 和基于证书的认证。
- 当通过证书进行验证时，server 将会发送 Certificate(4.4.2) 和 CertificateVerify(4.4.3) 消息。在本文定义的 TLS 1.3 中，会一直使用一个 PSK 或一个证书，但两者不会同时使用。将来的文档可能会定义怎样同时使用它们。

如果 server 不能协商出支持的参数集合（例如，client 和 server 的参数集合没有重叠），它必须用一个 "handshake_failure" 或一个 "insufficient_security" alert（见第6章）中止握手。

#### 4.1.2. Client Hello

当 client 第一次连接一个 server 时，它需要发送 ClientHello 作为第一个消息。当 server 用 HelloRetryRequest 来响应 client 的 ClientHello 时，client 也应当发送 ClientHello。这种条件下，client 必须发送相同的 ClientHello（无修改），除非：

- 如果 HelloRetryRequest 带有一个 "key_share" 扩展，则将共享列表用包含指定组中的一个 KeyShareEntry 的列表取代。
- 如果存在 "early_data" 扩展则将其移除。Early data 不允许在 HelloRetryRequest 之后出现。
- 如果 HelloRetryRequest 中提供了一个 "cookie" 扩展，则需要也包含一个 "cookie" 扩展。
- 如果需要重新计算 "obfuscated_ticket_age" 和绑定值、(可选地)删除与 server 指定的密码族不兼容的任何 PSK，则更新 "pre_shared_key" 扩展。
- 选择性增加、删除或更改 "padding" 扩展 [RFC7685] 的长度。
- 将来定义的其他 HelloRetryRequest 中扩展允许的修改。

由于 TLS 1.3 禁止重协商，如果 server 已经协商完成了 TLS 1.3，在任何其它时间收到了 ClientHello，必须用 "unexpected_message" alert 中止连接。

如果 server 用以前版本的 TLS 建立了连接并在重协商时接收了一个 TLS 1.3 的 ClientHello，它必须保持以前的协议版本，不能协商 TLS 1.3。

这个消息的结构：

```
      uint16 ProtocolVersion;
      opaque Random[32];

      uint8 CipherSuite[2];    /* Cryptographic suite selector */

      struct {
          ProtocolVersion legacy_version = 0x0303;    /* TLS v1.2 */
          Random random;
          opaque legacy_session_id<0..32>;
          CipherSuite cipher_suites<2..2^16-2>;
          opaque legacy_compression_methods<1..2^8-1>;
          Extension extensions<8..2^16-1>;
      } ClientHello;
```

- **legacy_version**：在以前版本的 TLS 里，这个字段用于版本协商和表示 client 支持的最高版本号。经验表明很多 server 没有适当地实现版本协商，导致 "版本容忍"，会使 server 拒绝本来可接受的版本号高于支持的ClientHello。在 TLS 1.3 中，client 在 "supported_versions" 扩展(4.2.1节)中表明版本偏好，且 legacy_version 字段必须设置为 TLS 1.2 的版本号 0x0303。TLS 1.3 ClientHello 的 legacy_version 为 0x0303，supported_versions 扩展值为最高版本 0x0304（关于后向兼容的细节见附录 D）
- **random**： 由一个安全随机数生成器产生的 32 字节随机数，更多信息见附录 C。
- **legacy_session_id**： 之前的 TLS 版本支持 "会话恢复" 特性，在 TLS 1.3 中此特性与预共享密钥合并了(见 2.2 节)。client 需要将此字段设置为 TLS 1.3 之前版本的 server 缓存的 session ID。在兼容模式下(见附录 D.4)此字段必须非空，所以一个不能提供 TLS 1.3 之前版本会话的 client 必须产生一个 32 字节的新值。这个值不必是随机的但应该是不可预测的以避免实现上固定为一个具体的值(也被称为僵化)。否则，它必须被设置为一个 0 长度向量(例如，一个 0 字节长度字段)。
- **cipher_suites**： client 支持的对称密码族选项列表，具体有记录保护算法（包括密钥长度）和与 HKDF 一起使用的 hash 算法，这些算法以 client 偏好降序排列。值定义在附录 B.4。如果列表中包含 server 不认识，不支持，或不想使用的密码族，server 必须忽略这些密码族并且正常处理其余的密码族。如果 client 试图确定一个 PSK，应该通告至少一个密码族以表明 PSK 关联 hash 算法。
- **legacy_compression_methods**：TLS 1.3 以前的版本支持压缩，并提供一个支持的压缩方法列表。对于 TLS 1.3 的每个ClientHello，这个向量必须只包含 1 字节并设置为 0，对应以前 TLS 版本的 "null" 压缩方法。如果收到的 TLS 1.3 ClientHello 这个字段是其它的值，server 必须以一个 "illegal_parameter" alert 来中止握手。请注意，TLS 1.3 server 可能会接收 TLS 1.2 或之前的 ClientHello 包含其它压缩方法，（如果与这样一个先前版本协商）必须遵循适当的 TLS 先前版本的流程。
- **extension**： Client 通过在 extension 字段发送数据来向 server 请求扩展的功能。实际的 "Extension" 格式定义在 4.2 节中。TLS 1.3 强制使用特定的 extension，因为一些功能被转移到了 extension 里以保留 ClientHello 与先前 TLS 版本的兼容性。Server 必须忽略无法识别的 extension。

所有版本的 TLS 都允许 compression_method 字段后跟着 extension 字段。TLS 1.3 ClientHello 消息始终包含 extension（至少包含 "supported_versions"，否则将被当做 TLS 1.2 ClientHello消息）。但是，TLS1.3 server 可能会从以前版本的 TLS 接收不带 extension 字段的 ClientHello 消息。可以通过 ClientHello 末尾 compression_methods 字段后面是否有字节来检测扩展的存在。注意，这种检测可选数据的方法不同于检测可变长度字段的常规 TLS 方法，但是可用于与 extension 定义之前的 TLS 版本兼容。TLS 1.3 server 需要首先执行此检查，并且只有存在 "supported_versions" 扩展时才尝试协商 TLS 1.3。如果协商 TLS 1.3 之前的版本，server 必须检查消息是否在 legacy_compression_methods 之后不包含任何数据，或者是否包含一个后面没有数据的有效扩展块。如果不是，则必须以 "decode_error" alert 中止握手。

如果 client 使用 extension 请求附加功能，而 server 不提供此功能，则 client 可能会中止握手。

client 发送 ClientHello 消息后，等待 ServerHello 或 HelloRetryRequest 消息。如果正在使用 early data，则 client 可以在等待下一个握手消息时传输早期应用数据（2.3节）。

#### 4.1.3. Server Hello

当可以根据 ClientHello 协商出一组可接受的握手参数时，server 将发送此消息响应 ClientHello 消息以继续握手流程。

此消息的结构：

```
      struct {
          ProtocolVersion legacy_version = 0x0303;    /* TLS v1.2 */
          Random random;
          opaque legacy_session_id_echo<0..32>;
          CipherSuite cipher_suite;
          uint8 legacy_compression_method = 0;
          Extension extensions<6..2^16-1>;
      } ServerHello;
```

- **legacy_version**：在此前的 TLS 版本中，此字段用来版本协商和承载连接的选定版本号。不幸的是，当存在新值时，一些中间件会处理失败。在 TLS1.3 中，TLS sever 使用 "supported_versions" 扩展来表明它的版本（4.2.1节），legacy_version 字段必须设置为 TLS1.2 的版本号 0x0303。（前向兼容具体见附录 D）
- **random**：由安全随机数生成器生成的 32 字节，其他信息见附录 C。如果协商 TLS 1.2 或 TLS 1.1，最后 8 个字节必须被重写，如下所述。该结构由 server 生成，必须与 ClientHello.random 分别生成。
- **legacy_session_id_echo**：client 的 legacy_session_id 字段内容。请注意，即使客户端的值与服务器选择不恢复的缓存的 TLS 1.3 之前会话相对应，也会回复此字段。如果客户端接收到与发送内容不匹配的legacy_session_id_echo 字段，则必须使用 "illegal_parameter" alert 中止握手。
- **cipher_suite**：服务器从 ClientHello.cipher_suites 列表中选择的单个密码套件。客户端接收到自己没有提供的密码套件必须中止握手。
- **legacy_compression_method**: 一个字节，值必须是 0。
- **extensions**：扩展列表。 ServerHello 必须只包含确定加密上下文和协商版本号所需的扩展。所有当前的 TLS 1.3 ServerHello 消息必须包含 "supported_versions" 扩展。目前 ServerHello 消息额外包含 "pre_shared_key" 扩展和 "key_share" 扩展之一，或者两者都包含（当使用 PSK 和 (EC)DHE 秘钥建立）。其他扩展（见4.2节）单独在 EncryptedExtensions 消息里发送。

由于中间件的后向兼容性原因（见附录D.4），HelloRetryRequest 消息与 ServerHello 使用相同的结构体，但带有设置为特殊值（ "HelloRetryRequest" 的 SHA-256）的随机值：

```
     CF 21 AD 74 E5 9A 61 11 BE 1D 8C 02 1E 65 B8 91
     C2 A2 11 16 7A BB 8C 5E 07 9E 09 E2 C8 A8 33 9C
```

在接收到类型为 server_hello 的消息时，必须首先检查随机值，如果该值与此匹配，则按照 4.1.4 节所述进行处理。

TLS 1.3 使用服务器的随机值实现降级保护机制。 TLS 1.3 服务器以 TLS 1.2 或更低版本响应 ClientHello 必须专门设置随机值的最后 8 个字节。

如果协商 TLS 1.2，TLS 1.3 服务器必须将随机值的最后 8 个字节设置为：

```
     44 4F 57 4E 47 52 44 01
```

如果协商 TLS 1.1 或更低版本，TLS 1.3 服务器和 TLS 1.2 服务器应该将随机值的最后 8 个字节设置为：

```
     44 4F 57 4E 47 52 44 00
```

TLS 1.3 客户端接收到 TLS 1.2 或更低版本的 ServerHello 必须检查最后八个字节不等于这些值。如果 ServerHello 指示 TLS 1.1 或更低版本，TLS 1.2 客户端还应该检查最低 8 字节不等于第二个值。 客户端如果找到匹配值则必须用 "illegal_parameter" alert 中止握手。这种机制提供了有限的保护，以防止降级攻击超出 Finished 交换提供内容：因为 TLS1.2 及以下版本的 ServerKeyExchange 消息包含两个随机值的签名，只要使用临时密码，攻击者就不可能探测修改随机值。当使用静态 RSA 时就不提供降级保护。

注意：这是从 [RFC5246] 开始修改的，所以实际上许多 TLS 1.2 客户端和服务器可能不会像上面说的那样。

旧的 TLS 客户端使用 TLS 1.2 或之前的版本执行重协商，重协商期间如果接收到 TLS 1.3 的 ServerHello，必须使用 "protocol_version" alert 终止握手。需要注意的是，如果 TLS 1.3 已经协商好了就不会重协商了。

#### 4.1.4. Hello Retry Request

如果可以找到可接受的一组参数，但 ClientHello 中没有足够的信息继续处理握手流程，服务器就将这个消息作为 ClientHello 的回应。如 4.1.3 节所说，HelloRetryRequest 跟 ServerHello 格式一样，并且 legacy_version、legacy_session_id_echo、cipher_suite 和 legacy_compression_method 字段的含义也一样。然而，为方便起见，本文中把 "HelloRetryRequest" 当做不同的消息讨论。

服务器扩展必须包含 "supported_versions"。另外，应该包含客户端产生正确 ClientHello 对的必须扩展最小集。像 ServerHello 一样，HelloRetryRequest 一定不能包含客户端 ClientHello 中未提供的扩展，可选 "cookie" 扩展除外（4.2.2节）。

Client 接收到 HelloRetryRequest 必须如 4.1.3 所说的检查 egacy_version、legacy_session_id_echo、cipher_suite 和 legacy_compression_method，然后处理扩展，先用 "supported_versions" 确定版本号。如果 HelloRetryRequest 不会对 ClientHello 造成任何改变，Client 必须以 "illegal_parameter" alert 终止握手。如果 client 在同一个链接中收到第二个 HelloRetryRequest（如即以 ClientHello 本身响应 HelloRetryRequest），必须以 "unexpected_message" alert 终止连接。

否则，客户端必须处理 HelloRetryRequest 中的所有扩展，并发送第二个更新的 ClientHello。本规范中定义的 HelloRetryRequest 扩展包括：

-  supported_versions (见4.2.1节)
-  cookie (见4.2.2节)
-  key_share (见4.2.8节)

Client 接收到一个没有提供过的密码套件必须终止握手。Server 接收到合法更新的 ClientHello 时，必须确保协商相同密码套件（如果 server 选择了协商过程中第一步的密码套件，这就会自动发生）。client 接收到 ServerHello 后，必须检查其中的密码套件是否与 HelloRetryRequest 中的相同，否则以 "illegal_parameter" alert 终止握手。

此外，在更新的 ClientHello 中，客户端不应提供与所选密码套件以外的哈希相关联的任何预共享密钥。 这允许客户端避免在第二个 ClientHello 中计算多个哈希的部分哈希转录。

接收未提供的密码套件的客户端必须中止握手。 服务器必须确保在接收到一致的更新 ClientHello 时协商相同的密码套件（如果服务器选择密码套件作为协商的第一步，则会自动发生）。客户端收到 ServerHello 后必须检查 ServerHello 中提供的密码套件是否与 HelloRetryRequest 中的密码套件相同，否则将以 "illegal_parameter" alert 中止握手。

HelloRetryRequest 中的 "supported_versions" 扩展的 selected_version 值必须在 ServerHello 中保留，如果值变化了，client 必须以 "illegal_parameter" alert 终止握手。

### 4.2. Extensions

许多 TLS 消息包含 tag-length-value 编码的扩展结构：

```
    struct {
        ExtensionType extension_type;
        opaque extension_data<0..2^16-1>;
    } Extension;

    enum {
        server_name(0),                             /* RFC 6066 */
        max_fragment_length(1),                     /* RFC 6066 */
        status_request(5),                          /* RFC 6066 */
        supported_groups(10),                       /* RFC 8422, 7919 */
        signature_algorithms(13),                   /* RFC 8446 */
        use_srtp(14),                               /* RFC 5764 */
        heartbeat(15),                              /* RFC 6520 */
        application_layer_protocol_negotiation(16), /* RFC 7301 */
        signed_certificate_timestamp(18),           /* RFC 6962 */
        client_certificate_type(19),                /* RFC 7250 */
        server_certificate_type(20),                /* RFC 7250 */
        padding(21),                                /* RFC 7685 */
        pre_shared_key(41),                         /* RFC 8446 */
        early_data(42),                             /* RFC 8446 */
        supported_versions(43),                     /* RFC 8446 */
        cookie(44),                                 /* RFC 8446 */
        psk_key_exchange_modes(45),                 /* RFC 8446 */
        certificate_authorities(47),                /* RFC 8446 */
        oid_filters(48),                            /* RFC 8446 */
        post_handshake_auth(49),                    /* RFC 8446 */
        signature_algorithms_cert(50),              /* RFC 8446 */
        key_share(51),                              /* RFC 8446 */
        (65535)
    } ExtensionType;
```

这里:

- "extension_type" 表示特定扩展类型。
- "extension_data" 包含特定扩展类型指定的信息。

扩展类型表由 IANA 维护，见 11 章。

扩展通常以 请求/响应 方式构造，尽管一些扩展仅仅是没有相应响应的指示。客户端在 ClientHello 消息中发送其扩展请求，服务器在 ServerHello、EncryptedExtensions 和 HelloRetryRequest 消息中发送其扩展响应。服务器在 CertificateRequest 消息中发送扩展请求，客户端可能以 Certificate 消息进行响应。服务器也可以在 NewSessionTicket 中发送未经请求的扩展，但客户端不直接响应这些。

如果对端没有发送相应的扩展请求（除 HelloRetryRequest 中的 "cookie" 扩展外），严禁发送扩展响应。在接收到这样的扩展时，端点必须用 "unsupported_extension" alert 中止握手。

下表列出了给定扩展可能出现的消息：

- CH (ClientHello)
- SH (ServerHello)
- EE (EncryptedExtensions)
- CT (Certificate)
- CR (CertificateRequest)
- NST (NewSessionTicket)
- HRR (HelloRetryRequest)

如果接收到可识别的扩展，并且对应消息未指定它，必须用 "illegal_parameter" alert 来中止握手。

```
   +--------------------------------------------------+-------------+
   | Extension                                        |     TLS 1.3 |
   +--------------------------------------------------+-------------+
   | server_name [RFC6066]                            |      CH, EE |
   |                                                  |             |
   | max_fragment_length [RFC6066]                    |      CH, EE |
   |                                                  |             |
   | status_request [RFC6066]                         |  CH, CR, CT |
   |                                                  |             |
   | supported_groups [RFC7919]                       |      CH, EE |
   |                                                  |             |
   | signature_algorithms (RFC 8446)                  |      CH, CR |
   |                                                  |             |
   | use_srtp [RFC5764]                               |      CH, EE |
   |                                                  |             |
   | heartbeat [RFC6520]                              |      CH, EE |
   |                                                  |             |
   | application_layer_protocol_negotiation [RFC7301] |      CH, EE |
   |                                                  |             |
   | signed_certificate_timestamp [RFC6962]           |  CH, CR, CT |
   |                                                  |             |
   | client_certificate_type [RFC7250]                |      CH, EE |
   |                                                  |             |
   | server_certificate_type [RFC7250]                |      CH, EE |
   |                                                  |             |
   | padding [RFC7685]                                |          CH |
   |                                                  |             |
   | key_share (RFC 8446)                             | CH, SH, HRR |
   |                                                  |             |
   | pre_shared_key (RFC 8446)                        |      CH, SH |
   |                                                  |             |
   | psk_key_exchange_modes (RFC 8446)                |          CH |
   |                                                  |             |
   | early_data (RFC 8446)                            | CH, EE, NST |
   |                                                  |             |
   | cookie (RFC 8446)                                |     CH, HRR |
   |                                                  |             |
   | supported_versions (RFC 8446)                    | CH, SH, HRR |
   |                                                  |             |
   | certificate_authorities (RFC 8446)               |      CH, CR |
   |                                                  |             |
   | oid_filters (RFC 8446)                           |          CR |
   |                                                  |             |
   | post_handshake_auth (RFC 8446)                   |          CH |
   |                                                  |             |
   | signature_algorithms_cert (RFC 8446)             |      CH, CR |
   +--------------------------------------------------+-------------+
```

当存在不同类型的多个扩展时，扩展可以以任何顺序出现，但 "pre_shared_key"（4.2.11节）必须是 ClientHello 中的最后一个扩展（但在 ServerHello 扩展块中可以出现在任何位置）。给定扩展块中同一类型的扩展只能出现一次。
与 TLS 1.2 不同的是，TLS 1.3 即使在 resumption-PSK 模式下，每次握手都重新协商扩展。然而，0-RTT 参数是在先前握手中协商的，不匹配可能需要拒绝 0-RTT（见4.2.7节）。

在新协议中可能会出现的新特性与现有功能之间存在微妙（但也不是那么微妙）交互，这可能会导致整体安全性的显著降低。设计新扩展时，应考虑以下注意事项：

- 服务器不同意扩展的一些情况是错误条件（如握手无法继续），有些则简单地拒绝支持特定功能。一般来说，前者应该使用错误警报，后者的服务器扩展响应中会有一个字段。
- 扩展应尽可能设计为防止任何通过操纵握手信息强制使用（或不使用）特定功能的攻击。无论该功能是否会引起安全问题，都应遵循这一原则。通常，扩展字段被包含在 Finished 消息散列的输入中是足够的，但是当扩展改变在握手阶段中发送消息的含义时，需要特别小心。设计者和实现者应该意识到，在握手被验证之前，主动攻击者可以修改消息并插入，删除或替换扩展。

#### 4.2.1. Supported Versions

```
      struct {
          select (Handshake.msg_type) {
              case client_hello:
                   ProtocolVersion versions<2..254>;

              case server_hello: /* and HelloRetryRequest */
                   ProtocolVersion selected_version;
          };
      } SupportedVersions;
```

客户端使用 "supported_versions" 扩展来表明自己支持哪些版本的 TLS。该扩展包含的版本列表以偏好顺序排列，第一个是最优先的版本。这个规范的实现必须在扩展中发送包含准备协商的 TLS 所有版本（对于这个规范，这意味着最低  0x0304，但是如果支持 TLS 以前的版本，也必须包含进去）。

如果 ClientHello 中没有此扩展，那么符合本规范的服务器必须按照 [RFC5246] 中的规定协商 TLS 1.2 或先前版本，就算 ClientHello.legacy_version 为 0x0304 或更高版本也是如此。Server 接收到 legacy_version 为 0x0304 的 ClientHello 可以终止握手。

如果 ClientHello 中有此扩展，服务器禁止使用 ClientHello.legacy_version 值来进行版本协商，并且必须只使用 "supported_versions" 扩展来确定客户端偏好。服务器必须只选择该扩展中存在的 TLS 版本，并且必须忽略任何未知版本。注意，如果一方支持稀疏范围，这种机制可以协商出 TLS 1.2 之前的版本。选择支持 TLS 以前版本的 TLS 1.3 的实现应支持 TLS 1.2。服务器应必须准备好接收包含此扩展的 ClientHello，但不要在版本列表中包含 0x0304。

协商 TLS 1.3 之前版本 TLS 的服务端必须设置 ServerHello.version，且不能发送 "supported_versions" 扩展。 协商 TLS 1.3 的服务器必须以包含选定版本值（0x0304）的 "supported_versions" 扩展回应。必须将 ServerHello.legacy_version 字段设置为 0x0303 (TLS 1.2)。客户端必须在处理其余 ServerHello 之前检查此扩展（虽然必须解析 ServerHello 来读取扩展）。如果此扩展出现，客户端必须忽略 ServerHello.legacy_version 值，且必须只使用 "supported_versions" 扩展来确定选定版本。如果 ServerHello 中的 "supported_versions" 扩展包含客户端没提供的版本或者包含 TLS 1.3 之前的版本，客户端必须以 "illegal_parameter" alert 终止握手。

#### 4.2.2. Cookie

```
      struct {
          opaque cookie<1..2^16-1>;
      } Cookie;
```

Cookie有两个主要目的：

- 允许服务器强制客户端展示其网络地址可达性（从而提供 DoS 保护）。这主要用于非面向连接的传输（见[RFC6347]）。
- 允许服务器向客户端卸载状态，从而可以发送 HelloRetryRequest 而不存储任何状态。服务器通过将 ClientHello 的哈希存储在 HelloRetryRequest cookie 中（用一些合适的完整性算法保护）来实现。

当发送 HelloRetryRequest 时，服务器可以向客户端提供 "cookie" 扩展（这是通常规则的一个例外，即只能发送出现在 ClientHello 中的扩展）。 当发送新的 ClientHello 时，客户端必须将 HelloRetryRequest 中收到的 cookie 复制到新 ClientHello 中的 "cookie" 扩展中。客户端不得在后续连接 initial ClientHello 中使用 Cookie。

无状态服务器可能在第一个和第二个 ClientHello 之间接收到一个 change_cipher_spec 类型的未加密数据（见第5章）。因为服务器不存储任何状态，看起来就像接收到的第一个消息。无状态服务器必须忽略这些数据。

#### 4.2.3. Signature Algorithms

TLS 1.3 提供了两个扩展来表示数字签名里会用哪种签名算法。"signature_algorithms_cert" 扩展提供证书中的签名，"signature_algorithms" 扩展是从 TLS 1.2 出现的，提供 CertificateVerify 消息中的签名。证书中的秘钥也必须是所使用的签名算法的适当类型。这是 RSA 密钥和 PSS 签名的一个特殊问题，如下所述。如果没有 "signature_algorithms_cert" 扩展，"signature_algorithms" 扩展也提供证书中的出现的签名。客户端如果期望服务器通过证书证明其身份，必须发送 "signature_algorithms" 扩展。如果服务器通过证书证明了其身份，且客户端没有发送 "signature_algorithms" 扩展，那么服务器必须以 "missing_extension" alert 终止握手（见9.2）.

"signature_algorithms_cert" 扩展允许支持证书的不同算法集的实现明确表达自己的能力。TLS 1.2 实现应该也处理这些扩展。两种情况里具有相同策略的实现都可以使用 "signature_algorithms_cert" 扩展。

ClientHello 扩展中的 "extension_data" 字段包含 SignatureSchemeList 值：

```
      enum {
          /* RSASSA-PKCS1-v1_5 algorithms */
          rsa_pkcs1_sha256(0x0401),
          rsa_pkcs1_sha384(0x0501),
          rsa_pkcs1_sha512(0x0601),

          /* ECDSA algorithms */
          ecdsa_secp256r1_sha256(0x0403),
          ecdsa_secp384r1_sha384(0x0503),
          ecdsa_secp521r1_sha512(0x0603),

          /* RSASSA-PSS algorithms with public key OID rsaEncryption */
          rsa_pss_rsae_sha256(0x0804),
          rsa_pss_rsae_sha384(0x0805),
          rsa_pss_rsae_sha512(0x0806),

          /* EdDSA algorithms */
          ed25519(0x0807),
          ed448(0x0808),

          /* RSASSA-PSS algorithms with public key OID RSASSA-PSS */
          rsa_pss_pss_sha256(0x0809),
          rsa_pss_pss_sha384(0x080a),
          rsa_pss_pss_sha512(0x080b),

          /* Legacy algorithms */
          rsa_pkcs1_sha1(0x0201),
          ecdsa_sha1(0x0203),

          /* Reserved Code Points */
          private_use(0xFE00..0xFFFF),
          (0xFFFF)
      } SignatureScheme;

      struct {
          SignatureScheme supported_signature_algorithms<2..2^16-2>;
      } SignatureSchemeList;
```

注意：枚举名为 "SignatureScheme"，因为 TLS 1.2 中已经有一个 "SignatureAlgorithm" 类型，将被此替换。 我们在全文中使用术语 "signature algorithm"。

每个 SignatureScheme 值列出客户端愿意验证的单一签名算法。这些值以优先级的降序排列。注意，签名算法输入任意长度的消息，而不是摘要。传统上作用于摘要的算法应在 TLS 中定义，首先使用指定的哈希算法对输入进行哈希，然后照常处理。上面列出的码点组具有以下含义：

- RSASSA-PKCS1-v1_5 算法：表示使用 RSASSA-PKCS1-v1_5 [RFC3447] 签名算法和 [SHS] 中定义的相应哈希算法。这些值仅涉及出现在证书中的签名（见4.4.1.2），并且未定义用于签名TLS握手消息。但它们可能出现在 "signature_algorithms" 和 "signature_algorithms_cert" 以对 TLS 1.2 做前向兼容。
- ECDSA 算法：表示使用 ECDSA [ECDSA] 签名算法，ANSI X9.62 [X962] 和 FIPS 186-4 [DSS] 中定义了相应曲线，[SHS] 中定义了相应哈希算法。签名表示为 DER-encoded [X690] 的 ECDSA-Sig-Value 结构。
- RSASSA-PSS RSAE 算法：表示使用具有掩码生成功能 1 的 RSASSA-PSS [RFC8017] 签名算法。掩码生成函数中使用的摘要和签名的摘要都是在 [SHS] 中定义的相应哈希算法。Salt 的长度必须等于摘要输出的长度。如果在 X.509 证书中携带公钥，必须使用 rsaEncryption OID [RFC5280]。
- EdDSA 算法：表示使用[RFC8032] 中定义的 EdDSA 签名算法或其后续算法。注意，这些对应于 "PureEdDSA" 算法，而不是 "prehash" 变体。
- RSASSA-PSS PSS 算法：表示使用掩码生成函数 1 的 RSASSA-PSS [RFC8017] 签名算法 。掩码生成函数中使用的摘要和签名的摘要都是 [SHS] 中定义的相应哈希算法。如果 X.509 证书中携带了公钥，必须使用 RSASSA-PSS OID  [RFC5756]。当在证书签名中使用时，算法参数必须是DER编码的。如果对应的公钥参数存在，算法中的参数必须与公钥参数相同。
- 遗留算法：表示由于具有已知缺点被弃用的算法，特别是在本文中使用的用 RASSA-PKCS1-v1_5 的 RSA 或 ECDSA。这些值仅指出现在证书中的签名（见4.4.2.2），并且未被定义用于签名TLS握手消息，虽然它们可以出现在 "signature_algorithms" 和 "signature_algorithms_cert" 中以前向兼容 TLS 1.2。端点不应该协商这些算法，但允许这样做仅仅是为了前向兼容。 提供这些值的客户端必须将其列为最低优先级（在 SignatureSchemeList 中的所有其他算法之后列出）。 TLS 1.3 服务器不得提供 SHA-1 签名证书，除非没有此证书就不能生成有效的证书链（见4.4.2.2）。

自签名证书或作为信任锚的证书上的签名不会生效，因为它们开始了认证路径（见 [RFC5280] 3.2 节）。 开始认证路径的证书可以使用未在 "signature_algorithms" 扩展中通告的签名算法。

注意，TLS 1.2 定义了不同扩展。愿意协商 TLS 1.2 的 TLS 1.3 实现在协商该版本时必须按照 [RFC5246] 的要求进行操作。尤其是：

- TLS 1.2 ClientHello 可以删除此扩展。
- 在 TLS 1.2 中，扩展包含 hash/signature 对。 这些对被编码为两个字节，因此已分配 SignatureScheme 值已与 TLS 1.2 的编码对齐。 一些遗留对保留未分配。 这些算法自 TLS 1.3 起已弃用，不得提供或协商。 特别是，不得使用 MD5 [SLOTH]、SHA-224 和 DSA。
- ECDSA 签名方案与 TLS 1.2 的 ECDSA哈希/签名对一致。 然而，旧语义没有约束签名曲线。如果协商 TLS 1.2，则必须准备接受使用 "supported_groups" 扩展中通告的任何曲线的签名。
- 支持 RSASSA-PSS（在 TLS 1.3 中是强制的）的实现必须准备接受使用该方案的签名，即使协商的是 TLS 1.2。 在 TLS 1.2 中，RSASSA-PSS 与 RSA 密码套件一起使用。

#### 4.2.4. Certificate Authorities

"certificate_authorities" 扩展用于指示端点支持的证书授权机构（CA），并且接收端点应该使用它来指导证书选择。

"certificate_authorities" 扩展的内容由 CertificateAuthoritiesExtension 结构组成。

```
      opaque DistinguishedName<1..2^16-1>;

      struct {
          DistinguishedName authorities<3..2^16-1>;
      } CertificateAuthoritiesExtension;
```

- authorities: 可接受证书颁发机构的名称 [X501] 列表，以 DER 编码 [X690] 格式表示。这些名称为信任锚或从属 CA 指定所需名称，因此，该消息可以用于描述已知的信任锚以及期望的授权空间。

客户端可以在 ClientHello 消息中发送 "certificate_authorities" 扩展。 服务器可以在 CertificateRequest 消息中发送。

"trusted_ca_keys" 扩展 [RFC6066] 目的类似，但更为复杂，TLS 1.3 中不使用，但可能出现在先前 TLS 版本客户端的 ClientHello 消息中。

#### 4.2.5. OID Filters

"oid_filters" 扩展允许服务器提供一组希望客户端的证书匹配的 OID/value 对。如果是服务器提供的扩展，必须只能在 CertificateRequest 消息中发送。

```
      struct {
          opaque certificate_extension_oid<1..2^8-1>;
          opaque certificate_extension_values<0..2^16-1>;
      } OIDFilter;

      struct {
          OIDFilter filters<0..2^16-1>;
      } OIDFilterExtension;
```

filters:  证书扩展 OID 列表 [RFC5280] 和允许值，以 DER-encoded [X690] 格式呈现。一些证书扩展 OID 允许多个值（如 Extended Key Usage）。如果服务器包含了一个非空 filters 列表，回应的客户端证书必须包含所有客户端认识的指定扩展 OID。对于每个客户端认识的扩展 OID，所有指定值都必须出现在客户端证书中（但证书也可以有其他值）。然而，客户端必需忽略并跳过任何不认识的扩展 OID。如果客户端忽略一些要求的证书扩展 OID，并提供一个不满足请求的证书，服务器可以自行决定是在没有客户端认证的情况下继续连接，还是以 "unsupported_certificate" alert 放弃握手。任何给定的 OID 禁止在 filters 列表中出现多次。

PKIX RFCs 定义了各种各样的证书扩展 OID 和对应值。匹配证书扩展值不一定位相等，取决于类型。TLS 实现最好由它们的 PKI 库使用证书扩展 OID 执行证书选择。

本文为 [RFC5280] 中的两个标准证书扩展定义了匹配规则：

- 当请求中 assert 的所有 key usage 位也在 Key Usage 证书扩展中 assert，则证书中的 Key Usage 扩展与请求匹配。
- 当请求中所有秘钥用途 OID 也在 Extended Key Usage 证书扩展中找到，则证书中的 Extended Key Usage 扩展与请求匹配。

不同规范可以为其他证书扩展定义匹配规则。

#### 4.2.6. Post-Handshake Client Authentication

"post_handshake_auth" 扩展用于表示客户端愿意执行 post-handshake authentication（4.6.2节）。 服务器禁止向不提供此扩展的客户端发送 post-handshake CertificateRequest。 服务器不得发送此扩展。

```
      struct {} PostHandshakeAuth;
```

"post_handshake_auth" 扩展的 "extension_data" 字段为零长度。

#### 4.2.7. Supported Groups

当客户端发送 "supported_groups" 扩展时，表示客户端支持的用于密钥交换的命名组（named groups），顺序从最优选到最不优选。

注意：在 TLS 1.3 之前的 TLS 版本中，此扩展名称为 "elliptic_curves"，并且只包含椭圆曲线组（见 [RFC4492] 和 [RFC7919]）。 此扩展也用于协商 ECDSA 曲线。签名算法现在独立协商（见4.2.3节）。

此扩展的 "extension_data" 字段包含 "NamedGroupList" 值：

```
      enum {

          /* Elliptic Curve Groups (ECDHE) */
          secp256r1(0x0017), secp384r1(0x0018), secp521r1(0x0019),
          x25519(0x001D), x448(0x001E),

          /* Finite Field Groups (DHE) */
          ffdhe2048(0x0100), ffdhe3072(0x0101), ffdhe4096(0x0102),
          ffdhe6144(0x0103), ffdhe8192(0x0104),

          /* Reserved Code Points */
          ffdhe_private_use(0x01FC..0x01FF),
          ecdhe_private_use(0xFE00..0xFEFF),
          (0xFFFF)
      } NamedGroup;

      struct {
          NamedGroup named_group_list<2..2^16-1>;
      } NamedGroupList;
```

- Elliptic Curve Groups（ECDHE）：表示支持对应的命名曲线，在 FIPS 186-4 [DSS] 或 [RFC7748] 中定义。值 0xFE00 到 0xFEFF 保留供私用。
- Finite Field Groups（DHE）：表示支持相应的有限域组，在 [RFC7919] 中定义。值 0x01FC 至 0x01FF 保留供私用。

named_group_list 中的条目根据发送者的偏好排序（最前面是最偏好的选择）。

从 TLS 1.3 开始，服务器可以向客户端发送 "supported_groups" 扩展。客户端不能在成功完成握手之前对 "supported_groups" 中的任何信息采取行动，但可以使用从成功完成的握手中获得的信息来更改后续连接中 "key_share" 扩展中使用的组。如果服务器有一个更偏向于用 "key_share" 扩展中信息的一个组，但仍然愿意接受 ClientHello，它应该发送 "supported_groups" 来更新客户端的偏好视图。此扩展应包含服务器支持的所有组，无论当前客户端是否支持。

#### 4.2.8. Key Share

"key_share" 扩展包含端点的加密参数。

客户端可以发送空的 client_shares 向量，以一个额外的往返代价从服务器请求组选择（见4.1.4）。

```
      struct {
          NamedGroup group;
          opaque key_exchange<1..2^16-1>;
      } KeyShareEntry;
```

- group：要交换的密钥的命名组。
- key_exchange: 密钥交换信息。此字段的内容由指定组及其相应的定义确定。有限域 Diffie-Hellman [DH76] 参数在 4.2.8.1 中描述。椭圆曲线 Diffie-Hellman 参数在 4.2.8.2 中描述。

在 ClientHello 消息中，这个扩展的 "extension_data" 字段包含一个 "KeyShareClientHello" 值：

```
      struct {
          KeyShareEntry client_shares<0..2^16-1>;
      } KeyShareClientHello;
```

- client_shares：一组提供的 KeyShareEntry 值，以客户端偏好降序排列。

如果客户端请求 HelloRetryRequest，则此向量可以为空。 每个 KeyShareEntry 值必须对应于在 "supported_groups" 扩展中提供的组，并且必须以相同的顺序排列。然而，值可以是 "supported_groups" 扩展的非连续子集，并且可以省略最优选的组。如果最优选的组是新的，并且没有足够的空间更高效地预生成共享秘钥，则可能出现这种情况。

客户端可以提供很多 KeyShareEntry 值，数量跟提供的支持的组一样，每个值表示一组密钥交换参数。例如，客户端可能为几个椭圆曲线或多个 FFDHE 组提供共享秘钥。每个 KeyShareEntry 的 key_exchange 值必须独立生成。 客户不得为同一组提供多个 KeyShareEntry 值。 客户端不得为 "supported_groups" 扩展中未列出的组提供 KeyShareEntry 值。服务器可能会检查违反了这些规则的行为，如果违反了则使用 "illegal_parameter" alert 来中止握手。

在 HelloRetryRequest 消息中，此扩展的 "extension_data" 字段包含一个 KeyShareHelloRetryRequest 值：

```
      struct {
          NamedGroup selected_group;
      } KeyShareHelloRetryRequest;
```

- selected_group：服务器打算协商的都支持的组，准备为期请求重试 ClientHello/KeyShare。

在 HelloRetryRequest 中接收到此扩展时，客户端必须验证：

1. selected_group 字段对应于在原始 ClientHello "supported_groups" 扩展中提供的组；
2. selected_group 字段与原始 ClientHello 中的 "key_share" 扩展中提供的组不对应。

如果这些检查中的任一个失败，则客户端必须用 "illegal_parameter" alert 来中止握手。 否则，当发送新的 ClientHello 时，客户端必须用仅包含新 KeyShareEntry 的扩展替换原来的 "key_share" 扩展，新的扩展对应于触发 HelloRetryRequest 的 selected_group 字段中指示的组。

在 ServerHello 中，此扩展的 "extension_data" 字段包含一个 KeyShareServerHello 值：

```
      struct {
          KeyShareEntry server_share;
      } KeyShareServerHello;
```

- server_share：一个跟客户端共享的在同一个组中的 KeyShareEntry 值。

如果使用 (EC)DHE 密钥协商，服务器在 ServerHello 中只提供一个 KeyShareEntry。 该值必须与服务器为协商密钥交换选择的客户端提供的 KeyShareEntry 值在同一组。服务器不得为 "supported_groups" 扩展中指定的任何组发送 KeyShareEntry，并且在使用 "psk_ke" PskKeyExchangeMode 时不得发送 KeyShareEntry。如果使用 (EC)DHE 密钥协商，并且客户端收到包含 "key_share" 扩展的 HelloRetryRequest，客户端必须验证 ServerHello 中选择的 NamedGroup 与 HelloRetryRequest 中的相同，否则必须以 "illegal_parameter" alert 中止握手。

##### 4.2.8.1. Diffie-Hellman Parameters

客户端和服务器的 Diffie-Hellman [DH76] 参数都编码在 KeyShare 结构中的 KeyShareEntry 的 opaque key_exchange 字段中。opaque 值包含指定组（参见 [RFC7919] 的组定义）的 Diffie-Hellman 公共值（Y=g^X mod p），编码为大端字节序整数，并用填充 0 到左侧至 p 字节。

注意：对于给定的 Diffie-Hellman 组，填充使所有公钥长度相同。

对端应该通过确保 `1 < Y < p-1` 来验证对方的公钥 Y。此检查确保对端在行为正常，并且不强制本地系统进入小型组。

##### 4.2.8.2. ECDHE Parameters

客户端和服务器的 ECDHE 参数都编码在 KeyShare 结构中 KeyShareEntry 的 opaque key_exchange 字段中。

对于 secp256r1、secp384r1、secp521r1，是以下结构体的序列化值：

```
      struct {
          uint8 legacy_form = 4;
          opaque X[coordinate_length];
          opaque Y[coordinate_length];
      } UncompressedPointRepresentation;
```

X 和 Y 分别是 X 和 Y 值的网络序二进制表示。由于没有内部长度标记，因此每个数字占用曲线参数隐含的字节数。 对于 P-256，这意味着 X 和 Y 分别使用 32 字节，如果需要，则在左侧填充零。对于 P-384，分别占用 48 字节，对于 P-521，各占用 66 字节。

对于曲线 secp256r1、secp384r1、secp521r1，对端必须通过确保该点是椭圆曲线上的有效点来验证彼此的公共值 Y。相应的验证程序在 [X962] 的 4.3.7 节中定义，或者在 [KEYAGREEMENT] 的 5.6.2.6 节中定义。 该过程由三个步骤组成：

1. 验证 Q 不是无穷大点 `(O)`，
2. 验证 `Q = (x,y)` 中 x 和 y 两个整数都在正确的间隔，
3. 确保 `(x,y)` 是椭圆曲线方程的正确解。对于这些曲线，实现者不需要验证正确子组中的成员。

对于 X25519 和 X448，公共值的内容是 [RFC7748] 中定义的相应功能的字节串输入和输出：X25519 是 32 字节，X448 是 56 字节。

注意：1.3 之前版本的 TLS 允许 point format 协商；TLS 1.3 删除了此功能，从而使每个曲线有一个 point format。

#### 4.2.9. Pre-Shared Key Exchange Modes

为了使用 PSK，客户端还必须发送一个 "psk_key_exchange_modes" 扩展。 此扩展的意思是客户端仅支持使用这些模式的 PSK，这限制了在这个 ClientHello 中提供的 PSK 的使用以及服务器通过 NewSessionTicket 提供的 PSK 的使用。

如果客户端提供了一个 "pre_shared_key" 扩展，也必须提供一个 "psk_key_exchange_modes" 扩展。 如果客户端提供了 "pre_shared_key"，但没提供 "psk_key_exchange_modes"，服务器必须中止握手。服务器不得选择客户端未给出的密钥交换模式。此扩展还限制 PSK 恢复使用的模式。服务器不应发送与通告模式不兼容的 NewSessionTicket，但是如果服务器这样做，影响将只是客户端尝试恢复会失败。

服务器不得发送 "psk_key_exchange_modes" 扩展。

```
      enum { psk_ke(0), psk_dhe_ke(1), (255) } PskKeyExchangeMode;

      struct {
          PskKeyExchangeMode ke_modes<1..255>;
      } PskKeyExchangeModes;
```

- psk_ke: PSK-only 密钥建立。 在这种模式下，服务器不得提供 "key_share" 值。
- psk_dhe_ke: 使用 (EC)DHE 密钥建立的 PSK。在这种模式下，客户端和服务器必须提供 "key_share" 值见 4.2.8 节）。

任何将来分配的值必须确保传输协议消息明确识别服务器选择的模式。目前这由 ServerHello 中的 "key_share" 表示。

#### 4.2.10. Early Data Indication

当使用 PSK 时，客户端可以在第一个消息中发送应用数据。 如果客户端选择这样做，就必须提供 "early_data" 扩展和 "pre_shared_key" 扩展。

此扩展的 "extension_data" 字段包含 "EarlyDataIndication" 值。

```
      struct {} Empty;

      struct {
          select (Handshake.msg_type) {
              case new_session_ticket:   uint32 max_early_data_size;
              case client_hello:         Empty;
              case encrypted_extensions: Empty;
          };
      } EarlyDataIndication;
```

max_early_data_size 字段的使用见 4.6.1 节。

0-RTT 参数（版本号、对称密码套件、ALPN 协议 [RFC7301] 等）是使用 PSK 的关联值。对于外部配置的 PSK，关联值与秘钥一起配置。对于通过 NewSessionTicket 消息确定的 PSK，关联值是在确定 PSK 的连接里协商的。用于加密早期数据的 PSK 必须是客户端 "pre_shared_key" 扩展中列出的第一个 PSK。

对于通过 NewSessionTicket 提供的 PSK，服务器必须验证所选 PSK 身份的 ticket 生存期 (PskIdentity.obfuscated_ticket_age mod 2 ^ 32 - ticket_age_add) 在从 ticket 开始使用的小时间范围内（见第 8 章）。如果不是，服务器应该继续握手，但拒绝 0-RTT，并且不应该采取任何假定该 ClientHello 是全新的其他操作。

在第一个消息中发送的 0-RTT 消息与其他消息（握手和应用程序数据）中发送的相应消息具有相同（加密）的内容类型，但受到不同密钥的保护。在收到服务器的 Finished 消息后，如果服务器已接收到早期数据，则会发送 EndOfEarlyData 消息以指示密钥更改。该消息使用 0-RTT 流量密钥加密。

接收 "early_data" 扩展的服务器必须以三种方式之一进行操作：

- 忽略扩展并返回常规的 1-RTT 响应。然后，服务器忽略早期数据并尝试使用握手流量秘钥解密收到的数据，忽略解密失败的数据（直到配置的 max_early_data_size ）。一旦数据成功解密，则被当做客户端第二个消息的开始，服务端当做普通 1-RTT 握手继续处理。
- 通过响应 HelloRetryRequest 请求客户端发送另一个 ClientHello。 客户端不得在其后续 ClientHello 中包含 "early_data" 扩展。然后，服务器跳过外部内容类型 "application_data" 的所有记录（表示被加密）来忽略 early data，直到配置的 max_early_data_size 长度。
- 在 EncryptedExtensions 中返回自己的 "early_data" 扩展，表示它打算处理 early data。 服务器不可能只接受 early data 消息的一部分。 即使服务器发送接收 early data 的消息，但是实际的 early data 本身可能已经在服务器生成此消息时发送了。

为了接受 early data，服务器必须先接受了 PSK 密码套件并且选择了客户端的 "pre_shared_key" 扩展中提供的第一个密钥。此外，必须验证以下值与选择的 PSK 相关联：

- TLS 版本号和加密套件。
- 选择的密码套件
- 选择的 ALPN 协议 [RFC7301]（如果有）。

这些要求是使用相关 PSK 执行 1-RTT 握手的要求的超集。对于外部配置的 PSK，关联值与秘钥一起提供。对于通过 NewSessionTicket 消息确定的 PSK，关联值是通过连接协商的。

未来的扩展必须定义它们与 0-RTT 的交互。

如果任一上述检查失败，服务器不得使用扩展进行响应，并且必须使用上面前两种机制之一丢弃所有第一个报文中的数据（回退到 1-RTT 或 2-RTT）。如果客户端尝试进行 0-RTT 握手，但是服务器拒绝，则服务器通常没有 0-RTT 保护密钥，必须使用试用解密（使用 1-RTT 握手密钥或在 HelloRetryRequest 的情况下通过查找明文 ClientHello）找到第一个非 0-RTT 消息。

如果服务器选择接受 "early_data" 扩展，那么在处理 early data 时，它必须遵守与所有记录相同的错误处理要求。 具体来说，如果服务器在接受的 "early_data" 扩展后无法解密任何 0-RTT 记录，则必须根据 5.2 节使用 "bad_record_mac" alert 终止连接。

如果服务器拒绝 "early_data" 扩展，则客户端应用程序可以在握手完成后重新发送 early data。 请注意，early data 的自动重新传输可能导致关于连接状态不正确的假设。例如，当协商的连接选择与早期数据不同的 ALPN 协议时，应用程序可能需要构建不同的消息。 类似地，如果早期数据假定任何连接状态，则握手完成后可能发送错误。

TLS 实现不应自动重新发送 early data；应用程序能够更好地决定重新传输是否合适。除非协商的连接选择相同的 ALPN 协议，否则 TLS 实现不得自动重新发送 early data。

#### 4.2.11. Pre-Shared Key Extension

"pre_shared_key" 扩展用于协商 PSK 密钥建立相关联握手使用的预共享密钥身份。

此扩展的 "extension_data" 字段包含 "PreSharedKeyExtension" 值：

```
      struct {
          opaque identity<1..2^16-1>;
          uint32 obfuscated_ticket_age;
      } PskIdentity;

      opaque PskBinderEntry<32..255>;

      struct {
          PskIdentity identities<7..2^16-1>;
          PskBinderEntry binders<33..2^16-1>;
      } OfferedPsks;

      struct {
          select (Handshake.msg_type) {
              case client_hello: OfferedPsks;
              case server_hello: uint16 selected_identity;
          };
      } PreSharedKeyExtension;
```

- Identity：秘钥标签。例如，附录 B.3.4 中定义的 ticket，或外部配置的预共享密钥标签。
- obfuscated_ticket_age：密钥生存时间的混淆版本。4.2.11.1 节描述了怎样通过 NewSessionTicket 消息建立的标识生成此值。对于外部配置的标识，应该使用 0 的 obfuscated_ticket_age，服务器必须忽略该值。
- Identities：客户端想要与服务器协商的标识列表。如果与 "early_data" 扩展一起发送（见4.2.10节），第一个标识是用于 0-RTT 数据。
- Binders：一系列 HMAC 值，每个标识值一个，并且以相同顺序排列，计算过程如下所述。
- selected_identity：服务器选择的标识，以客户端列表中的标识表示为 (0-based) 的索引。

每个 PSK 与一个哈希算法相对应。对于通过 ticket 机制建立的 PSK（4.6.1节），是建立 ticket 的连接上的 KDF Hash 算法。对于外部配置的 PSK，当 PSK 建立时，必须设置哈希算法，或者没指定算法时默认为 SHA-256。 服务器必须确保选择兼容的 PSK（如果有的话）和密码套件。

TLS 1.3 之前的版本中，服务器名字标识（Server Name Identification，SNI）值需要与 session 相对应（[RFC6066] 中第 3 章），服务器需要强制 SNI 值与握手恢复中指定的匹配 session 相对应。然而，现实中使用提供的 SNI 值中的哪一个，实现并不一致，这导致客户端事实上强制执行一致性要求。在 TLS 1.3 中，SNI 值总是在恢复握手中指定，服务器没必要将 SNI 值与 ticket 关联。但客户端应该与 PSK 一起存储 SNI 来满足 4.6.1 中的要求。

实现需注意：当会话恢复是 PSK 主要应用场景时，实现 PSK/密码套件匹配要求的最直接方法是先协商密码套件，然后排除任何不兼容的 PSK。任何未知的 PSK（如不在 PSK 数据库中或者以未知秘钥加密）应该忽略。如果没有可接受的 PSK，服务器可能的话应该执行 non-PSK 握手。如果前向兼容很重要，客户端的外部配置 PSK 应该决定密码套件选择。

在接受 PSK 密钥建立之前，服务器务必验证相应的 binder 值（见下面的 4.2.11.2 节）。如果此值不存在或未验证，则服务器必须中止握手。服务器不应该尝试验证多个 binder，而是应该选择单个 PSK 并且仅验证对应于该 PSK 的 binder。该要求的安全原理见 8.2 节和附录 E.6。为了接受 PSK 密钥建立，服务器发送 "pre_shared_key" 扩展来指示选择的标识。

客户端必须验证服务器的 selected_identity 是否在客户端提供的范围内，服务器选择了包含与 PSK 关联哈希的加密套件，并且如果 ClientHello "psk_key_exchange_modes" 扩展需要，服务器应该发送 "key_share" 扩展。如果这些值不一致，客户端必须使用 "illegal_parameter" alert 中止握手。

如果服务器提供了 "early_data" 扩展，客户端必须验证服务器的 selected_identity 是否为 0。如果返回任何其他值，客户端必须使用 "illegal_parameter" alert 中止握手。

"pre_shared_key" 扩展必须是 ClientHello 中的最后一个扩展（这有助于如下所述的实现）。 服务器必须检查它是最后一个扩展，否则以 "illegal_parameter" alert 握手失败。

##### 4.2.11.1. Ticket Age

客户端对 ticket 生存时间的看法是收到 NewSessionTicket 消息后的时间。客户端不得尝试使用 ticket 生存时间大于 "ticket_lifetime" 值的 ticket。每个 PskIdentity 的 "obfuscated_ticket_age" 字段包含了 ticket 生存时间的混淆版本，以毫秒为单位，加上票据中包含的 "ticket_age_add" 值（见4.6.1节），mod 2^32。这个添加可以防止被动观察者关联连接，除非票据被重复使用。请注意，NewSessionTicket 消息中的 "ticket_lifetime" 字段的单位是秒，但 "obfuscated_ticket_age" 的单位是毫秒。因为票据生存期被限制在一周之内，32 位足以代表任何可信的生存期，即使是以毫秒为单位。

##### 4.2.11.2. PSK Binder

PSK binder 值绑定了 PSK 和当前握手，并且绑定生成 PSK 的握手（如果通过 NewSessionTicket 消息）和当前握手。binder 列表中的每个条目被计算为直到（并包括）PreSharedKeyExtension.identities 字段的 ClientHello 部分（包括握手报头）上的 HMAC（transcript hash，见4.4.1）。也就是说，它包括所有 ClientHello，但不包括 bingder 列表本身。消息的长度字段（包括总长度，扩展块长度和 "pre_shared_key" 扩展长度）都是按照正确长度的 binder 来设置的。

PskBinderEntry 以 Finished 消息（4.4.4节）相同方式计算，但是 BaseKey 是提供相应 PSK 的密钥计划表派生的 binder_key（见第 7.1 节）。

如果握手包括 HelloRetryRequest，则初始 ClientHello 和 HelloRetryRequest 与新的 ClientHello 一起被包括在副本中。例如，如果客户端发送 ClientHello1，则其 binder 将如下计算：

```
      Transcript-Hash(Truncate(ClientHello1))
```

其中 Truncate() 从 ClientHello 中移除 binder 列表。

如果服务器响应 HelloRetryRequest，然后客户端然后发送 ClientHello2，其 binder 将通过计算：

```
      Transcript-Hash(ClientHello1,
                      HelloRetryRequest,
                      Truncate(ClientHello2))
```

完整的 ClientHello1/ClientHello2 包括在所有其他握手哈希计算中。注意在第一次发送中，Truncate(ClientHello1) 是直接哈希的，但在第二次发送中 ClientHello1 先哈希，然后作以 "message_hash" 消息注入，如 4.4.1 所述。

##### 4.2.11.3. Processing Order

客户端接收服务器的 Finished 之前都可以 "stream" 0-RTT 数据，然后发送 EndOfEarlyData 消息，接着是剩下的握手。为了避免死锁，当接受 "early_data" 时，服务器必须处理客户端的 ClientHello，然后立即发送消息，而不是在发送 ServerHello 前等待客户端的 EndOfEarlyData 消息。

### 4.3. Server Parameters

#### 4.3.1. Encrypted Extensions

#### 4.3.2. Certificate Request

### 4.4. Authentication Messages

#### 4.4.1. The Transcript Hash

#### 4.4.2. Certificate

##### 4.4.2.1. OCSP Status and SCT Extensions

##### 4.4.2.2. Server Certificate Selection

##### 4.4.2.3. Client Certificate Selection

##### 4.4.2.4. Receiving a Certificate Message

#### 4.4.3. Certificate Verify

#### 4.4.4. Finished

### 4.5. End of Early Data

### 4.6. Post-Handshake Messages

#### 4.6.1. New Session Ticket Message

#### 4.6.2. Post-Handshake Authentication

#### 4.6.3. Key and Initialization Vector Update

## 5. Record Protocol

### 5.1. Record Layer

### 5.2. Record Payload Protection

### 5.3. Per-Record Nonce

### 5.4. Record Padding

### 5.5. Limits on Key Usage

## 6. Alert Protocol

### 6.1. Closure Alerts

### 6.2. Error Alerts

## 7. Cryptographic Computations

### 7.1. Key Schedule

### 7.2. Updating Traffic Secrets

### 7.3. Traffic Key Calculation

### 7.4. (EC)DHE Shared Secret Calculation

#### 7.4.1. Finite Field Diffie-Hellman

#### 7.4.2. Elliptic Curve Diffie-Hellman

### 7.5. Exporters

## 8. 0-RTT and Anti-Replay

### 8.1. Single-Use Tickets

### 8.2. Client Hello Recording

### 8.3. Freshness Checks

## 9. Compliance Requirements

### 9.1. Mandatory-to-Implement Cipher Suites

### 9.2. Mandatory-to-Implement Extensions

### 9.3. Protocol Invariants

## 10. Security Considerations

## 11. IANA Considerations

## 12. References

## Appendix A. State Machine

### A.1. Client

### A.2. Server

## Appendix B. Protocol Data Structures and Constant Values

### B.1. Record Layer

### B.2. Alert Messages

### B.3. Handshake Protocol

#### B.3.1. Key Exchange Messages

##### B.3.1.1. Version Extension

##### B.3.1.2. Cookie Extension

##### B.3.1.3. Signature Algorithm Extension

##### B.3.1.4. Supported Groups Extension

#### B.3.2. Server Parameters Messages

#### B.3.3. Authentication Messages

#### B.3.4. Ticket Establishment

#### B.3.5. Updating Keys

### B.4. Cipher Suites

## Appendix C. Implementation Notes

### C.1. Random Number Generation and Seeding

### C.2. Certificates and Authentication

### C.3. Implementation Pitfalls

### C.4. Client Tracking Prevention

### C.5. Unauthenticated Operation

## Appendix D. Backward Compatibility

### D.1. Negotiating with an Older Server

### D.2. Negotiating with an Older Client

### D.3. 0-RTT Backward Compatibility

### D.4. Middlebox Compatibility Mode

### D.5. Security Restrictions Related to Backward Compatibility

## Appendix E. Overview of Security Properties

### E.1. Handshake

#### E.1.1. Key Derivation and HKDF

#### E.1.2. Client Authentication

#### E.1.3. 0-RTT

#### E.1.4. Exporter Independence

#### E.1.5. Post-Compromise Security

#### E.1.6. External References

### E.2. Record Layer

#### E.2.1. External References

### E.3. Traffic Analysis

### E.4. Side-Channel Attacks

### E.5. Replay Attacks on 0-RTT

#### E.5.1. Replay and Exporters

### E.6. PSK Identity Exposure

### E.7. Sharing PSKs

### E.8. Attacks on Static RSA



