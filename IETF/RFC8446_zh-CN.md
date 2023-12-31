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

来自服务器的接下来两条消息 EncryptedExtensions 和 CertificateRequest，包含来自决定其余握手的服务器的信息。这些消息使用从 server_handshake_traffic_secret 派生的密钥加密。

#### 4.3.1. Encrypted Extensions

在所有握手中，服务器必须在 ServerHello 消息之后立即发送 EncryptedExtensions 消息。这是使用 server_handshake_traffic_secret 派生密钥加密的第一条消息。

EncryptedExtensions 消息包含可以被保护的扩展，即不需要建立加密上下文但不与各个证书相关联的扩展。客户端必须检查 EncryptedExtensions 是否存在任何禁止的扩展，如果发现任何此类扩展，必须用 "illegal_parameter" alert 中止握手。

这个消息的结构：

```
   Structure of this message:

      struct {
          Extension extensions<0..2^16-1>;
      } EncryptedExtensions;
```

- Extensions：扩展列表。更多信息见 4.2 节的表格。

#### 4.3.2. Certificate Request

使用证书进行身份验证的服务器可以选择向客户端请求证书。如果发送了这个消息，必须在 EncryptedExtensions 之后。

此消息的结构：

```
      struct {
          opaque certificate_request_context<0..2^8-1>;
          Extension extensions<2..2^16-1>;
      } CertificateRequest;
```

- certificate_request_context：一个不透明的字符串，用于表示证书请求，并在客户端的证书消息中回复。 certificate_request_context 在此连接的范围内必须是唯一的（从而防止客户端 CertificateVerify 消息的重放）。该字段应为零长度，除非用于 4.6.2 节中描述的 post-handshake 身份验证交互。当请求 post-handshake 认证时，服务器应该使上下文对客户端不可预测（比如随机生成），从而防止临时访问客户端私钥的攻击者预先计算出有效的 CertificateVerify 消息。
- Extensions：描述正在请求的证书参数的扩展集。必须指定 "signature_algorithms" 扩展名，如果此消息定义了其他扩展也可以选择性包含。客户端必须忽略不认识的扩展。

在 TLS 的先前版本中，CertificateRequest 消息携带服务器接受的签名算法和证书授权列表。在 TLS 1.3 中，前者通过发送 "signature_algorithms" 和可选的 "signature_algorithms_cert" 扩展来表示。后者通过发送 "certificate_authorities" 扩展来表示（见4.2.4节）。

使用 PSK 认证的服务器不得在主握手中发送 CertificateRequest 消息，虽然可以在 post-handshake 认证中发送（见4.6.2），但客户端要已经发送了 "post_handshake_auth" 扩展（见 4.2.6）。

### 4.4. Authentication Messages

如第 2 章所述，TLS 通常使用一组消息来进行身份验证，密钥确认和握手完整性：Certificate，CertificateVerify 和 Finished。（PSK binder 也以类似的方式执行密钥确认。） 这三个消息总是在握手中的最后发送。Certificate 和 CertificateVerify 消息仅在某些情况下发送，如下所定义。Finished 的消息总是作为 Authentication Block 的一部分发送。这些消息使用 [sender]_handshake_traffic_secret 派生的密钥加密。

认证消息的计算统一采用以下输入：

- 要使用的证书和签名密钥。
- 一个握手上下文，由要包含在 transcript 哈希中的一组消息组成。
- 用于计算 MAC 密钥的基本密钥。

基于这些输入，消息包含：

- Certificate：用于认证的证书和链中的任何支持证书。请注意，基于证书的客户端身份验证在 PSK 握手流中不可用（包括0-RTT）。
- CertificateVerify：Transcript-Hash 值的签名（握手上下文+证书）。
- Finished：Transcript-Hash 值的 MAC（握手上下文+证书+证书验证），使用 Base Key 派生的 MAC 秘钥。

下表定义了每个场景的握手上下文和 MAC 基本密钥：

```
   +-----------+-------------------------+-----------------------------+
   | Mode      | Handshake Context       | Base Key                    |
   +-----------+-------------------------+-----------------------------+
   | Server    | ClientHello ... later   | server_handshake_traffic_   |
   |           | of EncryptedExtensions/ | secret                      |
   |           | CertificateRequest      |                             |
   |           |                         |                             |
   | Client    | ClientHello ... later   | client_handshake_traffic_   |
   |           | of server               | secret                      |
   |           | Finished/EndOfEarlyData |                             |
   |           |                         |                             |
   | Post-     | ClientHello ... client  | client_application_traffic_ |
   | Handshake | Finished +              | secret_N                    |
   |           | CertificateRequest      |                             |
   +-----------+-------------------------+-----------------------------+
```

#### 4.4.1. The Transcript Hash

TLS 的很多加密算法使用 transcript hash。这个值是通过对每个包含的握手消息进行哈希串联计算出来的，包括携带握手消息类型和长度字段的握手消息头，但不包括记录层头。 即：

```
    Transcript-Hash(M1, M2, ... Mn) = Hash(M1 || M2 || ... || Mn)
```

作为这个一般规则的例外，当服务器用 HelloRetryRequest 响应一个 ClientHello 时，ClientHello1 的值会被一个特殊的合成握手消息代替，握手类型为 "message_hash"，包含 Hash(ClientHello1)。 即：

```
  Transcript-Hash(ClientHello1, HelloRetryRequest, ... Mn) =
      Hash(message_hash ||        /* Handshake type */
           00 00 Hash.length  ||  /* Handshake message length (bytes) */
           Hash(ClientHello1) ||  /* Hash of ClientHello1 */
           HelloRetryRequest  || ... || Mn)
```

这种结构的原因是允许服务器只在 cookie 中存储 ClientHello1 的哈希值，而不用导出整个中间哈希状态，从而进行无状态的 HelloRetryRequest（见 4.2.2 节）。

为了具体化，transcript hash 总是取自以下握手消息序列，从第一个 ClientHello 开始，只包括那些被发送的消息：

- ClientHello
- HelloRetryRequest
- ClientHello
- ServerHello
- EncryptedExtensions
- server CertificateRequest
- server Certificate
- server CertificateVerify
- server Finished
- EndOfEarlyData
- client Certificate
- client CertificateVerify
- client Finished

一般来说，实现者可以根据协商好的哈希值保持一个运行中的 transcript hash 值来实现 transcript。 但请注意，后续的 post-handshake 认证不包括，只包含到主握手结束的消息。

#### 4.4.2. Certificate

该消息将端点的证书链传递给对端。

每当约定的密钥交换方法使用证书进行认证时，服务器必须发送一条Certificate消息（这包括本文中定义的所有密钥交换方法，PSK 除外）。

只有服务器通过 CertificateRequest 消息请求客户端认证时，客户端才必须发送 Certificate 消息（4.3.2节）。 如果服务器请求客户端认证，但客户端没有合适的证书，则必须发送一个不包含证书的 Certificate 消息（即 "certificate_list" 字段长度为 0）。 无论证书消息是否为空，都必须发送一个 Finished 消息。

```
      enum {
          X509(0),
          RawPublicKey(2),
          (255)
      } CertificateType;

      struct {
          select (certificate_type) {
              case RawPublicKey:
                /* From RFC 7250 ASN.1_subjectPublicKeyInfo */
                opaque ASN1_subjectPublicKeyInfo<1..2^24-1>;

              case X509:
                opaque cert_data<1..2^24-1>;
          };
          Extension extensions<0..2^16-1>;
      } CertificateEntry;

      struct {
          opaque certificate_request_context<0..2^8-1>;
          CertificateEntry certificate_list<0..2^24-1>;
      } Certificate;
```

- certificate_request_context：如果该消息是对 CertificateRequest 的回应，则为该消息中的 certificate_request_context 的值。 否则（在服务器认证的情况下），该字段长度为零。
- certificate_list：CertificateEntry 结构的序列(链)，每个结构包含一个证书和一组扩展。
- extensions：CertificateEntry 的一组扩展值。扩展格式在 4.2 节中定义。目前服务器证书的有效扩展包括 OCSP Status 扩展 [RFC6066] 和 SignedCertificateTimestamp 扩展 [RFC6962]；未来也可以为该消息定义扩展。服务器的 Certificate 消息中的扩展必须与 ClientHello 消息的扩展相对应。 客户端的 Certificate 消息中的扩展必须与服务器的 CertificateRequest 消息中的扩展相对应。 如果一个扩展用于整个链，它应该包含在第一个 CertificateEntry 中。

如果对应的证书类型扩展（server_certificate_type 或 client_certificate_type）没有在 EncryptedExtensions 中协商，或者协商为 X.509 证书类型，那么每个 CertificateEntry 都包含一个 DER 编码的 X.509 证书。发送者的证书必须在列表中的第一个 CertificateEntry 中。 后面的每个证书都应该直接认证紧邻它前面的证书。因为证书验证要求 trust anchors 是独立分发的，所以指定 trust anchors 的证书可以从链中省略，前提是支持的对端已知拥有任何省略的证书。

注意：在 TLS 1.3 之前，certificate_list 的排序要求每个证书都要对紧接在它前面的证书进行认证；但是，一些实现允许一些灵活性。服务器有时会为了过渡目的同时发送一个当前的和过时的中间文件，还有一些只是配置不正确，但这些情况还是可以被正确验证。为了实现最大的兼容性，所有的实现都应该准备好处理任何 TLS 版本中潜在的无关证书和任意的顺序，但最终实体证书必须放在第一位。

