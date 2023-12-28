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

握手失败或其它协议错误会触发连接中止，在这之前可以有选择地发送一个警报消息（第6章）。

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

如果 client 没有提供足够的 "key_share" 扩展（例如，只包含 server 不接受或不支持的 DHE 或 ECDHE 组），server 会使用 HelloRetryRequest 来纠正这个不匹配问题，client 需要使用一个合适的 "key_share" 扩展来重启握手，如图 2 所示。如果没有通用的密码参数能够协商，server 必须使用一个适当的警报来中止握手。

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

变长向量的定义是通过指定一个合法长度的子范围来实现（使用符号 <floor..ceiling> ）。当这些被编码时，在字节流中实际长度是在向量的内容之前。这个长度会以一个数字的形式出现，并使用足够多的字节以便表示向量的最大长度（ceiling）。一个实际长度是 0 的变长向量会被当做一个空向量。

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

协议消息必须以 4.4.1 中定义的顺序发送，这也已经在第 2 章的图中展示了。对端如果收到了不按顺序发送的握手消息必须使用一个 "unexpected_message" 警报来中止握手。

新的握手消息类型已经由 IANA 指定并在第 11 章中描述。

### 4.1. Key Exchange Messages

密钥交换消息用于确定 client 和 server 之间的安全能力，并确定包括流量密钥在内的共享密钥来保护其余的握手消息和数据。

#### 4.1.1. Cryptographic Negotiation

在TLS中，密码协商通过 client 在 ClientHello 中提供下面 4 个选项集合来实现：

- 一个密码族列表，指的是 client 所支持的 AEAD 算法或 HKDF hash 对。
- 一个 "supported_groups" (4.2.7) 扩展，指的是 client 支持的 (EC)DHE 组；一个 "key_share" (4.2.8) 扩展，包含了一些或全部组所共享的 (EC)DHE 秘钥。
- 一个 "signature_algorithms" (4.2.3) 扩展，指的是 client 能接受的签名算法；可能会添加一个 "signature_algorithms_cert" 扩展（4.2.3）来指明证书指定的签名算法。
- 一个 "pre_shared_key" (4.2.11) 扩展，包含了一个 client 知晓的对称密钥；一个 "psk_key_exchange_modes" (4.2.9) 扩展，表明与 PSK 一起使用的密钥交换模式。

如果 server 没有选择PSK，则这些选项的前 3 个是完全正交的：server 独立地选择一个密码族，一个 (EC)DHE 组和用于确定密钥的密钥共享，一个签名算法或证书对用于认证自己和 client。如果接收到的 "supported_groups" 和 server 所支持的组之间没有重叠，则 server 必须用一个 "handshake_failure" 或一个 "insufficient_security" 警报中止握手。

如果 server 选择了一个 PSK，则必须从 client 的 "psk_key_exchange_modes" 扩展（目前是仅 PSK 或带 (EC)DHE）所携带的集合中选择一个密钥建立模式。需要注意的是如果可以不带 (EC)DHE 就使用 PSK，则 "supported_groups" 参数不重叠不一定是致命的，就像之前讨论过的非 PSK 场景一样。

如果 server 选择了一个 (EC)DHE 组并且 client 没有在初始 ClientHello 中提供兼容的 "key_share" 扩展，server 必须响应 HelloRetryRequest (4.1.4) 消息。

如果 server 成功地选择了参数且没有发送 HelloRetryRequest，表明 ServerHello 中所选的参数如下：

- 如果使用 PSK，则 server 会发送一个 "pre_shared_key" 扩展表明所选的密钥。
- 使用 (EC)DHE 时，server 也会提供 "key_share" 扩展。如果没有使用 PSK，则一直使用 (EC)DHE 和基于证书的认证。
- 当通过证书进行验证时，server 将会发送 Certificate(4.4.2) 和 CertificateVerify(4.4.3) 消息。在本文定义的 TLS 1.3 中，会一直使用一个 PSK 或一个证书，但两者不会同时使用。将来的文档可能会定义怎样同时使用它们。

如果 server 不能协商出支持的参数集合（例如，client 和 server 的参数集合没有重叠），它必须用一个 "handshake_failure" 或一个 "insufficient_security" 警报（见第6章）中止握手。

#### 4.1.2. Client Hello

当 client 第一次连接一个 server 时，它需要发送 ClientHello 作为第一个消息。当 server 用 HelloRetryRequest 来响应 client 的 ClientHello 时，client 也应当发送 ClientHello。这种条件下，client 必须发送相同的 ClientHello（无修改），除非：

- 如果 HelloRetryRequest 带有一个 "key_share" 扩展，则将共享列表用包含指定组中的一个 KeyShareEntry 的列表取代。
- 如果存在 "early_data" 扩展则将其移除。Early data 不允许在 HelloRetryRequest 之后出现。
- 如果 HelloRetryRequest 中提供了一个 "cookie" 扩展，则需要也包含一个 "cookie" 扩展。
- 如果需要重新计算 "obfuscated_ticket_age" 和绑定值、(可选地)删除与 server 指定的密码族不兼容的任何 PSK，则更新 "pre_shared_key" 扩展。
- 选择性增加、删除或更改 "padding" 扩展 [RFC7685] 的长度。
- 将来定义的其他 HelloRetryRequest 中扩展允许的修改。

