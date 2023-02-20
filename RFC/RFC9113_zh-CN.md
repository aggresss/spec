# HTTP/2

> 原文 [https://www.rfc-editor.org/rfc/rfc9113](https://www.rfc-editor.org/rfc/rfc9113)

## 摘要

## 1. Introduction

使用超文本传输协议(HTTP， [HTTP])的应用程序的性能与每个HTTP版本如何使用底层传输协议以及传输协议运行的条件有关。

发送多个并发请求可以减少延迟并提高应用程序性能。HTTP/1.0 在给定的 TCP 连接上只允许在同一时间有一个未完成的请求。HTTP/1.1 增加了请求管道，但这只是部分解决了请求并发，仍然会受到应用层队首阻塞 (head of line blocking) 的影响。因此，HTTP/1.0 和 HTTP/1.1 客户端使用多个到服务器的连接来发起并发请求。

此外，HTTP 字段通常是重复和冗长的，这会导致不必要的网络流量，并导致初始 TCP 拥塞窗口很快被填满。当在一个新的 TCP 连接上发起多个请求时，可能会导致过大的延迟。

HTTP/2 通过定义一个 HTTP 语义到底层连接的优化映射来解决这些问题。具体来说，它允许在同一个连接上交叉消息，并对 HTTP 字段使用高效编码。它还允许对请求进行优先级排序，让更重要的请求更快地完成，进一步提高性能。

最终的协议对网络更友好，因为与 HTTP/1.x 相比，可以使用更少的 TCP 连接。这意味着与其他流的竞争更少，连接寿命更长，从而更好地利用可用的网络容量。不过要注意，这个协议没有解决 TCP 队首阻塞问题。

最后，HTTP/2 还通过使用二进制消息分帧实现了更高效的消息处理。

这份文件淘汰了 [RFC7540] 和 [RFC8740]。附录 B 列出了值得注意的变化。

## 2. HTTP/2 Protocol Overview

HTTP/2 为 HTTP 语义提供了优化的传输。HTTP/2 支持 HTTP 的所有核心特性，但目标是比 HTTP/1.1 更高效。

HTTP/2 是一个运行在 TCP 连接( [TCP] )上的面向连接的应用层协议。客户端是 TCP 连接的发起者。

HTTP/2 的基本协议单位是帧(frame 4.1节)。每种帧类型都有不同的用途。例如，HEADERS 和 DATA 帧构成了 HTTP 请求和响应的基础(第 8.1 节);其他帧类型如 SETTINGS、WINDOW_UPDATE 和 PUSH_PROMISE 用于支持其他 HTTP/2 特性。

请求的多路复用是通过让每个 HTTP 请求/响应 都与自己的流相关联来实现的(第 5 节)。流在很大程度上是彼此独立的，因此阻塞或停滞的请求或响应不会影响其他流的处理。

复用的有效利用依赖于流量控制和优先级。流量控制(第 5.2 节)通过将传输的数据限制为接收端能够处理的数据，确保能够有效地使用多路复用的流。确定优先级(第 5.3 节)可以确保有限的资源得到最有效的利用。HTTP/2 的这个修订版本废弃了 [RFC7540] 中的优先级信令方案。

因为连接中使用的 HTTP 字段可能包含大量冗余数据，所以包含这些数据的帧会被压缩(第 4.3 节)。通常情况下，这对请求长度有特别有利的影响，允许将多个请求压缩到一个分组中。

最后，HTTP/2 增加了一个新的可选交互模式，服务器可以将响应推送给客户端(8.4 节)。它的目的是允许服务器根据预期向客户端发送数据，以牺牲一些网络开销换取潜在的延迟增益。服务器通过合成一个请求来实现这一点，并将其作为 PUSH_PROMISE 帧发送。然后，服务器能够在单独的流上向合成请求发送响应。

### 2.1. Document Organization

HTTP/2规范分为四个部分。

- Starting HTTP/2 (Section 3) 介绍了如何发起HTTP/2连接。
- Frame (Section 4) 和 Stream (Section 5) 描述了 HTTP/2 帧的结构和形成多路复用流的方式。
- Frame Definitions (Section 6) and Error Definitions (Section 7) 包括 HTTP/2 中使用的帧和错误类型的详细信息。
- HTTP Mappings (Section 8) 和 Additional Requirements (Section 9) 描述了如何使用帧和流表达 HTTP 语义。

虽然有些帧层和流层的概念与 HTTP 是隔离的，但本规范并没有定义一个完全通用的帧层。帧层和流层是根据HTTP的需求定制的。

### 2.2. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

所有数值都是按网络字节顺序排列的。除非另有说明，否则值是无符号的。根据需要提供十进制或十六进制的字面量。十六进制字面量以 "0x" 作为前缀，以区别于十进制字面量。

本规范使用 RFC9000 [QUIC] 1.3 节中描述的约定描述二进制格式。注意，这种格式使用网络字节序，高值位在低值位之前。

这里使用了下列术语。

- client: 发起HTTP/2连接的端点。客户端发送HTTP请求并接收HTTP响应。
- connection: 两个端点之间的传输层连接。
- connection error: 影响整个 HTTP/2 连接的错误。
- endpoint: 连接的客户端或服务器。
- frame: HTTP/2 连接中通信的最小单位，由首部和根据帧类型构造的可变长度的字节序列组成。
- peer: 一个端点。当讨论一个特定的端点时，“peer”指的是与主要讨论对象相隔较远的端点。
- receiver: 接收帧的端点。
- sender: 传输帧的端点。
- server: 接受 HTTP/2 连接的端点。服务器接收 HTTP 请求并发送 HTTP 响应。
- stream: HTTP/2 连接中的双向帧流。
- stream error: 单个 HTTP/2 流的错误。

最后，术语 "gateway"、"intermediary"、"proxy" 和 "tunnel" 的定义见 [HTTP] 的第 3.7 节。中间设备在不同时间同时充当客户端和服务器。

适用于消息体的术语 "content" 定义在 [HTTP] 章节 6.4 中。

## 3. Staring HTTP/2

生成 HTTP 请求的实现需要知道服务器是否支持 HTTP/2。

HTTP/2 使用 [HTTP] 第 4.2 节中定义的 "HTTP" 和 "https" URI scheme，与 HTTP/1.1 [HTTP/1.1] 具有相同的默认端口号。这些 URI 不包含上游服务器(客户端希望建立连接的直接端)支持的 HTTP 版本的任何指示。

对于 "HTTP" 和 "https" URI，确定是否支持 HTTP/2 的方法是不同的。"https" URI 的发现在 3.2 节中描述。HTTP/2 对 "HTTP" URI 的支持只能通过带外方式发现，并且需要事先了解 3.3 节所述的支持情况。

### 3.1. HTTP/2 Version Identification

本文档中定义的协议有两个标识符。基于其中任何一个标识符创建连接都意味着使用本文档中描述的传输、帧和消息语义。

- 字符串 "h2" 表示 HTTP/2 使用传输层安全协议(Transport Layer Security, TLS)的协议;参见 9.2 节。这个标识符在 TLS 应用层协议协商(ALPN)扩展 [TLS-ALPN] 字段中使用，在任何可以识别 TLS 上 HTTP/2 的地方也会使用。 "h2" 字符串被序列化为 ALPN 协议的两个字节序列: `0x68、0x32` 。
- "h2c" 字符串以前用作令牌，用于 HTTP 升级机制的 Upgrade 头字段( [HTTP] 章节 7.8 )。这种用法从未被广泛部署，并且被本文所弃用。这同样适用于 HTTP2-Settings 头字段，该字段在升级到 "h2c" 时被使用。

### 3.2. Starting HTTP/2 for "https" URIs

客户端使用 TLS [TLS13] 和 ALPN 扩展 [TLS-ALPN] 向 "https" URI 发出请求。

基于 TLS 的 HTTP/2 使用 "h2" 协议标识符。"h2c" 协议标识符不能由客户端发送或由服务器选择; "h2c" 协议标识符描述了不使用 TLS 的协议。

一旦 TLS 协商完成，客户端和服务器都必须发送 connection preface (章节3.4)。

### 3.3. Starting HTTP/2 with Prior Knowledge

客户端可以通过其他方式得知某个服务器支持 HTTP/2。例如，客户端可以配置服务器是否支持 HTTP/2。

客户端知道服务器支持 HTTP/2，就可以建立一个 TCP 连接，并在发送 HTTP/2 帧之后发送连接序言(章节3.4)。服务器可以通过连接序言来识别这些连接。这只对通过明文 TCP 建立 HTTP/2 连接有影响。基于 TLS 的 HTTP/2 连接必须使用 TLS [TLS-ALPN] 中的协议协商。

同样，服务器必须发送连接序言(章节3.4)。

如果没有额外的信息，先前对 HTTP/2 的支持并不是某台服务器在以后的连接中也会支持 HTTP/2 的强信号。例如，服务器配置可能会改变，集群服务器中不同实例的配置可能不同，或者网络条件可能会改变。

### 3.4. HTTP/2 Connection Preface

在 HTTP/2 中，每个端点都需要发送连接序言以最终确认使用的协议，并为 HTTP/2 连接建立初始设置。客户端和服务器分别发送不同的连接序言。

客户端连接序言以24个字节序列开始，用十六进制表示法表示为:

```
  0x505249202a20485454502f322e300d0a0d0a534d0d0a0d0a
```

也就是说，连接序言以字符串 `"PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"` 开始。这个序列后面必须跟着一个 SETTINGS 帧(章节 6.5)，它可以是空的。客户端发送客户端连接序言作为连接的第一个应用数据字节。

> 注意: 选择客户端连接序言是为了让大部分 HTTP/1.1 或 HTTP/1.0 服务器和中间设备不再尝试处理其他帧。请注意，这并没有解决 [TALKING] 中提出的问题。

服务器连接序言由一个可能为空的 SETTINGS 帧(章节 6.5)组成，它必须是服务器在 HTTP/2 连接中发送的第一个帧。

从另一端接收到的 SETTINGS 帧作为连接序言的一部分，必须在发送连接序言后确认(参见 6.5.3 节)。

为了避免不必要的延迟，允许客户端在发送客户端连接序言后立即向服务器发送额外的帧，而无需等待接收服务器连接序言。但是，需要注意的是，服务器连接序言设置帧可能包括一些必要的设置，这些设置可能会改变客户端预期与服务器通信的方式。在收到设置帧后，客户端被期望遵守任何已建立的设置。在某些配置中，服务器可以在客户端发送额外帧之前发送设置，从而避免这个问题。

客户端和服务器必须将无效的连接序言视为类型为协议错误的连接错误(章节 5.4.1)。在这种情况下，超时帧(第 6.8 节)可能会被忽略，因为无效的序言表明对等端没有使用 HTTP/2。

## 4. HTTP Frames

一旦建立了 HTTP/2 连接，终端就可以开始交换帧了。

### 4.1. Frame Format

所有的帧都以一个固定格式的 9 字节头开始，然后是一个可变长度的帧负载。

```
HTTP Frame {
  Length (24),
  Type (8),

  Flags (8),

  Reserved (1),
  Stream Identifier (31),

  Frame Payload (..),
}
```

帧头的字段定义如下:

- Length: 以字节为单位的无符号 24 位整数表示的帧负载的长度。大于 2^14 (16,384) 的值不能发送，除非接收方为 SETTINGS_MAX_FRAME_SIZE 设置了更大的值。 帧头的 9 个字节不包括在这个值中。
- Type: 帧的 8 位类型。帧类型决定了帧的格式和语义。第 6 节列出了本文档中定义的框架。实现必须忽略和丢弃未知类型的帧。
- Flags: 一个8位字段，用于指定帧类型的布尔标志。标志的语义与指定的帧类型相关。未使用的标志是那些对特定帧类型没有定义语义的标志。未使用的标志在接收时必须忽略，发送时必须保持未设置(0x00)。
- Reserved: 保留的 1 位字段。该比特位的语义未定义，在发送时必须保持未设置 (0x00)，在接收时必须忽略。
- Stream Identifier: 用无符号 31 位整数表示的流标识符(参见第 5.1.1 节)。0x00 这个值是为连接中作为整体关联的帧保留的，而不是单独的流。

帧负载的结构和内容完全取决于帧类型。

### 4.2. Frame Size

帧有效载荷的大小受限于接收端在 SETTINGS_MAX_FRAME_SIZE 设置中通告的最大大小。这个值可以是 2^14 (16,384) 到 2^24-1 (16,777,215) 之间的任意值，包括这个值。

所有的实现必须能够接收和处理长度不超过 2^14 个字节的帧，加上 9 个字节的帧头(章节 4.1)。描述帧大小时，帧头的大小不包括在内。

> 注意:某些类型的帧，如 PING(章节 6.7)，对允许的帧负载数据量有额外的限制。

如果一个帧超过 SETTINGS_MAX_FRAME_SIZE 定义的大小，超过为帧类型定义的任何限制，或者太小不能包含强制帧数据，端点必须发送一个错误码 FRAME_SIZE_ERROR。帧内的帧大小错误可能会改变整个连接的状态，必须被视为连接错误(章节 5.4.1);这包括任何带有字段块(章节 4.3 )的帧(即 HEADERS、PUSH_PROMISE 和 CONTINUATION)、SETTINGS帧，以及任何流标识符为 0 的帧。

端点没有义务使用帧中的所有可用空间。响应性可以通过使用小于允许的最大尺寸的帧来提高。发送大帧可能会导致发送时间敏感帧(如 RST_STREAM、WINDOW_UPDATE 或 PRIORITY )的延迟，如果传输大帧时被阻塞，则会影响性能。

### 4.3. Field Section Compression and Decompression

field 部分的压缩是将一组 field 行( [HTTP] 的 5.2 节)压缩成 field block 的过程。域段解压缩是将域块解码为一组域线的过程。HTTP/2 字段段压缩和解压缩的细节在 [compression] 中定义，由于历史原因，它将这些过程称为 header compression 和 header decompression 。

每个 field block 将 field 部分所有的 filed 压缩成一个 block。标题段还包括以伪标题字段( 8.3 节)的形式与消息关联的控制数据，这些字段使用与字段行相同的格式。

> 注意: RFC7540 [RFC7540] 使用术语 "header block" 取代了更通用的 "field block"。

字段块携带请求、响应、承诺请求和推送响应的控制数据和首部部分(见 8.4 节)。除了在 PUSH_PROMISE (章节 6.6)帧中包含的临时响应和请求外，所有这些消息都可以选择包含一个带有 trailer 节的字段块。

field section 是 field line 的集合。field block 中的每一条 field line 都携带一个值。序列化后的 field block 被分成一个或多个字节序列，称为 field block fragments。第一个字段块片段是在 HEADERS (章节 6.2)或 PUSH_PROMISE (章节 6.6) 的帧负载中传输的，每一个都可以后面跟着 CONTINUATION (章节 6.10) 帧来携带后续的字段块片段。

Cookie 首部字段 [Cookie] 由 HTTP 映射特殊处理(参见 8.2.3 节)。

接收端通过拼接其片段重组字段块，然后解压块以重构字段部分。

一个完整的字段部分包括:

- 一个 HEADERS 或 PUSH_PROMISE 帧，设置了 END_HEADERS 标志，或者
- 未设置 END_HEADERS 标志的 HEADERS 或 PUSH_PROMISE 帧，以及一个或多个延续帧，其中最后一个延续帧设置了 END_HEADERS 标志。

每个字段块被作为一个离散单元处理。字段块必须作为连续的帧序列传输，不能有任何其他类型或来自任何其他流的交叉帧。HEADERS 或 CONTINUATION 帧序列中的最后一帧设置了 END_HEADERS 标志。PUSH_PROMISE 或 CONTINUATION 帧序列中的最后一帧设置了 END_HEADERS 标志。这使得字段块在逻辑上等同于单个帧。

字段块片段只能作为 HEADERS、PUSH_PROMISE 或 CONTINUATION 帧的帧负载发送，因为这些帧携带的数据可以修改接收端维护的压缩上下文。终端接收到 HEADERS、PUSH_PROMISE 或 CONTINUATION 帧时，即使这些帧将被丢弃，也需要重新组装字段块并执行解压缩。如果接收方没有解压缩字段块，则必须以类型为 COMPRESSION_ERROR 的连接错误(章节5.4.1)终止连接。

字段块中的解码错误必须被视为类型为 COMPRESSION_ERROR 的连接错误(章节 5.4.1)。

#### 4.3.1. Compression State

字段压缩是有状态的。每个端点都有一个 HPACK 编码器上下文和一个 HPACK 解码器上下文，用于对连接上的所有字段块进行编码和解码。第 4 节定义了动态表，它是每个上下文的主要状态。

动态表的最大长度是由 HPACK 解码器设定的。终端通过其 HPACK 解码器上下文的 SETTINGS_HEADER_TABLE_SIZE 设置来传递数据长度。参见 6.5.2 节。在建立连接时，两端 HPACK 解码器和编码器的动态表大小都是从 4096 字节开始的，这是 SETTINGS_HEADER_TABLE_SIZE 设置的初始值。

使用 SETTINGS_HEADER_TABLE_SIZE 对最大值所做的任何修改，在端点确认设置时生效(6.5.3 节)。该端点的 HPACK 编码器可以将动态表设置为解码器设置的最大值之前的任意大小。HPACK 编码器通过动态表大小更新指令声明动态表的大小( [COMPRESSION] 章节 6.3)。

一旦终端确认 SETTINGS_HEADER_TABLE_SIZE 发生了变化，使得最大值小于动态表的当前大小，它的 HPACK 编码器就必须用一条动态表大小更新指令开始下一个字段块，该指令将动态表的大小设置为小于或等于缩减后的最大值;参见 [COMPRESSION] 第4.2节。如果一个端点没有以一个符合的动态表大小更新指令开始，那么它必须将在确认缩减到最大动态表大小之后的字段块视为类型为 COMPRESSION_ERROR 的连接错误(章节5.4.1)。

> 建议实现者注意，减少 SETTINGS_HEADER_TABLE_SIZE 的值不能广泛互操作。使用连接序将值减少到初始值 4096 以下，这在某种程度上得到了更好的支持，但在某些实现中可能会失败。

## 5. Streams and Multiplexing

"stream" 是一个独立的、双向的帧序列，在 HTTP/2 连接中在客户端和服务器之间交换。流有几个重要的特征。

- 一个 HTTP/2 连接可以包含多个并发打开的流，任何一个终端都可以交错使用来自多个流的帧。
- 流可以单独建立和使用，也可以由任意一端共享。
- 两个端点都可以关闭流。
- 发送帧的顺序很重要。接收方按照接收帧的顺序处理帧。特别是，首部和数据帧的顺序在语义上很重要。
- 流由一个整数标识。流标识符由初始化流的端点分配给流。

### 5.1. Stream States

```
                                +--------+
                        send PP |        | recv PP
                       ,--------+  idle  +--------.
                      /         |        |         \
                     v          +--------+          v
              +----------+          |           +----------+
              |          |          | send H /  |          |
       ,------+ reserved |          | recv H    | reserved +------.
       |      | (local)  |          |           | (remote) |      |
       |      +---+------+          v           +------+---+      |
       |          |             +--------+             |          |
       |          |     recv ES |        | send ES     |          |
       |   send H |     ,-------+  open  +-------.     | recv H   |
       |          |    /        |        |        \    |          |
       |          v   v         +---+----+         v   v          |
       |      +----------+          |           +----------+      |
       |      |   half-  |          |           |   half-  |      |
       |      |  closed  |          | send R /  |  closed  |      |
       |      | (remote) |          | recv R    | (local)  |      |
       |      +----+-----+          |           +-----+----+      |
       |           |                |                 |           |
       |           | send ES /      |       recv ES / |           |
       |           |  send R /      v        send R / |           |
       |           |  recv R    +--------+   recv R   |           |
       | send R /  `----------->|        |<-----------'  send R / |
       | recv R                 | closed |               recv R   |
       `----------------------->|        |<-----------------------'
                                +--------+
```

- send: endpoint sends this frame
- recv: endpoint receives this frame
- H: HEADERS frame (with implied CONTINUATION frames)
- ES: END_STREAM flag
- R: RST_STREAM frame
- PP: PUSH_PROMISE frame (with implied CONTINUATION frames); state transitions are for the promised stream

请注意，该图只显示了流状态转换以及影响这些转换的帧和标志。在这方面，延续帧不会导致状态转换;它们实际上是它们遵循的 HEADERS 或 PUSH_PROMISE 的一部分。为了进行状态转换，END_STREAM 标志作为带有它的帧的单独事件处理。设置了 END_STREAM 标志的 HEADERS 帧可以导致两次状态转换。

在传输帧时，两端对流的状态都有主观的看法，可能会有所不同。端点不参与流的创建; 它们是由任何一个端点单方面创建的。状态不匹配的负面影响仅限于发送 RST_STREAM 后的“关闭”状态，此时帧可能会在关闭后一段时间内接收到。

流有以下几种状态:

**idle**

All streams start in the "idle" state.

从这个状态开始，下列状态转换是有效的:

- 作为客户端发送 HEADERS 帧，或者作为服务器接收 HEADERS 帧，会导致流变为 "open"。流标识符的选择如 5.1.1 节所述。同样的 HEADERS 帧也会导致流立即变为 "half-closed" 状态。
- 在另一个流上发送 PUSH_PROMISE 帧，会为以后使用预留空闲流。保留流的状态变为 "reserved (local)"。只有服务器可以发送 PUSH_PROMISE 帧。
- 在另一个流上接收 PUSH_PROMISE 帧会预留一个空闲流供以后使用。保留流的状态变为 "reserved (remote)"。只有客户端可以接收 PUSH_PROMISE 帧。
- 注意，PUSH_PROMISE 帧不会在空闲流上发送，而是在被承诺的流 ID 字段中引用新预留的流。
- 通过将流标识符位置 1 来打开流，会导致流立即过渡到 "closed" 状态; 注意，图中没有显示此转换。

在这种状态下，在流上接收到除报头或优先级以外的任何帧都必须被视为类型为协议错误的连接错误(章节 5.4.1)。如果该流是由服务器发起的，如章节 5.1.1 所述，那么接收到 HEADERS 帧也必须被视为类型为协议错误的连接错误(章节5.4.1)。

**reserved (local)**

处于 "reserved (local)" 状态的流是指已经发送了 PUSH_PROMISE 帧的流。PUSH_PROMISE 帧通过将流与远端发起的打开流关联起来来保留一个空闲流(参见 8.4 节)。

在这种状态下，只有以下转换是可能的:

- 端点可以发送 HEADERS 帧。这会导致流以 "reserved (remote)" 状态打开。
- 任何一个终端都可以发送 RST_STREAM 帧来导致流变为 "closed"。这释放了流预留。

在这种状态下，终端不能发送除 HEADERS、RST_STREAM 或 PRIORITY 以外的任何类型的帧。

在这种状态下，可能会接收到 PRIORITY 或 WINDOW_UPDATE 帧。在这种状态下，在流上接收到除 RST_STREAM、PRIORITY 或 WINDOW_UPDATE 以外的任何类型的帧都必须被视为类型为 PROTOCOL_ERROR 的连接错误(章节 5.4.1)。

**reserved (remote)**

处于 "reserved (remote)" 状态的流已经被远程对等端保留了。

在这种状态下，只有以下转换是可能的:

- 接收到 HEADERS 帧会导致流变为 "half-closed (local)" 状态。
- 任何一个终端都可以发送 RST_STREAM 帧来导致流变为 "closed"。这释放了流预留。

在这种状态下，终端不能发送除 RST_STREAM、WINDOW_UPDATE 或 PRIORITY 以外的任何类型的帧。

在这种状态下，流上接收到除 HEADERS、RST_STREAM 或 PRIORITY 以外的任何类型的帧都必须被视为类型为 PROTOCOL_ERROR 的连接错误(章节 5.4.1)。

**open**

处于 "open" 状态的流可以被两端用来发送任何类型的帧。在这种状态下，发送端遵守被通告流级别的流量控制限制(章节 5.2)。

在这种状态下，任何一端都可以发送带有 END_STREAM 标志的帧，这会导致流过渡到 "helf-closed" 状态之一。终端发送 END_STREAM 标志会导致流状态变为 "half-closed (local)"; 终端接收到 END_STREAM 标志会导致流状态变为 "half-closed (remote)"。

任何一端都可以从这个状态发送一个 RST_STREAM 帧，使其立即过渡到 "closed" 状态。

**half-closed (local)**

处于 "half-closed (local)" 状态的流不能用于发送帧，只能用于 WINDOW_UPDATE、PRIORITY 和 RST_STREAM。

当接收到设置了 END_STREAM 标志的帧或任何一端发送了 RST_STREAM 帧时，流从这种状态转换为 "closed" 状态。

在这种状态下，终端可以接收任何类型的帧。使用 WINDOW_UPDATE 帧提供流量控制信用值对于继续接收流量控制帧是必要的。在这种状态下，接收端可以忽略 WINDOW_UPDATE 帧，因为 WINDOW_UPDATE 帧可能会在发送带有 END_STREAM 标志的帧之后的一小段时间内到达。

在这个状态下可以接收到 PRIORITY 帧。

**half-closed (remote)**

"half-closed (remote)" 的流不再被对等端用来发送帧。在这种状态下，终端不再需要维护接收者流量控制窗口。

如果终端接收到 WINDOW_UPDATE、PRIORITY 或 RST_STREAM 以外的其他帧，并且流处于这种状态，它必须响应一个类型为 STREAM_CLOSED 的流错误(章节 5.4.2)。

"half-closed (remote)" 的流可以被终端用来发送任何类型的帧。在这种状态下，端点会继续遵守通告流级别的流量控制限制(章节5.2)。

流可以通过发送设置了 END_STREAM 标志的帧或者任何一端发送 RST_STREAM 帧来从这种状态过渡到 "closed" 状态。

**closed**

"closed" 状态是终端状态。

在终端发送和接收设置了 END_STREAM 标志的帧后，流进入 "closed" 状态。在终端发送或接收 RST_STREAM 帧后，流也会进入 "closed" 状态。

终端不能在已关闭的流上发送除 PRIORITY 以外的帧。终端可以将在已关闭流上接收到的任何其他类型的帧视为类型为 STREAM_CLOSED 的连接错误(章节 5.4.1)，除非如下所述。

一个发送带有 END_STREAM 标志的帧或 RST_STREAM 帧的终端可能会从它的对等端接收到 WINDOW_UPDATE 或 RST_STREAM 帧，这可能是在它接收到并处理关闭流的帧之前的时间。

在处于 "open" 或 "half-closed (local)" 状态的流上发送 RST_STREAM 帧的端点可以接收任何类型的帧。在处理 RST_STREAM 帧之前，端可能已经发送或排队等待发送这些帧。终端必须最低限度地处理然后丢弃在这种状态下接收到的任何帧。这意味着更新 HEADERS 和 PUSH_PROMISE 帧的报头压缩状态。接收到 PUSH_PROMISE 帧也会导致被承诺流变为 "reserved (remote)"，即使是在已关闭的流上接收到 PUSH_PROMISE 帧时也是如此。此外，数据框的内容计入连接流量控制窗口。

终端可以对所有处于 "closed" 状态的流执行这种最小的处理。终端可以使用其他信号来检测一端已经接收到导致流进入 "closed" 状态的帧，并将接收到的除 PRIORITY 以外的任何帧视为类型为协议错误的连接错误(章节5.4.1)。端点可以使用帧来指示另一端已经接收到关闭信号来驱动这一过程。端点不应该为此使用定时器。例如，终端在关闭流后发送 SETTINGS 帧，在收到设置确认后，可以安全地将该流上的 DATA 帧视为错误。其他可能会用到的功能包括 PING 帧、在关闭流后创建的流上接收数据，或者响应关闭流后创建的请求。

在没有更具体规则的情况下，实现应该将接收到的状态描述中没有明确允许的帧作为类型为协议错误的连接错误(章节5.4.1)处理。注意，PRIORITY 可以在任何流状态下发送和接收。

本节中的规则仅适用于本文档中定义的框架。接收到语义未知的帧不能被视为错误，因为发送和接收这些帧的条件也是未知的。参见 5.5 节。

HTTP 请求/响应交换的状态转换示例可以在 8.8 节中找到。在 8.4.1 节和 8.4.2 节可以找到服务器推送的状态转换示例。

#### 5.1.1. Stream Identifiers

流由一个 31 位无符号整数标识。由客户端发起的流必须使用奇数的流标识符;由服务器发起的流必须使用偶数流标识符。0 的流标识符 (0x00) 用于连接控制消息;流标识符 0 不能用于建立新的流。

新建立流的标识符必须在数字上大于起始端点已打开或保留的所有流。它管理使用报头帧打开的流和使用 PUSH_PROMISE 保留的流。接收到意外流标识符的端点必须响应类型为 PROTOCOL_ERROR 的连接错误(章节5.4.1)。

HEADER 帧将由帧头中的流标识符标识的客户端发起的流从 "idle" 转换为 "open"。PUSH_PROMISE 帧会将由承诺流 ID 字段在帧负载中标识的服务器启动流从 "idle" 转换为 "reserved (local)" 或 "reserved (remote)"。当流从 "idle" 状态转换出来时，所有处于 "idle" 状态的流，如果被流标识符值较低的对端打开，将立即转换为 "closed" 状态。也就是说，端点可以跳过流标识符，其效果是被跳过的流立即被关闭。

流标识符不能重用。长时间的连接可能导致端点耗尽可用的流标识符范围。无法建立新流标识符的客户端可以为新流建立新连接。无法建立新的流标识符的服务器可以发送超时帧，这样客户端就被迫为新的流打开一个新连接。

#### 5.1.2. Stream Concurrency

另一端可以在 SETTINGS 帧中使用 SETTINGS_MAX_CONCURRENT_STREAMS 参数(参见 6.5.2 节)来限制并发活动流的数量。最大并发流设置是特定于每个端点的，只适用于接收该设置的一端。也就是说，客户端指定服务器可以发起的最大并发流数，服务器指定客户端可以发起的最大并发流数。

处于 "open" 状态或处于 "half-closed" 状态的流计数为终端允许打开的最大流数。处于这三种状态的流都会计入 SETTINGS_MAX_CONCURRENT_STREAMS 设置中公布的限制。处于任何一种 "reserved" 状态的流都不计入流的限制。

端点不能超过其对等体设置的限制。端点收到报头帧导致其发布的并发流超过限制，必须将其视为类型为 PROTOCOL_ERROR 或 REFUSED_STREAM 的流错误(章节 5.4.2)。错误码的选择决定了端点是否希望启用自动重试(详细信息参见 8.7 节)。

如果终端希望 SETTINGS_MAX_CONCURRENT_STREAMS 的值小于当前打开的流数，可以关闭超过这个值的流，或者允许流完成。

### 5.2. Flow Control

使用流进行多路复用会引起 TCP 连接的争用，从而导致流阻塞。流量控制方案确保同一个连接上的流不会相互干扰。流量控制既用于单个流，也用于整个连接。

HTTP/2 通过使用 WINDOW_UPDATE 帧(章节6.9)提供流量控制。

#### 5.2.1. Flow-Control Principles

HTTP/2 流流量控制的目的是在不改变协议的情况下允许使用各种流量控制算法。HTTP/2 中的流量控制具有以下特点:

1. 流量控制是特定于连接的。HTTP/2 流量控制在单跳端点之间进行，而不是在整个端到端路径上进行。
2. 流量控制基于 WINDOW_UPDATE 帧。接收端通告他们准备在流上以及整个连接中接收多少个字节。这是一个基于信用的计划。
3. 流量控制是定向的，由接收器提供整体控制。接收方可以为每个流和整个连接设置所需的任何窗口大小。发送端必须遵守接收端的流量控制限制。客户端、服务器和中间设备都独立地将它们的流量控制窗口作为接收端通告，并在发送时遵守其他节点设置的流量控制限制。
4. 对于新流和整个连接，流量控制窗口的初始值都是 65,535 字节。
5. 帧类型决定了流控制是否适用于帧。在本文档中指定的帧中，只有 DATA 帧受流量控制;所有其他帧类型都不占用所通告的流量控制窗口的空间。这确保了重要的控制帧不会被流量控制阻塞。
6. 终端可以选择禁用自己的流量控制，但终端不能忽略来自另一端的流量控制信号。
7. HTTP/2 只定义了 WINDOW_UPDATE 帧的格式和语义(章节6.9)。这个文档没有规定接收方如何决定何时发送这个帧或它发送的值，也没有规定发送方如何选择发送数据包。实现可以选择满足需求的任何算法。

实现还负责发送请求和响应的优先级，选择如何避免请求的队首阻塞，以及管理新流的创建。它们的算法选择可以与任何流控制算法交互。

#### 5.2.2. Appropriate Use of Flow Control

流量控制是为了保护在资源限制下运行的端点。例如，一个代理需要在多个连接之间共享内存，也可能有一个缓慢的上游连接和一个快速的下游连接。流量控制解决了接收端无法在一个流上处理数据，但希望在同一个连接上继续处理其他流的情况。

不需要这个功能的部署可以通告一个最大大小的流量控制窗口 (231-1)，并且可以在接收到数据时发送一个 WINDOW_UPDATE 帧来维护这个窗口。这实际上禁用了对该接收端的流量控制。相反，发送端总是受制于接收端通告的流量控制窗口。

在资源受限(例如内存)的部署中，可以采用流量控制来限制一端可以消耗的内存数量。但是，请注意，如果在不知道 带宽*延迟乘积 的情况下启用流量控制，这可能导致可用网络资源的次优使用(参见 [RFC7323])。

即使完全了解当前的 带宽*延迟 乘积，实现流量控制也是很困难的。一旦数据可用，终端必须从 TCP 接收缓冲区读取并处理 HTTP/2 帧。如果不能及时读取关键帧(如 WINDOW_UPDATE)，可能会导致死锁。及时读取帧不会让终端面临资源耗尽攻击，因为 HTTP/2 流量控制限制了资源承诺。

#### 5.2.3. Flow-Control Performance

如果一个端点不能确保它的端在这个连接上始终有大于另一端 带宽*延迟乘积 的可用流量控制窗口空间，它的接收吞吐量将受到 HTTP/2 流量控制的限制。这将导致性能下降。

及时发送 WINDOW_UPDATE 帧可以提高性能。终端需要在提高接收吞吐量的需求和管理资源耗尽风险的需求之间取得平衡，在定义管理窗口大小的策略时，应该仔细注意 10.5 节。

### 5.3. Prioritization

在 HTTP/2 这样的多路复用协议中，优先分配带宽和计算资源给流是获得良好性能的关键。一个糟糕的优先级方案会导致 HTTP/2 性能不佳。如果 TCP 层没有并行，性能可能会比 HTTP/1.1 差得多。

一个好的优先级方案受益于上下文知识的应用，例如资源的内容，资源如何相互关联，以及这些资源将如何被对等节点使用。特别是，客户端可以掌握与服务器优先级相关的请求优先级信息。在这些情况下，让客户端提供优先级信息可以提高性能。

#### 5.3.1. Background on Priority in RFC 7540

#### 5.3.2. Priority Signaling in This Document

### 5.4. Error Handing

#### 5.4.1. Connection Error Handing

#### 5.4.2. Stream Error Handing

#### 5.4.3. Connection Termination

### 5.5. Extending HTTP/2

## 6. Frame Definitions

### 6.1. DATA

### 6.2. HEADERS

### 6.3. PRIORITY

### 6.4. RST_STREAM

### 6.5. SETTINGS

#### 6.5.1. SETTINGS Format

#### 6.5.2. Defined Settings

#### 6.5.3. Settins Synchronization

### 6.6. PUSH_PROMISE

### 6.7. PING

### 6.8. GOAWAY

### 6.9. WINDOW_UPDATE

#### 6.9.1. The Flow-Control Window

#### 6.9.2. Initial Flow-Control Window Size

#### 6.9.3. Reducing the Stream Window Size

### 6.10. CONTINUATION

## 7. Error Codes

## 8. Expressing HTTP Semantics in HTTP/2

### 8.1. HTTP Message Framing

#### 8.1.1. Malformed Messages

### 8.2. HTTP Fields

#### 8.2.1. Field Validity

#### 8.2.2. Connection-Specific Header Fields

#### 8.2.3. Compressing the Cookie Header Fields

### 8.3. HTTP Control Data

#### 8.3.1. Request Pseudo-Header Fields

#### 8.3.2. Response Pseudo-Header Fields

### 8.4. Server Push

#### 8.4.1. Push Requests

#### 8.4.2. Push Responses

### 8.5. The CONNECT Method

### 8.6. The Upgrade Header Field

### 8.7. Request Reliability

### 8.8. Examples

#### 8.8.1. Simple Request

#### 8.8.2. Simple Response

#### 8.8.3. Complex Request

#### 8.8.4. Response with Body

#### 8.8.5. Informational Responses

## 9. HTTP/2 Connections

### 9.1. Connection Management

#### 9.1.1. Connection Reuse

### 9.2. Use of TLS Features

#### 9.2.1. TLS 1.2 Features

#### 9.2.2. TLS 1.2 Cipher Suites

#### 9.2.3. TLS 1.3 Features