如果协商为 RawPublicKey 证书类型，那么 certificate_list 必须不超过一个 CertificateEntry，其中包含 [RFC7250] 第 3 节中定义的 ASN1_subjectPublicKeyInfo 值。

OpenPGP 证书类型 [RFC6091] 不得用于 TLS 1.3。

服务器的 certificate_list 必须是非空的。如果客户端没有合适的证书来响应服务器的认证请求，客户端将发送一个空的证书列表。

##### 4.4.2.1. OCSP Status and SCT Extensions

[RFC6066] 和 [RFC6961] 提供了扩展来协商服务器向客户端发送 OCSP 响应。 在 TLS 1.2 及之前版本中，服务器用一个空的扩展来表示协商这个扩展，OCSP 信息携带在 CertificateStatus 消息中。 在 TLS 1.3 中，服务器的 OCSP 信息在包含相关证书的 CertificateEntry 中的扩展中。具体来说，服务器的 "status_request" 扩展的主体必须是 [RFC6066] 中定义的 CertificateStatus 结构，其解释在 [RFC6960] 中定义。

注意：status_request_v2 扩展 [RFC6961] 已经废弃。TLS 1.3 服务器在处理 ClientHello 消息时，不能根据 status_request_v2 的存在或其中的信息采取行动，尤其是不能在 EncryptedExtensions、CertificateRequest 或 Certificate 消息中发送此扩展。TLS 1.3 服务器必须能够处理包含 status_request_v2 的 ClientHello 消息，因为使用早期协议版本的客户端发送可能想要使用。

服务器可以通过在 CertificateRequest 消息中发送一个空的 "status_request" 扩展，要求客户机提供 OCSP 响应和它的证书。如果客户端选择发送 OCSP 响应，其 "status_request" 扩展的主体必须是 [RFC6066] 中定义的 CertificateStatus 结构。
同样，[RFC6962] 提供了一种机制，在 TLS 1.2 及以下版本中，服务器可以将签名证书时间戳（SCT，Signed Certificate Timestamp）作为 ServerHello 中的扩展发送。 在 TLS 1.3 中，服务器的 SCT 信息在 CertificateEntry 的扩展中。

##### 4.4.2.2. Server Certificate Selection

服务器发送的证书遵循以下规则：

- 证书类型必须是X.509v3[RFC5280]，除非另有显式协商（如[RFC7250]）。
- 服务器的终端实体证书的公钥（和相关的限制）必须与客户的 "signature_algorithms" 扩展（目前是 RSA、ECDSA 或 EdDSA）中选择的认证算法兼容。
- 证书必须允许使用秘钥进行签名（即如果有 Key Usage 扩展，digitalSignature 位必须被设置），签名方案在客户端的 signature_algorithms 或 signature_algorithms_cert 扩展中指明（见4.2.3节）。
- server_name [RFC6066] 和 certificate_authorities 扩展用来指导证书的选择。由于服务器可能需要 server_name 扩展，客户端应该可用情况下发送此扩展。

服务器提供的所有证书必须由客户端提供的签名算法签名（如果客户端能够提供这样的链，见 4.2.3 节）。自签证书或预期为 trust anchors 的证书不作为链的一部分进行验证，因此可以用任何算法进行签名。

如果服务器不能产生一个只通过指定的支持算法进行签名的证书链，那么它应该向客户端发送一个自己选择的包含不知道客户端是否支持的算法证书链来继续握手。这个 fallback 链一般不应使用已淘汰的 SHA-1 散列算法，但如果客户端通告允许的话可以使用，否则不得使用。

如果客户端不能使用提供的证书构建一个可接受的链，并决定放弃握手，那么它必须用一个适当的 certificate-related 警报（默认情况下为 unsupported_certificate；更多信息见 6.2 节）来放弃握手。
如果服务器有多个证书，它就会根据上述标准（除此之外还有其他标准，如传输层端点、本地配置和偏好）选择其中一个。

##### 4.4.2.3. Client Certificate Selection

客户端发送的证书遵循以下规则：

- 证书类型必须是 X.509 v3 [RFC5280]，除非另有明确协商(如 [RFC7250])。
- 如果 CertificateRequest 消息中存在 certificate_authorities 扩展，那么证书链中至少有一个证书应该是由所列的 CA 签发的。
- 证书必须使用可接受的签名算法来签署，如 4.3.2 所述。需要注意的是，这放宽了以前版本的 TLS 中对证书签名算法的限制。
- 如果 CertificateRequest 消息包含一个非空的 oid_filters 扩展，那么终端实体证书必须与客户端识别的扩展名 OID 匹配，如 4.2.5 节所述。

##### 4.4.2.4. Receiving a Certificate Message

一般来说，详细的证书验证程序不在 TLS 的范围内（见 [RFC5280]）。本节提供了特定于 TLS 的要求。

如果服务器提供了空的 Certificate 消息，客户端必须用 decode_error alert 中止握手。

如果客户端没有发送任何证书(比如发送一个空的 Certificate 消息)，服务器可以自行决定是在没有客户端认证的情况下继续握手，还是用 certificate_required 警告中止握手。 另外，如果证书链的某些方面是不能接受的(例如，不是由已知的、受信任的 CA 签发的)，服务器可以自行决定继续握手（考虑到客户端未被认证）或中止握手。

任何端点收到任何需要使用 MD5 哈希签名算法来验证的证书，都必须以 bad_certificate alert 来中止握手。SHA-1 已废弃，建议任何端点收到任何需要使用 SHA-1 散列签名算法来验证的证书时，应当以 bad_certificate 警告中止握手。为明确起见，这意味着端点可以接受这些算法，用于自签或 trust anchors 的证书。

建议所有端点尽快过渡到 SHA-256 或更高版本，以保持与目前正在逐步取消 SHA-1 支持的实施方案的互操作性。

请注意，包含一种签名算法密钥的证书可以使用不同的签名算法进行签名（例如，用 ECDSA 密钥签名的 RSA 密钥）。

#### 4.4.3. Certificate Verify

该消息用于明确证明一个端点拥有与其证书相对应的私钥。CertificateVerify 消息也为截至当前的握手提供完整性。服务器在通过证书进行验证时必须发送此消息，客户端在通过证书进行验证时必须发送此消息。客户端在通过证书进行验证时必须发送此消息（如 Certificate 消息非空时）。该消息必须紧接在 Certificate 消息之后发送，并在 Finished 消息之前出现。

消息结构如下：

```
      struct {
          SignatureScheme algorithm;
          opaque signature<0..2^16-1>;
      } CertificateVerify;
```

算法字段指定所使用的签字算法（该类型的定义见 4.2.3 节）。签名是使用该算法的数字签名。签名的内容是 4.4.1 节所述的哈希输出，即：

```
     Transcript-Hash(Handshake Context, Certificate)
```

然后，数字签名的计算是在下列各项的连接上进行的：

- 由 32（0x20）重复 64 次组成的字符串
- context 字符串
- 一个 0 字节，作为分隔符。
- 需要签署的内容

这个结构的目的是为了防止对以前版本的 TLS 的攻击，在这种情况下，ServerKeyExchange 格式意味着攻击者可以获得一个带有 32 字节前缀（ClientHello.random）的消息签名。 初始的 64 字节填充会清除这个前缀和服务器控制的ServerHello.random。

服务器签名的 context 字符串是 "TLS 1.3, server CertificateVerify"。 客户端签名的 context 字符串是 "TLS 1.3, client CertificateVerify"。 它用来区分不同环境下的签名，有助于防止潜在的跨协议攻击。

例如，如果 transcript 哈希是 32 字节的 01（这个长度对于 SHA-256 来说是有意义的），那么服务器 CertificateVerify 的数字签名所覆盖的内容就是：

```
      2020202020202020202020202020202020202020202020202020202020202020
      2020202020202020202020202020202020202020202020202020202020202020
      544c5320312e332c207365727665722043657274696669636174655665726966
      79
      00
      0101010101010101010101010101010101010101010101010101010101010101
```

在发送方，计算 CertificateVerify 消息的签名字段的输入为：

- 数字签名所涵盖的内容
- 之前信息中发送的证书所对应的私人签名密钥。

如果 CertificateVerify 消息是由服务器发送的，签名算法必须是客户端 signature_algorithms 扩展中提供的算法，除非没有不支持的算法就不能产生有效的证书链（见 4.2.3 节）。

如果是客户端发送的，签名中使用的签名算法必须是 CertificateRequest 消息中 signature_algorithms 扩展中 supported_signature_algorithms 字段中的算法之一。

此外，签名算法必须与发送者的终端实体证书中的秘钥兼容。RSA 签名必须使用 RSASSA-PSS 算法，不管 RSASSA-PKCS1-v1_5 算法是否出现在 signature_algorithms 中。 SHA-1 算法不得用于任何 CertificateVerify 消息的签名中。

本规范中的所有 SHA-1 签名算法只用于传统证书，而不适用于 CertificateVerify 签名。

CertificateVerify 消息的接收方必须验证签名字段。 验证过程的输入是：

- 数字签名所涵盖的内容
- 相关 Certificate 信息中找到的终端实体证书中包含的公钥。
- CertificateVerify 消息的签名中收到的数字签名。

如果验证失败，接收方必须用 decrypt_error alert 终止握手。

#### 4.4.4. Finished

Finished 消息是 Authentication Block 中的最后一条消息，它对提供握手和密钥的认证至关重要。

Finished 消息的接收者必须验证其内容是否正确，如果不正确必须用 decrypt_error alert 终止连接。

一旦一方发送了 Finished 消息，并且收到并验证了对端的 Finished 消息，它就可以开始通过连接发送和接收应用数据。有两种设置允许它在收到对端的 Finished 之前发送数据：