由于 TLS 1.3 禁止重协商，如果 server 已经协商完成了 TLS 1.3，在任何其它时间收到了 ClientHello，必须用 "unexpected_message" 警报中止连接。

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

- **legacy_version**：在以前版本的 TLS 里，这个字段用于版本协商和表示 client 支持的最高版本号。经验表明很多 server 没有适当地实现版本协商，导致 “版本容忍”，会使 server 拒绝本来可接受的版本号高于支持的ClientHello。在 TLS 1.3 中，client 在 "supported_versions" 扩展(4.2.1节)中表明版本偏好，且 legacy_version 字段必须设置为 TLS 1.2 的版本号 0x0303。TLS 1.3 ClientHello 的 legacy_version 为 0x0303，supported_versions 扩展值为最高版本 0x0304（关于后向兼容的细节见附录 D）
- **random**： 由一个安全随机数生成器产生的 32 字节随机数，更多信息见附录 C。
- **legacy_session_id**： 之前的 TLS 版本支持 "会话恢复" 特性，在 TLS 1.3 中此特性与预共享密钥合并了(见 2.2 节)。client 需要将此字段设置为 TLS 1.3 之前版本的 server 缓存的 session ID。在兼容模式下(见附录 D.4)此字段必须非空，所以一个不能提供 TLS 1.3 之前版本会话的 client 必须产生一个 32 字节的新值。这个值不必是随机的但应该是不可预测的以避免实现上固定为一个具体的值(也被称为僵化)。否则，它必须被设置为一个 0 长度向量(例如，一个 0 字节长度字段)。
- **cipher_suites**： client 支持的对称密码族选项列表，具体有记录保护算法（包括密钥长度）和与 HKDF 一起使用的 hash 算法，这些算法以 client 偏好降序排列。值定义在附录 B.4。如果列表中包含 server 不认识，不支持，或不想使用的密码族，server 必须忽略这些密码族并且正常处理其余的密码族。如果 client 试图确定一个 PSK，应该通告至少一个密码族以表明 PSK 关联 hash 算法。
- **legacy_compression_methods**：TLS 1.3 以前的版本支持压缩，并提供一个支持的压缩方法列表。对于 TLS 1.3 的每个ClientHello，这个向量必须只包含 1 字节并设置为 0，对应以前 TLS 版本的 "null" 压缩方法。如果收到的 TLS 1.3 ClientHello 这个字段是其它的值，server 必须以一个 "illegal_parameter" alert 来中止握手。请注意，TLS 1.3 server 可能会接收 TLS 1.2 或之前的 ClientHello 包含其它压缩方法，（如果与这样一个先前版本协商）必须遵循适当的 TLS 先前版本的流程。
- **extension**： Client 通过在 extension 字段发送数据来向 server 请求扩展的功能。实际的 "Extension" 格式定义在 4.2 节中。TLS 1.3 强制使用特定的 extension，因为一些功能被转移到了 extension 里以保留 ClientHello 与先前 TLS 版本的兼容性。Server 必须忽略无法识别的 extension。

所有版本的 TLS 都允许 compression_method 字段后跟着 extension 字段。TLS 1.3 ClientHello 消息始终包含 extension（至少包含 "supported_versions"，否则将被当做 TLS 1.2 ClientHello消息）。但是，TLS1.3 server 可能会从以前版本的 TLS 接收不带 extension 字段的 ClientHello 消息。可以通过 ClientHello 末尾 compression_methods 字段后面是否有字节来检测扩展的存在。注意，这种检测可选数据的方法不同于检测可变长度字段的常规 TLS 方法，但是可用于与 extension 定义之前的 TLS 版本兼容。TLS 1.3 server 需要首先执行此检查，并且只有存在 "supported_versions" 扩展时才尝试协商 TLS 1.3。如果协商 TLS 1.3 之前的版本，server 必须检查消息是否在 legacy_compression_methods 之后不包含任何数据，或者是否包含一个后面没有数据的有效扩展块。如果不是，则必须以 "decode_error" alert 中止握手。

如果 client 使用 extension 请求附加功能，而 server 不提供此功能，则 client 可能会中止握手。

client 发送 ClientHello 消息后，等待 ServerHello 或 HelloRetryRequest 消息。如果正在使用 early data，则 client 可以在等待下一个握手消息时传输早期应用数据（2.3节）。

#### 4.1.3. Server Hello

#### 4.1.4. Hello Retry Request

### 4.2. Extensions

#### 4.2.1. Supported Versions

#### 4.2.2. Cookie

#### 4.2.3. Signature Algorithms

#### 4.2.4. Certificate Authorities

#### 4.2.5. OID Filters

#### 4.2.6. Post-Handshake Client Authentication

#### 4.2.7. Supported Groups

#### 4.2.8. Key Share

##### 4.2.8.1. Diffie-Hellman Parameters

##### 4.2.8.2. ECDHE Parameters

#### 4.2.9. Pre-Shared Key Exchange Modes

#### 4.2.10. Early Data Indication

#### 4.2.11. Pre-Shared Key Extension

##### 4.2.11.1. Ticket Age

##### 4.2.11.2. PSK Binder

##### 4.2.11.3. Processing Order

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



