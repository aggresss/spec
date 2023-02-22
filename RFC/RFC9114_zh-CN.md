# HTTP/3

> 原文 [https://www.rfc-editor.org/rfc/rfc9114](https://www.rfc-editor.org/rfc/rfc9114)

## 摘要

QUIC 传输协议有几个 HTTP 传输中需要的特性，例如流多路复用、每流流控制和低延迟连接建立。本文档描述了 HTTP 语义在 QUIC 上的映射。本文档还确定了 QUIC 包含的 HTTP/2 功能，并描述了如何将 HTTP/2 扩展移植到 HTTP/3。

## 1. Introduction

HTTP 语义 ( [ HTTP ] ) 用于 Internet 上范围广泛的服务。这些语义最常用于 HTTP/1.1 和 HTTP/2。HTTP/1.1 已用于各种传输层和会话层，而 HTTP/2 主要用于 TCP 上的 TLS。HTTP/3 通过新的传输协议支持相同的语义：QUIC。

### 1.1. Prior Versions of HTTP

HTTP/1.1 ( [ HTTP/1.1 ] ) 使用空格分隔的文本字段来传送 HTTP 消息。虽然这些交换是人类可读的，但使用空格进行消息格式化会导致解析复杂性和对变异行为的过度容忍。

因为 HTTP/1.1 不包含多路复用层，所以通常使用多个 TCP 连接来并行处理请求。然而，这对拥塞控制和网络效率有负面影响，因为 TCP 不在多个连接之间共享拥塞控制。

HTTP/2 ( [ HTTP/2 ] ) 引入了一个二进制框架和多路复用层，以在不修改传输层的情况下改善延迟。然而，由于 HTTP/2 的多路复用的并行性质对于 TCP 的丢失恢复机制是不可见的，丢失或重新排序的数据包会导致所有活动事务出现停顿，无论该事务是否直接受到丢失数据包的影响。

### 1.2. Delegation to QUIC

QUIC 传输协议结合了流多路复用和每个流的流量控制，类似于 HTTP/2 框架层提供的功能。通过在流级别提供可靠性和跨整个连接的拥塞控制，与 TCP 映射相比，QUIC 有能力提高 HTTP 的性能。QUIC还在传输层整合了 TLS 1.3 ( [ TLS ] )，提供与在 TCP 上运行 TLS 相当的机密性和完整性，并改进了 TCP 快速打开 ( [ TFO ] ) 的连接设置延迟。

本文档定义了 HTTP/3：HTTP 语义在 QUIC 传输协议上的映射，很大程度上借鉴了 HTTP/2 的设计。HTTP/3 依靠 QUIC 提供数据的机密性和完整性保护；对等认证；可靠、有序、按流传输。在将流生命周期和流量控制问题委托给 QUIC 时，每个流都使用了类似于 HTTP/2 框架的二进制框架。QUIC 包含一些 HTTP/2 功能，而其他功能则在 QUIC 之上实现。

QUIC 在[ QUIC-TRANSPORT ]中有描述。有关 HTTP/2 的完整描述，请参阅[ HTTP/2 ]。

## 2. HTTP/3 Protocol Overview

HTTP/3 使用 QUIC 传输协议和类似于 HTTP/2 的内部框架层为 HTTP 语义提供传输。

一旦客户端知道某个端点存在 HTTP/3 服务器，它就会打开 QUIC 连接。QUIC 提供协议协商、基于流的多路复用和流量控制。3.1 节描述了 HTTP/3 端点的发现 。

在每个流中，HTTP/3 通信的基本单位是帧（第 7.2 节）。每种帧类型都有不同的用途。例如，标题 和数据帧构成 HTTP 请求和响应的基础（第 4.1 节）。适用于整个连接的帧在专用控制流上传送.

使用 QUIC 流抽象执行请求的多路复用，这在[ QUIC-TRANSPORT ]的第 2 节中进行了描述。每个请求-响应对使用单个 QUIC 流。流彼此独立，因此一个流被阻塞或遭受数据包丢失不会阻止其他流的进展。

服务器推送是 HTTP/2 ( [ HTTP/2 ] ) 中引入的一种交互模式，它允许服务器在客户端发出指示请求的情况下将请求-响应交换推送到客户端。这会权衡网络使用与潜在的延迟增益。几个 HTTP/3 帧用于管理服务器推送，例如 PUSH_PROMISE, MAX_PUSH_ID, 和 CANCEL_PUSH.

与 HTTP/2 一样，请求和响应字段在传输时被压缩。因为 HPACK ( [ HPACK ] ) 依赖于压缩字段部分的有序传输 (QUIC 不提供保证)，HTTP/3 将 HPACK 替换为 QPACK ( [ QPACK ] )。QPACK 使用单独的单向流来修改和跟踪字段表状态，而编码字段部分引用表的状态而不修改它。

### 2.1. Document Organization

以下部分详细概述了 HTTP/3 连接的生命周期：

- “连接设置和管理”（第 3 节）介绍了如何发现 HTTP/3 端点和建立 HTTP/3 连接。
- “在 HTTP/3 中表达 HTTP 语义”（第 4 节）描述了如何使用帧来表达 HTTP 语义。
- “ Connection Closure ”（第 5 节）描述了 HTTP/3 连接是如何终止的，无论是优雅的还是突然的。

协议的细节和与传输的交互在后续部分中描述：

- “流映射和使用”（第 6 节）描述了 QUIC 流的使用方式。
- “HTTP 框架层”（第 7 节）描述了大多数流中使用的框架。
- “错误处理”（第 8 节）描述了如何处理和表达错误条件，无论是针对特定流还是针对整个连接。

最后几节提供了其他资源：

- “HTTP/3 的扩展”（第 9 节）描述了如何在未来的文档中添加新功能。
- 可以在 附录 A 中找到 HTTP/2 和 HTTP/3 之间更详细的比较。

### 2.2. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

This document uses the variable-length integer encoding from [QUIC-TRANSPORT].

The following terms are used:

- abort: An abrupt termination of a connection or stream, possibly due to an error condition.
- client: The endpoint that initiates an HTTP/3 connection. Clients send HTTP requests and receive HTTP responses.
- connection: A transport-layer connection between two endpoints using QUIC as the transport protocol.
- connection error: An error that affects the entire HTTP/3 connection.
- endpoint: Either the client or server of the connection.
- frame: The smallest unit of communication on a stream in HTTP/3, consisting of a header and a variable-length sequence of bytes structured according to the frame type.

Protocol elements called "frames" exist in both this document and [QUIC-TRANSPORT]. Where frames from [QUIC-TRANSPORT] are referenced, the frame name will be prefaced with "QUIC". For example, "QUIC CONNECTION_CLOSE frames". References without this preface refer to frames defined in Section 7.2.

- HTTP/3 connection: A QUIC connection where the negotiated application protocol is HTTP/3.
- peer: An endpoint. When discussing a particular endpoint, "peer" refers to the endpoint that is remote to the primary subject of discussion.
- receiver: An endpoint that is receiving frames.
- sender: An endpoint that is transmitting frames.
- server: The endpoint that accepts an HTTP/3 connection. Servers receive HTTP requests and send HTTP responses.
- stream: A bidirectional or unidirectional bytestream provided by the QUIC transport. All streams within an HTTP/3 connection can be considered "HTTP/3 streams", but multiple stream types are defined within HTTP/3.
- stream error:

An application-level error on the individual stream.

The term "content" is defined in Section 6.4 of [HTTP].

Finally, the terms "resource", "message", "user agent", "origin server", "gateway", "intermediary", "proxy", and "tunnel" are defined in Section 3 of [HTTP].

Packet diagrams in this document use the format defined in Section 1.3 of [QUIC-TRANSPORT] to illustrate the order and size of fields.

## 3. Connection Setup and Management

### 3.1. Discovering an HTTP/3 Endpoint

HTTP 依赖于权威响应的概念：在源服务器（或在其方向）发出响应消息时，在给定目标资源状态的情况下，已确定该响应是对该请求最合适的响应在目标 URI 中标识。[ HTTP ]的第 4.3 节讨论了为 HTTP URI 定位权威服务器。

“https” 方案将授权与证书的拥有相关联，客户端认为该证书对于由 URI 的授权组件标识的主机是值得信赖的。在 TLS 握手中收到服务器证书后，客户端 必须使用[ HTTP ]的第 4.3.4 节 中描述的过程验证该证书是否与 URI 的原始服务器匹配。如果证书无法针对 URI 的源服务器进行验证，则客户端 不得 认为该服务器是该源的权威服务器。

客户端 可以 尝试访问具有 “https” URI 的资源，方法是将主机标识符解析为 IP 地址，在指定端口上建立到该地址的 QUIC 连接（包括如上所述的服务器证书验证），并发送一个 HTTP/3 请求消息通过该安全连接将 URI 定向到服务器。除非使用某些其他机制来选择 HTTP/3，否则在 TLS 握手期间，令牌 “h3” 用于应用层协议协商（ALPN；参见 [ RFC7301 ]）扩展。

连接问题（例如，阻塞 UDP）可能导致无法建立 QUIC 连接；在这种情况下，客户端 应该 尝试使用基于 TCP 的 HTTP 版本。

服务器 可以 在任何 UDP 端口上提供 HTTP/3；替代服务广告总是包含一个显式端口，并且 URI 包含一个显式端口或与方案关联的默认端口。

#### 3.1.1. HTTP Alternative Services

HTTP 来源可以使用 “h3” ALPN 令牌通过 Alt-Svc HTTP 响应标头字段或 HTTP/2 ALTSVC 帧（[ ALTSVC ] ）通告等效 HTTP/3 端点的可用性。

例如，源可以在 HTTP 响应中指示 HTTP/3 在同一主机名的 UDP 端口 50781 上可用，方法是包含以下标头字段：

```
Alt-Svc: h3=":50781"
```

#### 3.1.2. Oter Schemes

### 3.2. Connection Establishment

### 3.3. Connection Reuse

## 4. Expressing HTTP Semantics in HTTP/3

### 4.1. HTTP Message Framing

#### 4.1.1. Request Cancellation and Rejection

#### 4.1.2. Malformed Requests and Responses

### 4.2. HTTP Fields

#### 4.2.1. Field Compression

#### 4.2.2. Header Size Constraints

### 4.3. HTTP Control Data

#### 4.3.1. Request Pseudo-Header Fields

#### 4.3.2. Response Pseudo-Header Fields

### 4.4. The CONNECT Method

### 4.5. HTTP Upgrade

### 4.6. Server Push

## 5. Connection Closure

### 5.1. Idle Connections

### 5.2. Connection Shutdown

### 5.3. Immediate Application Closure

### 5.4. Transport Closure

## 6. Stream Mapping and Usage

### 6.1. Bidirectional Streams

### 6.2. Unidirectional Streams

#### 6.2.1. Control Streams

#### 6.2.2. Push Streams

#### 6.2.3. Reserved Stream Types

## 7. HTTP Framing Layer

### 7.1. Frame Layout

### 7.2. Frame Definitions

### 7.2.1. DATA

### 7.2.2. HEADERS

### 7.2.3. CANCEL_PUSH

### 7.2.4. SETTINGS

### 7.2.5. PUSH_PROMISE

### 7.2.6. GOAWAY

### 7.2.7. MAX_PUSH_ID

### 7.2.8. Reserved Frame Types

## 8. Error Handling

### 8.1. HTTP/3 Error Codes

## 9. Extensions to HTTP/3