1. 发送 0-RTT 数据的客户端，如 4.2.10 所述。
2. 服务器在第一次发送后可以发送数据，但由于握手还没有完成，所以既不能保证对端的身份，也不能保证它的活性（如 ClientHello 可能已经被重放）。

用于计算 Finished 消息的密钥是由 4.4 节中定义的 Base Key 使用 HKDF 计算的（见7.1节）。具体来说：

```
   finished_key =
       HKDF-Expand-Label(BaseKey, "finished", "", Hash.length)

   Structure of this message:

      struct {
          opaque verify_data[Hash.length];
      } Finished;

   The verify_data value is computed as follows:

      verify_data =
          HMAC(finished_key,
               Transcript-Hash(Handshake Context,
                               Certificate*, CertificateVerify*))

      * Only included if present.
```

HMAC [RFC2104] 采用 Hash 算法进行握手。如上所述，HMAC 的输入一般可以通过运行哈希来实现，如此时只需要握手哈希。

在以前的 TLS 版本中，verify_data 总是 12 个字节长。在 TLS 1.3 中，是用于握手的 Hash 的 HMAC 输出大小。

注意：警报和任何其他非握手记录类型不属于握手消息，不包括在哈希计算中。

在 Finished 消息之后的任何记录必须按照 7.2 节所述的适当应用流量密钥进行加密。特别是，这包括服务器响应客户端 Certificate 和 CertificateVerify 消息而发送的任何警报。

### 4.5. End of Early Data

```
      struct {} EndOfEarlyData;
```

如果服务器在 EncryptedExtensions 中发送了 early_data 扩展，那么客户端必须在收到服务器 Finished 后发送 EndOfEarlyData 消息。如果服务器没有在 EncryptedExtensions 中发送 early_data 扩展，那么客户端不得发送 EndOfEarlyData 消息。该消息表示所有的 0-RTT 应用_数据消息（如果有的话）已经传输完毕，之后的数据由握手流量密钥保护。服务器不得发送此消息，客户端收到此消息必须以 unexpected_message alert 终止连接。这个消息是由 client_early_traffic_secret 衍生的密钥进行加密的。

### 4.6. Post-Handshake Messages

TLS 还允许在主握手之后发送其他消息。这些消息使用握手内容类型，并由合适的应用流量密钥进行加密。

#### 4.6.1. New Session Ticket Message

在收到客户端 Finished 消息后，服务端任何时候都可以发送 NewSessionTicket 消息。 该消息在 ticket 值和从恢复主秘钥中导出的秘密 PSK 之间建立了对应关系（一一对应，见第 7 节）。

客户端可以通过在 ClientHello（4.2.11 节）中的 pre_shared_key 扩展中包含 ticket 值，在未来的握手中使用这个 PSK。服务器可以在一个连接上发送多个 ticket，可以是紧接着发送，也可以是在特定事件之后（见附录 C.4）。例如，服务器可能会在 post-handshake 认证之后发送一个新的 ticket，以便封装额外的客户端认证状态。 多 tikcet 对客户端有多种用途，包括：

- 打开多个并行 HTTP 连接。
- 通过（例如）Happy Eyeballs [RFC8305] 或相关技术跨接口和地址族执行连接竞速。

任何 ticket 必须只用建立原始连接时使用的 KDF 哈希算法相同的密码套件来恢复。

只有当新的 SNI 值对原始会话中提交的服务器证书有效时，客户端才必须恢复；只有当 SNI 值与原始会话中使用的 SNI 值匹配时，客户端才应该恢复。后者是一种性能优化：通常情况下，没有理由期望一个证书所覆盖的不同服务器能够接受对方的 ticket；因此，在这种情况下尝试恢复会浪费一张单次使用的 ticket。 如果提供了这样的指示（外部或任何其他方式），客户端可以用不同的 SNI 值恢复。

如果恢复时向应用程序传递 SNI 值，实现必须使用恢复 ClientHello 中发送的值，而不是上一个会话中发送的值。注意，如果服务器实现拒绝所有不同 SNI 值的 PSK 标识，则这两个值总是相同的。

注意：虽然恢复主秘钥取决于客户端的第二次发送，但不要求客户端认证的服务器可以独立计算 transcript 的剩余部分，然后在发送 Finished 后立即发送 NewSessionTicket，而不是等待客户端 Finished。 这可能适用于这样的情况：例如，客户端预计将并行打开多个 TLS 连接，并将从减少恢复握手的开销中受益。

```
      struct {
          uint32 ticket_lifetime;
          uint32 ticket_age_add;
          opaque ticket_nonce<0..255>;
          opaque ticket<1..2^16-1>;
          Extension extensions<0..2^16-2>;
      } NewSessionTicket;
```

- ticket_lifetime：表示从发布 ticket 时间开始的 32 位无符号整数，以秒为单位，网络字节序。服务器不得使用任何大于 604800 秒（7天）的值。 值为零时表示 ticket 应立即丢弃。无论 ticket_lifetime 多少，客户端都不得将 ticket 缓存超过 7 天，可以根据本地政策提前删除 ticket。 服务器可以在比 ticket_lifetime 更短的时间内将票据视为有效。
- ticket_age_add：一个安全生成的随机 32 位值，用于掩盖客户端 pre_shared_key 扩展中的 ticket 年龄。 此值加上客户端的 tikcet 年龄，mod 2^32，得到客户端传输的值。服务器必须为每次发送的 ticket 生成一个新的值。
- ticket_nonce：每个 ticket 对应一个值，使 ticket 在这个连接上唯一。
- Ticket：作为 PSK 标识的 ticket 值。ticket 本身是一个不透明的标签。 它可以是一个数据库查询键，也可以是一个自加密和自认证的值。
- Extensions：ticket 的一组扩展。扩展格式在 4.2 节中定义。客户端必须忽略不认识的扩展。

目前为 NewSessionTicket 定义的唯一扩展是 early_data，表示该 ticket 可用于发送 0-RTT 数据（4.2.10 节）。 包含以下值：

- max_early_data_size：使用该 ticket 时，客户端允许发送的 0-RTT 数据的最大数量，单位为字节。 大小只包含应用数据有效载荷（如明文，但不包括填充或内部内容类型字节）。 服务器接收到超过 max_early_data_size 字节的 0-RTT 数据时，应该用 unexpected_message 警报终止连接。 请注意，由于缺乏加密材料而拒绝早期数据的服务器将无法区分 padding 和内容，所以客户端不要依赖能够在早期数据记录中发送大量的 padding。

tikcet 关联的 PSK 计算方式为：

```
       HKDF-Expand-Label(resumption_master_secret,
                        "resumption", ticket_nonce, Hash.length)
```

因为每个 NewSessionTicket 消息的 ticket_nonce 值是不同的，所以每个 ticket 都会衍生出不同的 PSK。

请注意，原则上可以继续发行新的 ticket，它可以无限期地延长最初从 initial non-PSK 握手（很可能与对端证书绑定）中派生的密钥材料的寿命。 建议实施者对这种密钥材料的总寿命进行限制；这些限制应考虑到对端证书的寿命、干预性撤销的可能性以及对端在线 CertificateVerify 签名后的时间。

#### 4.6.2. Post-Handshake Authentication

当客户端发送了 post_handshake_auth 扩展（见 4.2.6 节）后，服务器可以在握手完成后的任何时候通过发送 CertificateRequest 消息来请求客户端认证。 客户端必须用适当的 Authentication 消息来响应（见 4.4 节）。 如果客户端选择认证，则必须发送 Certificate、CertificateVerify 和 Finished。 如果客户端拒绝，则必须发送一个不包含证书的 Certificate 消息，然后是 Finished。 客户端对给定响应的所有消息必须连续发送，中间不能有其他类型的消息。
客户端如果在没有发送 post_handshake_auth 扩展的情况下接收到 CertificateRequest 消息，必须发送一个 unexpected_message 的致命警告。

注意：由于客户端认证可能涉及到提示用户，服务器必须准备好一些延迟，包括在发送 CertificateRequest 和接收响应之间收到任意数量的其他消息。 此外，如果客户端连续收到多个 CertificateRequest，可能会以不同的顺序响应（certificate_request_context 值允许服务器识别响应）。

#### 4.6.3. Key and Initialization Vector Update

KeyUpdate 握手消息用于指示发送方正在更新其发送的加密密钥。这个消息可以由任何一端在 Finished 消息后发送。在收到 Finished 消息之前收到 KeyUpdate 消息必须用 unexpected_message alert 来终止连接。在发送 KeyUpdate 消息后，发送方必须使用下一代密钥发送所有流量，这些密钥按照 7.2 节的描述计算。在收到 KeyUpdate 消息后，接收方必须更新接收密钥。

```
      enum {
          update_not_requested(0), update_requested(1), (255)
      } KeyUpdateRequest;

      struct {
          KeyUpdateRequest request_update;
      } KeyUpdate;
```

- request_update：表示 KeyUpdate 的接收者是否应该用自己的 KeyUpdate 来响应。如果接收到任何其他的值，必须用 illegal_parameter alert 来终止连接。

如果 request_update 字段被设置为 update_requested，那么接收方必须在发送下一个应用数据之前，发送一个自己的 KeyUpdate，并将 request_update 设置为 update_not_requested。这个机制允许任何一方强制更新整个连接，但是会导致接收多个 KeyUpdates 的实现在沉默的时候用一个更新来响应。 请注意，在发送 request_update 设置为 update_requested 的 KeyUpdate 和接收对端的 KeyUpdate 之间，可能会收到任意数量的消息，因为这些消息可能已经在路上。 然而，由于发送和接收密钥来自于独立的流量秘钥，因此保留接收流量秘钥并不会威胁到发送者更改密钥之前发送的数据的前向保密性。

