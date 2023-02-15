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

### 4.1. Frame Format

### 4.2. Frame Size

### 4.3. Field Section Compression and Decompression

#### 4.3.1. Compression State

## 5. Streams and Multiplexing

### 5.1. Stream States

#### 5.1.1. Stream Identifiers

#### 5.1.2. Stream Concurrency

### 5.2. Flow Control

#### 5.2.1. Flow-Control Principles

#### 5.2.2. Appropriate Use of Flow Control

#### 5.2.3. Flow-Control Performance

### 5.3. Prioritization

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