如果实现独立发送 request_update 设置为 update_requested 的 KeyUpdate，并且它们在传输中交叉，那么每一方也会发送一个响应，结果就是每一方递增两代。

发送方和接收方都必须用旧密钥加密他们的 KeyUpdate 消息。此外，双方都必须强制要求在接受任何用新密钥加密的消息之前，收到用旧密钥的 KeyUpdate。如果不这样做，可能会引发消息截断攻击。

## 5. Record Protocol

TLS 记录协议将要传输的数据分片为可管理的块，加密后传输。接收到的数据经过验证、解密、重组，然后传递给上层协议。

TLS 记录是分类型的，允许多个上层协议在同一个记录层上复用。本文档规定了四种内容类型：握手、应用数据、警报和 change_cipher_spec。 change_cipher_spec 记录仅用于兼容性目的（见附录 D.4）。

可能会在发送或接收第一个 ClientHello 消息之后、接收到对端的 Finished 消息之前的任何时候接收到一个类型为 change_cipher_spec 的单字节值 0x01 的未加密记录，这种情况必须简单地丢弃而不做进一步处理。 需要注意的是，该记录可能出现在握手时的某一点上（在等待加密的记录），因此，在试图解密记录之前，有必要检测这种情况。 接收到任何其他 change_cipher_spec 值，或者接收到加密 change_cipher_spec 记录，必须以 unexpected_message 警报中止握手。如果在第一个 ClientHello 消息之前或在对端 Finished 消息之后收到 change_cipher_spec 记录，必须视为意外的记录类型（尽管无状态服务器可能无法将这些情况与允许的情况区别开）。

除非经过扩展协商，否则不能发送本文中没定义的记录类型。如果收到意外的记录类型，必须用 unexpected_message alert 来终止连接。新的记录内容类型值由 IANA 在 TLS ContentType 注册表中分配，如 11 节所述。

### 5.1. Record Layer

记录层将信息块分段为 TLSPlaintext 记录，每块携带的数据不多于 2^14 字节。根据底层 ContentType 的不同，信息边界的处理方式也不同。 任何未来的内容类型必须指定适当的规则。 请注意，这些规则比 TLS 1.2 中强制的规则更严格。

握手信息可以合并进一条 TLSPlaintext 记录，也可以分散在几条记录中，但前提是：

- 握手消息不得与其他记录类型交织在一起。也就是说，如果一个握手消息被分割成两条或更多的记录，它们之间不能有任何其他记录。
- 握手消息不得跨越密钥变化。必须确保紧接在密钥变化之前的所有消息是否与记录边界一致；如果不一致，则必须用 unexpected_message alert 终止连接。因为 ClientHello、EndOfEarlyData、ServerHello、Finished 和 KeyUpdate 消息可以在秘钥变化之后立即发送，所以必须按照记录边界来发送这些消息。

不得发送零长度的握手类型片段，即使这些片段包含填充。

警报消息（第6节）绝不能分散在多个记录里，并且多个警报消息绝不能合并到一个 TLSPlaintext 记录中。换句话说，具有 Alert 类型的记录必须只包含一条消息。

Application Data 消息包含对 TLS 不透明的数据。Application Data 消息总是加密的。可以发送零长度的 Application Data 片段，因为可能作为流量分析手段。Application Data 片段可以分散在多个记录中，也可以合并成一个记录。

```
      enum {
          invalid(0),
          change_cipher_spec(20),
          alert(21),
          handshake(22),
          application_data(23),
          (255)
      } ContentType;

      struct {
          ContentType type;
          ProtocolVersion legacy_record_version;
          uint16 length;
          opaque fragment[TLSPlaintext.length];
      } TLSPlaintext;
```

- type: 用于处理所附片段的上层协议。
- legacy_record_version：对于所有 TLS1.3 的实现都必须设置为 0x0303，除了初始的 ClientHello 可以出于兼容性考虑设置为 0x0301（比如在 HelloRetryRequest 之后没有生成）。这个字段已经被废弃，必须忽略。以前版本的 TLS 在某些情况下会在这个字段中使用其他的值。
- length：下面 TLSPlaintext.fragment 的长度（以字节为单位），长度不得超过 2^14 字节。接收到超过此长度的记录必须使用 record_overflow 警报终止连接。
- fragment：正在传输的数据。这个值视为一个独立的块透明传递给类型字段指定的上层协议处理。

本文介绍了 TLS 1.3，使用的版本是 0x0304。这个版本值是历史性的，源于 TLS 1.0 的 0x0301 和 SSL 3.0 的 0x0300。为了最大限度地提高向后兼容性，包含初始 ClientHello 的记录必须有 0x0301 版本（代表 TLS 1.0），包含第二个 ClientHello 或 ServerHello 的记录必须有 0x0303 版本（代表TLS 1.2）。当协商以前版本的 TLS 时，端点遵循附录 D 中提供的程序和要求。

当记录保护尚未参与时，TLSPlaintext 结构会直接发送。 记录保护开始后，TLSPlaintext 记录将受到保护，并按下一节所述发送。 请注意，Application Data 记录不得在未受保护的情况下发送（详情见第 2 节）。

### 5.2. Record Payload Protection

记录保护函数将 TLSPlaintext 结构转换为 TLSCiphertext 结构。去保护函数则相反。 TLS 1.3 与之前 TLS 版本不同，所有的密码都被建模为关联数据认证加密（AEAD，Authenticated Encryption with Associated Data）[RFC5116]。 AEAD 功能提供了统一的加密和认证操作，将明文转变成经过认证的密文，然后再转回来。 每条加密记录由一个 plaintext 头组成，后面是一个加密体，加密体包含一个类型和可选填充。

```
      struct {
          opaque content[TLSPlaintext.length];
          ContentType type;
          uint8 zeros[length_of_padding];
      } TLSInnerPlaintext;

      struct {
          ContentType opaque_type = application_data; /* 23 */
          ProtocolVersion legacy_record_version = 0x0303; /* TLS v1.2 */
          uint16 length;
          opaque encrypted_record[TLSCiphertext.length];
      } TLSCiphertext;
```

- Content：TLSPlaintext.fragment 值，包含握手或警报消息的字节编码，或应用程序要发送的原始数据。
- type：TLSPlaintext.type 值，包含记录的内容类型。
- zeros：类型字段后的 cleartext 中可能出现任意长度的零值字节。这为发送者提供了一个机会，只要总长度不超过记录大小的限制，发送者就可以用选择的数量来填充任何 TLS 记录。 更多细节见 5.4 节。
- opaque_type： TLSCiphertext 记录的外层 opaque_type 字段总是设置为 23(application_data)，以兼容习惯于以前版本 TLS 的中间件。 记录的实际内容类型可以在解密后的 TLSInnerPlaintext.type 中找到。
- legacy_record_version：legacy_record_version 字段总是 0x0303。TLS 1.3 的 TLSCiphertexts 是在 TLS 1.3 协商后才生成的，所以不存在收到其他值的历史兼容性问题。 请注意，握手协议，包括 ClientHello 和 ServerHello 消息，都会对协议版本进行认证，所以这个值是冗余的。
- length：以下 TLSCiphertext.encrypted_record 的长度(以字节为单位)，是内容长度加上填充长度，加上内部内容类型的长度，再加上 AEAD 算法添加的任何扩展。长度不得超过 2^14+256 字节。 接收到超过这个长度的记录必须用 record_overflow 警报终止连接。
- encrypted_record：序列化 TLSInnerPlaintext 结构的 AEAD 加密格式。

AEAD 算法的输入是一个密钥、一个 nonce、一个 plaintext 和 "附加数据"，这些数据将被包含在认证检查中，如 [RFC5116] 2.1 节所述。秘钥是 client_write_key 或 server_write_key，nonce 是从序列号和 client_write_iv 或 server_write_iv 中分离的(见 5.3 节)，附加数据是记录头。

如：

```
      additional_data = TLSCiphertext.opaque_type ||
                        TLSCiphertext.legacy_record_version ||
                        TLSCiphertext.length
```

AEAD 算法的 plaintext 输入是经过编码的 TLSInnerPlaintext 结构。 流量密钥的派生在 7.3 节中定义。

AEAD 输出由 AEAD 加密操作输出的密文组成。由于包含了 TLSInnerPlaintext.type 和发送者的填充，明文的长度大于相应的 TLSPlaintext.length。 AEAD 输出的长度一般会比明文大，但大小随 AEAD 算法而变化。

由于密文可能包含填充，开销的数量可能会随着不同长度的明文而变化。典型地：

```
      AEADEncrypted =
          AEAD-Encrypt(write_key, nonce, additional_data, plaintext)
```

TLSCiphertext 的 encrypted_record 字段设置为 AEADEncrypted。

为了解密和验证，解密函数将密钥、nonce、附加数据和 AEADEncrypted 值作为输入。输出是明文或表示解密失败的错误。没有单独的完整性检查。符号化为：

```
      plaintext of encrypted_record =
          AEAD-Decrypt(peer_write_key, nonce,
                       additional_data, AEADEncrypted)
```

如果解密失败，接收方必须以bad_record_mac警告终止连接。

在 TLS 1.3 中使用的 AEAD 算法不得产生大于 255 字节的扩展。如果从对端接收到 TLSCiphertext.length 大于 2^14+256 八位数的记录，必须用 record_overflow alert 终止连接。 这个限制是由最大的 TLSInnerPlaintext 长度 2^14 + 一个字节的 ContentType + 最大的 AEAD 扩展的 255 字节得出的。

### 5.3. Per-Record Nonce

读取和写入记录会分别维护一个 64 位的序列号。读取或写入每条记录后，相应的序列号都会递增 1。序列号在连接开始时和改变密钥时都被设置为 0；在特定流量密钥下传输的第一条记录必须使用序列号 0。

因为序列号的大小是 64 位，所以不应该 wrap。如果 TLS 实现需要对序列号进行 wrap，那么它必须 rekey（4.6.3 节）或者终止连接。

每一种 AEAD 算法都会规定每记录 nonce 的长度范围，从 N_MIN 字节到 N_MAX 字节的输入 [RFC5116]。对于 AEAD 算法来说，TLS 每记录 nonce 的长度(iv_length)被设置为 8 字节和 N_MIN 中较大的一个(见 [RFC5116] 第 4 节)。 如果 N_MAX 小于 8 个字节，那么 AEAD 算法就不能用于 TLS。AEAD 结构中的每记录 nonce 的构成如下：

1. 64 位记录序列号按网络序编码，并向左加零到 iv_length。
2. 填充的序列号与静态 client_write_iv 或 server_write_iv（取决于角色）进行异或。

所得到的结果（长度为iv_length）被用作每记录的 nonce。

注意：这与 TLS 1.2 中的结构不同，TLS 1.2 指定了一个部分显式的 nonce。

### 5.4. Record Padding

所有加密的 TLS 记录都可以被填充以增加 TLSCiphertext 的大小。 这允许发送者对观察者隐藏流量的大小。

当生成 TLSCiphertext 记录时，实现者可以选择填充。一个未填充的记录只是一个填充长度为零的记录。 填充是加密前附加到 ContentType 字段的零值的字符串。在加密前，实现者必须将 padding 设置为全零。

如果发送者愿意，Application Data 记录可以包含一个零长度的 TLSInnerPlaintext.content。这允许在敏感活动存在或不存在的情况下生成合理大小的覆盖流量。不能发送包含零长度 TLSInnerPlaintext.content 的握手和警报记录；如果收到这样的消息，接收者必须用 unexpected_message alert 来终止连接。

发送的填充由记录保护机制自动验证；在成功解密一个 TLSCiphertext.encrypted_record 后，接收者从末尾向开头扫描该字段，直到找到一个非零字节。 这个非零字节就是消息的内容类型。之所以选择这种填充方案，是因为它允许对任何加密的 TLS 记录进行任意大小的填充（从零到 TLS 记录大小限制），而不需要引入新的内容类型。该设计还强制执行全零的 padding，这允许快速检测填充错误。

实现必须将其扫描限制在 AEAD 解密返回的 cleartext 上。 如果接收者没有在 cleartext 中找到非零字节，则必须用 unexpected_message alert 来终止连接。

填充的存在不会改变整个记录大小限制：完整编码的 TLSInnerPlaintext 不得超过 2^14+1 字节。 如果最大的片段长度被减小（例如，通过 [RFC8449] 的 record_size_limit 扩展），那么减小的限制适用于完整的 plaintext，包括内容类型和填充。

选择一个填充策略，建议何时填充，填充多少，是一个复杂的话题，超出了本文的范围。 如果在 TLS 之上的应用层协议有自己的填充，那么在应用层中对应用数据TLS记录进行填充可能是比较好的。 不过，加密的握手或警报记录的填充仍然必须在 TLS 层处理。 以后的文档可能会定义 padding 选择算法，或者通过 TLS 扩展或其他方式定义一个 padding 策略请求机制。

### 5.5. Limits on Key Usage

在一组给定的密钥下，可以安全加密的明文数量是有密码学限制的。[AEAD-LIMITS] 提供了对这些限制的分析，其假设是底层基元（AES 或 ChaCha20）没有弱点。在达到这些限制之前，实现应该按照 4.6.3 节的描述进行密钥更新。

对于 AES-GCM 来说，在给定的连接上，最多可加密 2^24.5 大小的记录（约 2400 万条），同时保持约 2^-57 的安全系数，以保证验证加密（AE，Authenticated Encryption）的安全性。 对于 ChaCha20/Poly1305，记录序列号将在达到安全限值之前被 wrap。

## 6. Alert Protocol

TLS 提供了一个 Alert 内容类型来指示关闭信息和错误。和其他消息一样，警告消息也是根据当前连接状态进行加密的。

警告消息传达了警告的描述和一个遗留字段，在以前的 TLS 版本中传达了消息的严重程度。 警告分为两类：关闭警告和错误警告。 在 TLS 1.3 中，警告的严重性是隐含在警告的类型中的，"level" 字段可以被忽略。 close_notify 警报用于指示连接的一个方向的有序关闭。当收到这样的告警时，TLS 应该向应用程序发出数据结束的提示。

错误警报表示连接的中止关闭（见6.2节）。当收到错误警报时，TLS 应该向应用程序提示错误，并且不允许再在连接上发送或接收任何数据。服务器和客户端必须丢弃在失败的连接中建立的秘密值和密钥，但与会话 ticket 相关联的 PSK 除外，如果可能的话，这些 PSK 应该被丢弃。

6.2 节中列出的所有告警必须以 AlertLevel=fatal 的方式发送，并且不管消息中的 AlertLevel 是什么都必须作为错误告警处理。 未知的警报类型必须作为错误警报处理。

注意：TLS 定义了两个通用的警报（见第6节），消息解析失败时使用。当对端收到不能按照语法解析的消息时（例如，消息长度超出了消息边界或包含一个超出范围的长度），必须使用 decode_error alert 终止连接。如果接收到语法正确但语义无效的消息（例如，DHE share 为 p - 1，或无效的枚举），必须以 illegal_parameter alert 终止连接。

```
      enum { warning(1), fatal(2), (255) } AlertLevel;

      enum {
          close_notify(0),
          unexpected_message(10),
          bad_record_mac(20),
          record_overflow(22),
          handshake_failure(40),
          bad_certificate(42),
          unsupported_certificate(43),
          certificate_revoked(44),
          certificate_expired(45),
          certificate_unknown(46),
          illegal_parameter(47),
          unknown_ca(48),
          access_denied(49),
          decode_error(50),
          decrypt_error(51),
          protocol_version(70),
          insufficient_security(71),
          internal_error(80),
          inappropriate_fallback(86),
          user_canceled(90),
          missing_extension(109),
          unsupported_extension(110),
          unrecognized_name(112),
          bad_certificate_status_response(113),
          unknown_psk_identity(115),
          certificate_required(116),
          no_application_protocol(120),
          (255)
      } AlertDescription;

      struct {
          AlertLevel level;
          AlertDescription description;
      } Alert;
```

### 6.1. Closure Alerts

客户端和服务器必须共享连接结束的信息，以避免截断攻击。

- close_notify：这个警报通知接收者，发送者不会再在这个连接上发送任何消息。在收到关闭警报后收到任何数据都必须被忽略。
- user_canceled：该警报通知接受者，发送者将不再在该连接上发送任何消息。该警报通知接收者，发送者因为某些与协议失败无关的原因取消了握手。如果用户在握手完成后取消了一个操作，那么通过发送close_notify来关闭连接更为合适。这个警报后面应该有一个 close_notify。这个告警一般 AlertLevel=warning。

任何一方都可以通过发送 close_notify alert 来关闭其写侧的连接。在收到关闭警报后收到的任何数据都必须忽略。如果在 close_notify 之前收到了一个传输级的关闭，那么接收者就不能知道所有发送的数据都已经收到了。

每一方必须在关闭其连接的写侧之前发送 close_notify alert，除非它已经发送了一些错误警报。这对它的读端连接没有任何影响。请注意，这是 TLS 1.3 对之前 TLS 版本的一个变化，在 TLS 1.3 之前的版本中，需要对 close_notify 做出反应，丢弃等待写的数据，并立即发送 close_notify alert。之前的要求可能会导致读端截断。双方不需要等待收到 close_notify 警报后再关闭其读端连接，尽管这样做会带来截断的可能性。

如果使用 TLS 的应用协议在 TLS 连接关闭后仍然有数据想要通过底层传输，TLS 实现必须收到一个 close_notify 警报再向应用层指示数据结束。这个标准的任何部分都不应该被视为规定了 TLS 的使用配置文件管理数据传输的方式，包括何时打开或关闭连接。

注意：我们假设关闭连接的写端会在销毁传输之前可靠地传送待处理的数据。

### 6.2. Error Alerts

TLS 中的错误处理非常简单。 当检测到错误时，检测方会向对端发送一条消息。 在发送或收到致命的警报消息后，双方必须立即关闭连接。

每当遇到致命的错误条件时，应该发送一个适当的致命警告，并且必须关闭连接，不能再发送或接收任何数据。在本规范的其余部分，当使用 "终止连接"（terminate the connection）和 "中止握手"（abort the handshake）这两个短语时，如果没有特定的警报，则意味着应该发送下面描述的警报。 短语 "终止连接并发出X警报" 和 "中止握手并发出X警报" 意味着如果要发出警报必须发送警报X。 本节中定义的所有警报，以及所有未知的警报，从 TLS 1.3 开始被认为是致命的（见第6节）。实现应该提供一种方法来方便记录警报的发送和接收。

定义了以下错误提示：

- unexpected_message：收到了不适当的信息（例如，错误的握手信息、过早的应用数据等）。在适当的实现之间的通信中绝对不应该观察到这个警报。
- bad_record_mac：如果收到一个不能解密的记录，就会返回这个警报。 因为AEAD算法结合了解密和验证，同时也为了避免侧信道攻击，所以这个警报用于所有的解密失败。在适当的实现之间的通信中，除非消息在网络中被破坏，否则绝不应该观察到这个警报。
- record_overflow：收到的 TLSCiphertext 记录长度超过 2^14+256 字节，或记录解密后的 TLSPlaintext 记录长度超过 2^14 字节(或其他协商的限制)。 在适当的实现之间的通信中，除非消息在网络中被破坏，否则绝不应该观察到这个警报。
- handshake_failure：收到 handshake_failure 的警报消息表明发送者无法在给定选项下协商出一套可接受的安全参数。
- bad_certificate：证书损坏了，包含的签名不能通过验证等。
- unsupported_certificate：证书的类型不受支持。
- certificate_revoked：证书被签署者撤销。
- certificate_expired：证书已过期。证书已过期或当前无效。
- certificate_unknown：在处理证书的过程中出现了一些其他(未指明)问题，导致无法接受。
- illegal_parameter：握手中的一个字段不正确或与其他字段不一致。该警报用于符合正式协议语法但其他方面不正确的错误。
- unknown_ca：收到了有效的证书链或部分证书链，但证书没有被接受，因为无法找到 CA 证书或无法与已知的 trust anchor 匹配。
- access_denied：收到了一个有效的证书或 PSK，但当使用了访问控制，发送者决定不继续协商。
- decode_error：由于某些字段不在指定范围内或消息的长度不正确，消息不能被解码。 该警报用于消息不符合正式协议语法的错误。在适当的实现之间的通信中绝对不应该观察到这个警报，除非消息在网络中被破坏了。
- decrypt_error：握手（不是记录层）解密操作失败，包括签名无法验证通过或无法验证 Finished 消息或 PSK binder。
- protocol_version： 对端想要协商的协议版本可以识别但不支持（见附录D）。
- insufficient_security：当协商失败时，替代 handshake_failure 返回，具体原因是服务器要求的参数比客户端支持的更安全。
- internal_error： 内部错误。 与对端或协议的正确性无关的内部错误（如内存分配失败）使其无法继续。
- inappropriate_fallback：由服务器响应客户端无效连接重试而发送（见 [RFC7507]）。
- missing_extension：收到不包含必要扩展的握手消息则发送。
- unsupported_extension：接收到包含禁止在给定握手中使用的扩展则发送，或者 ServerHello 或 Certificate 中包含任何没有在相应的 ClientHello 或 CertificateRequest 中提供的扩展。
- unrecognized_name：当客户端通过 server_name 扩展提供的名称所标识的服务器不存在时，由服务器发送（见 [RFC6066]）。
- bad_certificate_status_response：当服务器通过 status_request 扩展提供无效或不可接受的 OCSP 响应时，由客户端发送（见 [RFC6066]）。
- unknown_psk_identity：当需要建立 PSK 密钥，但客户端没有提供可接受的 PSK 标识时，服务器会发送该警报。发送该警报是可选的；服务器可以选择发送 decrypt_error 警报，以表明无效的PSK标识。
- certificate_required：当需要客户端证书，但客户没有提供证书时，由服务器发送。
- no_application_protocol：当客户端的 application_layer_protocol_negotiation 扩展只提供了服务器不支持的协议时，由服务器发送（参见 [RFC7301]）。

新的警报值由 IANA 分配，如第 11 节所述。

## 7. Cryptographic Computations

TLS 的握手建立了一个或多个输入 secret，组合起来创建实际的工作密钥材料，详见下文。密钥推导过程包含了输入 secret 和握手 transcript。 需要注意的是，由于握手 transcript 包含了 Hello 消息中的随机值，因此任何一次握手都会有不同的流量 secret，即使使用了相同的输入 secret，就像在多个连接中使用相同的 PSK 一样。

### 7.1. Key Schedule

密钥推导过程使用了 HKDF 定义的 HKDF-Extract 和 HKDF-Expand 函数 [RFC5869]，以及下面定义的函数：

```
       HKDF-Expand-Label(Secret, Label, Context, Length) =
            HKDF-Expand(Secret, HkdfLabel, Length)

       Where HkdfLabel is specified as:

       struct {
           uint16 length = Length;
           opaque label<7..255> = "tls13 " + Label;
           opaque context<0..255> = Context;
       } HkdfLabel;

       Derive-Secret(Secret, Label, Messages) =
            HKDF-Expand-Label(Secret, Label,
                              Transcript-Hash(Messages), Hash.length)
```

Transcript-Hash 和 HKDF 使用的 Hash 函数是密码套件的哈希算法。Hash.length 是以字节为单位的输出长度。Messages 是指握手消息，包括握手消息类型和长度字段，但不包括记录层头。请注意，在某些情况下，一个零长度的 Context（用""表示）被传递给 HKDF-Expand-Label。本文中提到的标签都是 ASCII 字符串，不包括尾部的 NUL 字节。

注意：对于普通的散列函数，任何超过 12 字符的标签都需要散列函数额外迭代计算。本规范中的标签都选在这个限制范围内。

密钥是使用 HKDF-Extract 和 Derive-Secret 函数从两个输入的 secret 中导出的。一般来说，添加一个新 secret 的模式是使用 HKDF-Extract，使用 Salt 作为当前的 secret 状态，输入密钥材料（IKM，Input Keying Material）作为新 secret 添加。在 TLS 1.3 这个版本中，这两个输入secret是：

- PSK (外部建立的预共享密钥，或从以前连接中的 resumption_master_secret 值导出)
- (EC)DHE 共享 secret (7.4 节)

这产生了一个完整的密钥推导表，如下图所示。此图使用以下格式约定：

- HKDF-Extract 从上取 Salt 参数，从左取 IKM 参数，其输出在底部，输出的名称在右侧。
- Derive-Secret 的 Secret 参数用进位箭头表示。例如，Early Secret 是生成 client_early_traffic_secret 的 Secret。
- "0" 表示一串 Hash.length 字节设置为 0。

```
             0
             |
             v
   PSK ->  HKDF-Extract = Early Secret
             |
             +-----> Derive-Secret(., "ext binder" | "res binder", "")
             |                     = binder_key
             |
             +-----> Derive-Secret(., "c e traffic", ClientHello)
             |                     = client_early_traffic_secret
             |
             +-----> Derive-Secret(., "e exp master", ClientHello)
             |                     = early_exporter_master_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   (EC)DHE -> HKDF-Extract = Handshake Secret
             |
             +-----> Derive-Secret(., "c hs traffic",
             |                     ClientHello...ServerHello)
             |                     = client_handshake_traffic_secret
             |
             +-----> Derive-Secret(., "s hs traffic",
             |                     ClientHello...ServerHello)
             |                     = server_handshake_traffic_secret
             v
       Derive-Secret(., "derived", "")
             |
             v
   0 -> HKDF-Extract = Master Secret
             |
             +-----> Derive-Secret(., "c ap traffic",
             |                     ClientHello...server Finished)
             |                     = client_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "s ap traffic",
             |                     ClientHello...server Finished)
             |                     = server_application_traffic_secret_0
             |
             +-----> Derive-Secret(., "exp master",
             |                     ClientHello...server Finished)
             |                     = exporter_master_secret
             |
             +-----> Derive-Secret(., "res master",
                                   ClientHello...client Finished)
                                   = resumption_master_secret
```

这里的一般模式是，图中左边显示的 secret 只是没有上下文的原始熵，而右边显示的 secret 包括握手上下文，因此可以用来导出工作密钥，而不需要额外的上下文。请注意，对 Derive-Secret 的不同调用可能会使用不同的 Messages 参数，即使是相同的 secret。 在 0-RTT 中，Derive-Secret 被调用时有四个不同的 transcript；在仅 1-RTT 中，它被调用时有三个不同的 transcript。

如果给定的 secret 不可用，则使用 Hash.length 字节的 0 值。 请注意，这并不意味着跳过一轮，所以如果没有使用 PSK，Early Secret 仍将是 HKDF-Extract(0，0)。 对于 binder_key 的计算，外部 PSK（在 TLS 之外提供的 PSK）的标签为 ext binder，恢复 PSK（作为之前握手的恢复主 secret 提供的 PSK）的标签为 res binder。 不同的标签防止了一种PSK被另一种 PSK 所替代。
有多个潜在的 Early Secret值，取决于服务器最终选择的 PSK。 客户端需要为每个潜在的 PSK 计算一个值；如果没有选择 PSK，则需要计算对应于零 PSK 的 Early Secret。

一旦计算出所有从某个 secret 中得出的值，该 secret 就应该被删除。

### 7.2. Updating Traffic Secrets

一旦握手完成，任何一方都可以使用 4.6.3 节中定义的 KeyUpdate 握手消息更新其发送的流量密钥。 如本节所述，通过从 client_/server_application_traffic_secret_N 中生成 client_/server_application_traffic_secret_N+1 来计算下一代流量密钥，然后如7.3节所述重新生成流量密钥。

下一代 application_traffic_secret 的计算方式为：

```
   The next-generation application_traffic_secret is computed as:

       application_traffic_secret_N+1 =
           HKDF-Expand-Label(application_traffic_secret_N,
                             "traffic upd", "", Hash.length)
```

一旦计算出 client_/server_application_traffic_secret_N+1 及其相关的流量密钥，实现者就应该删除 client_/server_application_traffic_secret_N 及其相关的流量密钥。

### 7.3. Traffic Key Calculation

流量密钥材料由以下输入值生成：
   - 一个 secret 值
   - 表示正在生成的具体值的 purpose 值
   - 正在生成的密钥的长度

流量密钥材料由输入的流量secret值生成，使用：

- [sender]_write_key = HKDF-Expand-Label(Secret, "key", "", key_length)
- [sender]_write_iv  = HKDF-Expand-Label(Secret, "iv", "", iv_length)
- [sender] 表示发送方。 各记录类型的 Secret 值如下表所示：

```
       +-------------------+---------------------------------------+
       | Record Type       | Secret                                |
       +-------------------+---------------------------------------+
       | 0-RTT Application | client_early_traffic_secret           |
       |                   |                                       |
       | Handshake         | [sender]_handshake_traffic_secret     |
       |                   |                                       |
       | Application Data  | [sender]_application_traffic_secret_N |
       +-------------------+---------------------------------------+
```

每当底层 Secret 变化时，所有的流量密钥材料都会被重新计算（例如，从握手密钥变为应用数据密钥或密钥更新时）。

### 7.4. (EC)DHE Shared Secret Calculation

#### 7.4.1. Finite Field Diffie-Hellman

对于有限域组，进行传统的 Diffie-Hellman [DH76] 计算。 协商后的密钥（Z）以大字节序转为字节串，并以 0 左垫，直到质数的大小。 这个字节串在上面的密钥图中被用作 shared secret。

请注意，这种结构与之前的TLS版本不同，后者去掉了前面的零。

#### 7.4.2. Elliptic Curve Diffie-Hellman

对于 secp256r1、secp384r1 和 secp521r1，ECDH 计算(包括参数和密钥生成以及 shared secret 计算)按照 [IEEE1363] 使用 ECKAS-D1H 方案进行，以 identity map 作为密钥派生函数(KDF)，因此 shared secret 是以字符串表示的 ECDH shared secret 椭圆曲线点的 x 坐标。需要注意的是，FE2OSP(Field Element to Octet String Conversion Primitive)输出的这个字符串(IEEE1363 术语中的"Z")对于任何给定字段来说都是恒定长度的；在这个八位数字串中前面的零一定不能被截断。
   (请注意，这种身份 KDF 的使用是一个技术问题。完整的情况是，ECDH 采用了一个重要的 KDF，因为 TLS 除了计算其他 secret 外，并不直接使用这个 secret。)

对于 X25519 和 X448，ECDH 的计算方法如下：

- 放入 KeyShareEntry.key_exchange 结构的公钥是将 ECDH 标量乘法函数应用于适当长度的秘钥（放入标量输入）和标准的公共基点（放入 u 坐标点输入）的结果。
- ECDH shared secret 是将 ECDH 标量乘法函数应用于秘钥（成标量输入）和对端的公钥（作为 u 坐标点输入）的结果。输出的结果是不经过任何处理直接使用的。

对于这些曲线，实现者应该使用 [RFC7748] 中指定的方法来计算 Diffie-Hellman shared secret。实现者必须检查计算出的 Diffie-Hellman shared secret 是否是全零值，如果是，则按照 [RFC7748] 第 6 节的描述中止。 如果实现者使用这些椭圆曲线的替代实现，他们必须执行 [RFC7748] 第 7 节中规定的附加检查。

### 7.5. Exporters

[RFC5705] 用 TLS 伪随机函数 (PRF, pseudorandom function) 定义了 TLS 的密钥材料导出器。 本文用 HKDF 替换了 PRF，因此需要一个新的结构。 输出器接口保持不变。

输出器的值计算为：

```
   TLS-Exporter(label, context_value, key_length) =
       HKDF-Expand-Label(Derive-Secret(Secret, label, ""),
                         "exporter", Hash(context_value), key_length)
```

其中 Secret 是 early_exporter_master_secret 或 exporter_master_secret。除非应用程序明确指定，否则必须使用 exporter_master_secret。early_exporter_master_secret 定义为用于 0-RTT 数据需要 exporter 的设置。 建议为早期 exporter 定义一个单独的接口，这样可以避免 exporter 用户在需要常规 exporter 时意外地使用早期 exporter，反之亦然。

如果没有提供上下文，context_value 的长度为零。因此，不提供上下文和提供空上下文的计算结果是一样的。 这与以前的 TLS 版本不同，在以前的版本中，空上下文的输出与没有上下文的输出是不同的。 从本文发布之时起，无论是否有上下文，都不使用分配的 exporter 标签。 未来的规范不得定义导出器使用空上下文和无上下文使用相同的标签。导出器的新用途应该在所有导出器计算中提供一个上下文，尽管这个值可以是空的。

对导出器标签格式的要求在 [RFC5705] 第 4 节中定义。

## 8. 0-RTT and Anti-Replay

如 2.3 节和附录 E.5 所述，TLS 没有为 0-RTT 数据提供内在重放保护。有两种潜在的威胁需要关注：

- 攻击者通过简单复制 0-RTT 数据进行重放攻击。
- 攻击者利用客户端重试行为，使服务器接收到一个消息的多个副本。这种威胁在一定程度上已经存在，因为重视健壮性的客户端会通过重试请求来应对网络错误。 然而，0-RTT 为任何不保持全局一致的服务器系统增加了一个额外的维度。具体来说，如果一个服务器系统有多个区， A 区的 ticket 在 B 区不被接受，那么攻击者可以将打算用于 A 区的 ClientHello 和早期数据复制到 A 区和 B 区，在 A 区，数据将以 0-RTT 的方式被接受，但在 B 区，服务器将拒绝 0-RTT 数据，强制进行完全握手。 如果攻击者阻止了 A 的 ServerHello，那么客户端将与 B 完成握手，并可能重试请求，导致整个服务器系统重复。

第一类攻击可以通过共享状态来防止，以保证 0-RTT 数据最多接受一次。 服务器应该通过实施本节所述的方法或通过同等手段提供该级别的重播安全。 然而，我们知道，由于操作上的考虑，并非所有的部署都会将状态保持在该级别。 因此，在正常操作中，客户端将不知道服务器实现了这些机制中的哪一种（如果有的话），因此必须只发送他们认为可以安全重放的早期数据。

除了重播的直接影响外，还有一类攻击，即使是通常被认为是幂等的操作也会被大量重放利用（定时攻击、资源限制耗尽和其他，如附录 E.5 所述）。 可以通过确保每个 0-RTT 有效载荷只能重放有限次数来缓解这些攻击。 服务器必须确保任何实例（无论是一台机器、一个线程或相关服务基础设施中的任何其他实体）最多接受一次同一 0-RTT 握手的 0-RTT；这将重播次数限制在部署的服务器实例数量上。 这样的保证可以通过本地记录最近收到的 ClientHello 的数据并拒绝重复，或者通过任何其他能够提供相同或更强保证的方法来实现。 "每个服务器实例最多一次"的保证是最低要求，服务器应该在可行的情况下进一步限制 0-RTT 重播。

第二类攻击在 TLS 层无法防止，必须由应用程序来处理。 需要注意的是，任何应用程序的客户端实现任何类型的重试行为，都需要实现某种反重试防御。

### 8.1. Single-Use Tickets

最简单的防重放防御方式是服务器只允许每个会话票据使用一次。 例如，服务器可以维护一个所有未使用的有效票据的数据库，在使用时从数据库中删除每个票据。 如果提供了未知的票据，服务器就会回落到完全握手。

如果票据不是自带的，而是数据库密钥，相应的 PSK 在使用时被删除，那么使用 PSK 建立的连接就享有前向保密性。这样就提高了所有 0-RTT 数据和 PSK 使用的安全性，当使用 PSK 时，不使用 (EC)DHE。

由于这种机制在有多个分布式服务器的环境中，需要在服务器节点之间共享会话数据库，因此与自加密票证相比，可能很难实现 PSK 0-RTT 连接的高成功率。与会话数据库不同的是，即使没有一致的存储，会话票也可以成功地进行基于 PSK 的会话建立，不过当允许 0-RTT 时，它们仍然需要一致的存储，以便对 0-RTT 数据进行反重放，详见下节。

### 8.2. Client Hello Recording

反重放攻击的另一种形式是记录一个从 ClientHello 中派生出来的唯一值（一般是随机值或 PSK binder），并拒绝重复。 记录所有的 ClientHello 会导致状态无限制地增长，但服务器也可以记录给定时间窗口内的 ClientHello，并使用 obfuscated_ticket_age 来确保 ticket 在该窗口之外不会被重复使用。

为了实现这一点，当接收到 ClientHello 时，服务器首先验证 PSK binder，如 4.2.11 节所述。然后，按照下一节的描述计算预期的到达时间（expected_arrival_time），如果在记录窗口之外，则拒绝 0-RTT，回退到 1-RTT 握手。

如果 expected_arrival_time 在窗口内，那么服务器会检查是否记录了一个匹配的 ClientHello。 如果找到了，要么用 illegal_parameter 警告中止握手，要么接受 PSK 但拒绝 0-RTT。 如果没有找到匹配的 ClientHello，那么就接受 0-RTT，然后将 ClientHello 存储至 expected_arrival_time。 服务器也可以以 false positive 实现数据存储，例如 Bloom 过滤器，在这种情况下，必须通过拒绝 0-RTT 来响应明显的重放，但不得中止握手。

服务器必须只从 ClientHello 的验证部分导出存储密钥。 如果 ClientHello 包含多个 PSK 身份，那么攻击者可以在假设服务器不会验证的情况下，使用不同的 binder 值为较不喜欢的身份创建多个 ClientHello（如4.2.11节所推荐的）。 比如，如果客户端发送 PSK A 和 B，但服务器更喜欢 A，那么攻击者可以改变 B 的 binder，而不影响 A 的 binder，如果 B 的 binder 是存储密钥的一部分，那么这个 ClientHello 不会被当作重复，这将导致 ClientHello 被接受，并可能导致重放缓存污染等副作用，尽管 0-RTT 数据无法解密，因为它将使用不同的密钥。 如果使用验证过的 binder 或 ClientHello.random 作为存储密钥，那么这种攻击是不可能的。

由于这种机制不需要存储所有未完成的 ticket，因此在具有高恢复率和 0-RTT 的分布式系统中可能更容易实现，但代价是由于难以可靠地存储和检索接收到的 ClientHello 消息，因此反重放防御可能较弱。在许多这样的系统中，对所有接收到的 ClientHello 进行全局一致的存储是不切实际的。在这种情况下，最好的防重放保护方法是让一个存储区负责某一 ticket，并拒绝该 ticket 在任何其他区的 0-RTT。这种方法可以防止攻击者进行简单的重放，因为只有一个区会接受 0-RTT 数据。 一个较弱的设计是为每个区实现单独的存储，但允许任何区的 0-RTT。 这种方法将重放的次数限制在每个区一次。 当然，无论哪种设计，应用消息复制仍然是可能的。

当刚启动时，只要其记录窗口的任何部分与启动时间重叠，就应该拒绝 0-RTT。 否则，它们就有可能接受最初在该期间发送的重放。

注意：如果客户端的时钟运行速度比服务器快得多，那么未来可能会收到一个在窗口之外的 ClientHello，在这种情况下，它可能会被接受为 1-RTT，导致客户端重试，之后再成为 0-RTT 可接受状态。 这是第8章中描述的第二种攻击形式的另一种变体。

### 8.3. Freshness Checks

因为 ClientHello 指示了客户端发送的时间，所以可以有效地判断一个 ClientHello 是否可能是最近合理发送的，对于这样的 ClientHello 只接受 0-RTT，否则回落到 1-RTT 的握手。 这对于 8.2 节描述的 ClientHello 存储机制来说是必要的，否则服务器需要存储无限的 ClientHello，对于独立的单次使用的 ticket 来说，这是一个有用的优化，因为可以高效地拒绝不能用于 0-RTT 的 ClientHello。

为了实现这一机制，服务器需要存储服务器生成会话 ticket 的时间，并加上客户端和服务器之间的往返时间估计。 即：

```
       adjusted_creation_time = creation_time + estimated_RTT
```

这个值可以在 ticket 中编码，从而避免为每个未完成的 ticket 保留状态。 服务器可以通过从客户端 pre_shared_key 扩展中的 obfuscated_ticket_age 参数中减去该 ticket 的 ticket_age_add 值来确定客户端对 ticket age 的看法。服务器可以确定 ClientHello 的 expected_arrival_time 为：

```
     expected_arrival_time = adjusted_creation_time + clients_ticket_age
```

当接收到一个新的 ClientHello 时，将 expected _arrival_time 与当前服务器时钟时间进行比较，如果两者相差超过一定量，则拒绝 0-RTT，不过可以允许 1-RTT 握手完成。

有几个潜在的错误来源可能会导致 expected_arrival_time 和测量时间不匹配。客户端和服务器时钟速率的不同可能是影响最小的，虽然绝对时间可能会有较大的偏差。网络传输延迟是导致经过时间的合法值不匹配的最可能原因。NewSessionTicket 和 ClientHello 消息都可能被重传，因此会有延迟，这可能会被 TCP 隐藏。对于互联网上的客户端来说，这意味着十秒左右的窗口，以考虑时钟错误和测量不同；其他部署场景可能有不同的需求。时钟偏移分布不是对称的，因此最佳的权衡可能涉及到一个不对称的允许偏差。

请注意，仅靠时新性检查不足以防止重放，因为它没有在错误窗口期间检测到重放，而这一窗口 -- 取决于带宽和系统容量 -- 在现实世界中可能包括数十亿次重放。 此外，这种时新性检查只在接收 ClientHello 时进行，而不是在接收后续早期应用数据记录时进行。 在接受早期数据后，记录可能会在较长的时间内继续流向服务器。

## 9. Compliance Requirements

### 9.1. Mandatory-to-Implement Cipher Suites

在没有应用配置标准规定的情况下：

- 一个符合 TLS 标准的应用程序必须实现 TLS_AES_128_GCM_SHA256 [GCM] 密码套件，并且应该实现 TLS_AES_256_GCM_SHA384 [GCM] 和 TLS_CHACHA20_POLY1305_SHA256 [RFC8439] 密码套件（见附录 B.4）。
- 一个符合 TLS 标准的应用程序必须支持使用 rsa_pkcs1_sha256 (用于证书)、rsa_pss_rsae_sha256 (用于 CertificateVerify 和证书)和 ecdsa_secp256r1_sha256 的数字签名。 一个符合 TLS 标准的应用程序必须支持与 secp256r1(NIST P-256) 的密钥交换，并且应该支持与 X25519 [RFC7748] 的密钥交换。

### 9.2. Mandatory-to-Implement Extensions

在没有应用配置标准的情况下，一个符合TLS标准的应用必须实现以下TLS扩展：

- 支持的版本 (supported_versions， 4.2.1 节)
- Cookie (cookie，4.2.2 节)
- 签名算法 (signature_algorithms，4.2.3 节)
- 签名算法证书 (signature_algorithms_cert，4.2.3 节)
- 协商组（supported_groups，4.2.7节）。
- 共享秘钥（key_share，4.2.8 节）。
- 服务器名称(server_name，[RFC6066] 第 3 节)

所有的实现在提供适用功能时必须发送和使用这些扩展：

- supported_versions 对于所有的 ClientHello, ServerHello, 和 HelloRetryRequest 消息是必需的。
- signature_algorithms 对于证书认证来说是必须的。
- supported_groups 对于使用 DHE 或 ECDHE 密钥交换的 ClientHello 消息是必须的。
- key_share 对于使用 DHE 或 ECDHE 密钥交换的 ClientHello 消息是必须的。
- pre_shared_key 对于 PSK 密钥协议来说是必须的。
- psk_key_exchange_modes 是对 PSK 密钥协议的必须的。

如果 ClientHello 在其正文中包含有 0x0304 的 supported_versions 扩展，则认为客户端试图使用本规范进行协商。 这样的 ClientHello 消息必须满足以下要求：

- 如果不包含 pre_shared_key 扩展，则必须同时包含 signature_algorithms 扩展和 supported_groups 扩展。
- 如果包含一个 supported_groups 扩展，则必须同时包含一个 key_share 扩展，反之亦然。允许使用空的 KeyShare.client_shares 向量。

服务器接收到不符合这些要求的 ClientHello 必须用 missing_extension alert 中止握手。

此外，所有的实现都必须支持 server_name 扩展，应用程序必须能够使用该扩展。服务器可以要求客户端发送一个有效的 server_name 扩展名。需要这个扩展的服务器应该对缺少 server_name 扩展的 ClientHello 响应以 missing_extension 警告，来终止连接。

### 9.3. Protocol Invariants

本节描述了 TLS 端点和中间件必须遵循的 invariant。它也适用于 TLS 的早期版本。

TLS 被设计为安全和可兼容的扩展。新的客户端或服务器，在与新对端通信时，应该协商最优先的通用参数。 TLS 握手提供降级保护。中间件在没有终止 TLS 的情况下，在新的客户端和新的服务器之间传递流量，应该无法影响握手（见附录 E.1）。同时，部署以不同的速度更新，所以新的客户端或服务器可能会继续支持旧的参数，这将允许它与旧的端点互操作。

为了使之工作，实现必须正确处理可扩展字段：

- 客户端发送的 ClientHello 必须支持其中所有的参数。否则，服务器可能会因为选择了其中的一个参数而导致互操作失败。
- 接收 ClientHello 的服务器必须正确地忽略所有不认识的密码套件、扩展和其他参数。否则，它可能无法与新的客户端进行互操作。在 TLS 1.3 中，接收到 CertificateRequest 或 NewSessionTicket 的客户端也必须忽略所有不认识的扩展。
- 终止 TLS 连接的中间件必须作为一个合规的 TLS 服务器（对原客户端），包括拥有客户端愿意接受的证书，同时也必须作为一个合规的 TLS 客户端（对原服务器），包括验证原服务器的证书。 特别是，它必须生成自己的 ClientHello，只包含自己理解的参数，它必须生成一个更新的 ServerHello 随机值，而不是转发端点的值。
> 请注意，TLS 的协议要求和安全分析只适用于两个单独的连接。 安全部署 TLS 终结器需要额外的安全考虑，这超出了本文的范围。
- 转发不理解的 ClientHello 参数的中间件不得处理该 ClientHello 以外的任何消息。它必须不加修改地转发所有后续流量。否则，可能无法与较新的客户端和服务器互通。

转发的 ClientHello 可能包含中间件不支持的功能通告，因此响应可能包含中间件不识别的未来 TLS 添加物。 这些附加功能可能会任意改变 ClientHello 之外的任何消息。 特别是，ServerHello 中发送的值可能会改变，ServerHello 格式可能会改变，TLSCiphertext 格式可能会改变。

TLS 1.3 的设计受到了广泛部署的不兼容的 TLS 中间件的限制（见附录 D.4）; 但是，它并没有放松 invariant。这些中间件仍然是不合规的。
