# QUIC: A UDP-Based Multiplexed and Secure Transport

> 原文 [https://www.rfc-editor.org/rfc/rfc9000.html](https://www.rfc-editor.org/rfc/rfc9000.html)

## 摘要

本文档定义了 QUIC 传输协议的核心。QUIC 为应用程序提供流控制流，用于结构化通信、低延迟连接建立和网络路径迁移。QUIC 包括安全策略，可在一系列部署环境中确保机密性、完整性和可用性。随附的文档描述了集成 TLS 的密钥协商、丢包检测以及一个样例拥塞控制算法。

## 1. 概述

QUIC 是一种安全的通用传输协议。本文档定义了 QUIC 的 Version 1，它符合在 [QUIC-INVARIANTS] 中定义的 QUIC 的版本无关属性。

QUIC 是一种面向连接的协议，它在客户端和服务端之间创建有状态的交互。

QUIC 握手结合了加密参数和传输参数的协商。QUIC 集成了 TLS 握手 [TLS13]，尽管使用了定制的帧来保护数据包。TLS 和 QUIC 的集成在 [QUIC-TLS] 中有更详细的描述。握手的结构允许尽快交换应用程序数据。这包括通过某种形式的事先通信或配置，一个客户端可以立即发送数据(0-RTT)的选项。

端点通过交换 QUIC 数据包在 QUIC 中进行通信。大多数数据包包含帧，这些帧在端点之间携带控制信息和应用程序数据。QUIC 验证每个包的完整性，并在实际情况下尽可能多的加密每个包。QUIC 报文以 UDP 数据报 [UDP] 的形式携带，以便于在现有系统和网络中更好的部署。

应用协议通过流在 QUIC 连接上交换信息，流是字节的有序序列。可以创建两种类型的流:

- 双向流，允许两个端点发送数据;
- 单向流，它允许单个端点发送数据。

基于信用 (credit-based) 的机制用于限制流的创建和可发送的数据量。

QUIC 提供必要的反馈来实现可靠的传输和拥塞控制。在 [QUIC-RECOVERY] 的第 6 节中描述了一种检测和从数据丢失中恢复的算法。QUIC 依靠拥塞控制来避免网络拥塞。在 [QUIC-RECOVERY] 的第 7 节中描述了一个典型的拥塞控制算法。

QUIC 连接并不严格绑定到单个网络路径。连接迁移使用连接标识符允许连接转移到新的网络路径。在当前版本的 QUIC 协议中只有客户端能够迁移。这种设计还允许在网络拓扑或地址映射发生变化 (比如 NAT 重新绑定可能引起的变化) 后继续连接。

建立连接后，会提供多个终止连接的选项。应用程序可以管理一个安全的关闭，端点可以协商一个超时时间，错误可以导致立即断开连接，并且在一个端点失去状态后提供了一种无状态机制来终止连接。

### 1.1. 文档结构

本文档描述了 QUIC 核心协议，其结构如下:

- 流是 QUIC 提供基本服务的抽象
  - Section 02 描述流的核心概念
  - Section 03 提供流状态的参考模型
  - Section 04 概述了流量控制的操作
- 连接是 QUIC 端点通信的上下文
  - Section 05 描述连接的核心概念
  - Section 06 描述版本协商
  - Section 07 详细说明连接的建立过程
  - Section 08 描述地址验证和关键的拒绝服务缓解措施
  - Section 09 描述端点如何将连接迁移到新的网络路径
  - Section 10 列出终止打开连接的选项
  - Section 11 提供流和连接错误处理指南
- 数据包和帧是 QUIC 通信的基本单位
  - Section 12 描述数据包和帧的概念
  - Section 13 定义数据的传输、重传和确认模型
  - Section 14 指定数据报携带 QUIC 包大小的管理规则。
- 最后，QUIC 协议元素的编码细节描述如下
  - Section 15 (versions)
  - Section 16 (integer encoding)
  - Section 17 (packet headers)
  - Section 18 (transport parameters)
  - Section 19 (frames)
  - Section 20 (errors)

随附的文档描述了 QUIC 的丢失检测和拥塞控制 [QUIC-RECOVERY]，以及 TLS 和其他加密机制 [QUIC-TLS] 的使用。

本文档定义了 QUIC version 1，它符合 [QUIC-INVARIANTS] 中的协议不变量。

要参考 QUIC version 1，请引用本文档。如需参考 QUIC 有限的版本无关属性集可以引用 [QUIC-INVARIANTS]。

### 1.2. 术语和定义

本文档中的关键词“必须”、“不得”、“必需”、“应”、“不应”、“建议”、“不建议”、“可”和“可选”在所有大写字母出现时（如图所示）应按照 BCP 14[RFC2119] [RFC8174] 所述进行解释。

本文档中常用的术语描述如下:

本文档中常用的术语描述如下。

- `QUIC`: 本文档描述的传输协议。QUIC 是一个名称，而不是首字母缩略词。
- `Endpoint`: 通过生成、接收和处理 QUIC 报文来参与 QUIC 连接的实体。QUIC 中只有两种类型的端点:客户端和服务端。
- `Client`: 发起 QUIC 连接的端点。
- `Server`: 接受 QUIC 连接的端点。
- `QUIC packet`: 一个完整的 QUIC 处理单元，可以封装在 UDP 数据报中。一个或多个 QUIC 包可以封装在单个 UDP 数据报中。
- `Ack-eliciting packet`: 一种包含 ACK、PADDING 和 CONNECTION_CLOSE 以外帧的 QUIC 包。这导致收件人发送确认, 参考 13.2.1 节。
- `Frame`:  一种结构化的协议信息单位。有多种帧类型，每一种都承载着不同的信息。帧包含在 QUIC 包中。
- `Address`: 在不加限制的情况下，表示网络路径一端的 IP 版本、IP 地址和 UDP 端口号的元组。
- `Connection ID`: 一种标识符，用于标识端点上的 QUIC 连接。每个端点为对端选择一个或多个连接 ID，将其包含在发送给端点的数据包中。该值对于对端是不透明的。
- `Stream`: 一种在 QUIC 连接中由有序字节组成的单向或双向通道。一个 QUIC 连接可以同时携带多个流。
- `Application`: 使用 QUIC 发送和接收数据的实体。

本文档使用术语 “QUIC包”、“UDP数据报” 和 “IP包” 来指代各自协议的单位。也就是说，一个或多个 QUIC 包可以封装在 UDP 数据报中，而 UDP 数据报又封装在 IP 包中。

### 1.3. 符号约定

本文档中的数据包和框架图使用自定义格式。这种格式的目的是总结(而不是定义)协议元素。冗长定义了结构的完整语义和细节。

对复杂字段进行命名，然后后跟由一对匹配的大括号包围的字段列表。此列表中的每个字段由逗号分隔。

单个字段包括长度信息，以及关于固定值、可选性或重复的指示。每个字段使用以下符号约定，所有长度都以比特为单位:

- `x (A)`: 表示 x 为 1 比特长度
- `x (i)`: 表示 x 为可变长度编码的整数，参考 16 节
- `x (A..B)`: 表示 x 可以是从 A 到 B 的任意长度，可以省略 A 表示 0 位的最小值，可以省略 B 表示没有设置上限；这种格式的值总是以字节边界结束
- `x (L) = C`: 表示 x 有一个固定的 C 值；x 的长度由 L 来描述，L 可以使用上面的任何长度形式
- `x (L) = C..D`: 表示 x 的取值范围为 C 到 D(含)，长度为 L
- `[x (L)]`: 表示 x 是可选的，长度为 L
- `x (L) ...`: 表示 x 被重复 0 次或多次，并且每个实例的长度为 L

本文档使用网络字节序(big-endian)。字段从每个字节的高位开始放置。

按照惯例，单个字段通过使用复杂字段的名称引用复杂字段。

示例 1:

```quic
Example Structure {
  One-bit Field (1),
  7-bit Field with Fixed Value (7) = 61,
  Field with Variable-Length Integer (i),
  Arbitrary-Length Field (..),
  Variable-Length Field (8..24),
  Field With Minimum Length (16..),
  Field With Maximum Length (..128),
  [Optional Field (64)],
  Repeated Field (8) ...,
}
```

当在文中引用一个单位字段时，可以使用携带该字段的字节的值和该字段的值集来确定该字段的位置。例如，值0x80可以用来引用字节最重要位中的单位字段，例如图1中的 One-bit field。

## 2. Streams

QUIC 中的流为应用程序提供了一个轻量级的、有序的字节流抽象。流可以是单向的，也可以是双向的。

流可以通过发送数据来创建。与流管理相关的其他进程，如结束、取消和管理流控制，都被设计为施加最小的开销。例如，单个流帧(章节19.8)可以打开、携带数据和关闭流。流也可以是长期存在的，可以持续整个连接的持续时间。

流可以由任意端点创建，可以与其他流交叉并发发送数据，也可以取消。QUIC 不提供任何方法来确保不同流上字节之间的顺序。

QUIC 允许任意数量的流并发操作，并允许任意数量的数据在任何流上发送，受流控制约束和流限制，参考第 4 节。

### 2.1 流类型和标识符

流可以是单向的，也可以是双向的。单向流在一个方向上携带数据:从流的发起方到对端。双向流允许数据在两个方向上发送。

在连接中，流具有一个数字值标识，称为流 ID。流标识符是一个 62 位整数 (2^26-1)，对于连接上的所有流来说是唯一的。流 ID 使用变长整数编码，参考 16 节。QUIC 终端不能在连接中重用流 ID。

流 ID 的最低有效位 (0x01) 标识流的启动器。客户端发起的流有偶数的流 ID (位设置为 0)，而服务器发起的流有奇数的流 ID (位设置为 1)。

流 ID 的第二个最低有效位 (0x02) 用来区分双向流 (位设置为 0) 和单向流 (位设置为 1)。

因此，来自流 ID 的两个最低有效位将流标识为四种类型之一，如表 1 所示。

Bits|Stream Type
---|---
0x00|Client-Initiated, Bidirectional
0x01|Server-Initiated, Bidirectional
0x02|Client-Initiated, Unidirectional
0x03|Server-Initiated, Unidirectional

表 1 : 流 ID 类型

每种类型的流空间从最小值开始(分别从 0x00 到 0x03) 每种类型的连续流创建时，流 ID 数值递增。乱序使用的流 ID 会导致该类型的所有流 ID 编号较低的流也被打开。

### 2.2 发送和接收数据

STREAM 帧(19.8 节)封装了应用程序发送的数据。终端使用 STREAM 帧中的 Stream ID 和 Offset 字段来排列数据。

终端必须能够将流数据作为有序的字节流交付给应用程序。交付有序字节流要求端点缓冲任何无序接收的数据，最高可达通告的流控制限制。

QUIC 对无序传输的流数据没有特别的考虑。然而，实现可以选择提供按顺序向接收应用程序交付数据的能力。

终端可以在同一流偏移处多次接收流的数据。已经接收到的数据可以被丢弃。在给定偏移量处的数据如果被发送多次，绝对不能改变;终端可以将接收到的同一偏移量的不同数据作为类型为协议冲突的连接错误处理。

流是一种有序的字节流抽象，QUIC 看不到其他结构。流帧边界在数据传输、丢包后重传或在接收端交付给应用程序时不被期望保留。

终端在没有确保在对端设置的流控制范围内的情况下，绝不能在任何流上发送数据。流量控制将在第4节中详细描述。

### 2.3. 流的优先级

如果资源分配给流的优先级正确，流复用可以对应用程序的性能产生显著的影响。

QUIC 不提供交换优先级信息的机制。相反，它依赖于从应用程序接收优先级信息。

QUIC 实现应该提供应用程序可以指示流的相对优先级的方法。实现使用应用程序提供的信息来确定如何将资源分配给活动流。

### 2.4． 操作流

本文档没有为 QUIC 定义 API；相反，它在流上定义了一组应用协议可以依赖的函数。应用程序协议可以假定 QUIC 实现提供了包含本节中描述的操作的接口。设计用于特定应用程序协议的实现可能只提供该协议使用的那些操作。

在流的发送部分，应用协议可以:

- 写数据，理解何时流流控制信用(章节4.1)已成功保留以发送写的数据;
- 结束流(干净终止)，导致一个设置FIN位的流帧(章节19.8);和
- 重置流(突然终止)，如果流还没有处于终端状态，就会产生RESET_STREAM帧(章节19.4)。

在流的接收部分，应用协议可以:

- 读取数据
- 终止读取流并请求关闭，可能导致一个 STOP_SENDING 帧(19.5 节)。
- 应用程序协议也可以请求流状态更改通知，包括对端打开或重置了流、对端中止读取流、新数据可用和由于流量控制当数据可以或不能写入到流。

## 3. 流状态

本节描述流的发送或接收组件。描述了两个状态机:一个用于终端传输数据的流(3.1 节)，另一个用于终端接收数据的流(3.2 节)。

单向流使用发送或接收状态机，这取决于流类型和端点角色。双向流在两个端点都使用两个状态机。在大多数情况下，无论流是单向的还是双向的，这些状态机的使用都是相同的。对于双向流来说，打开流的条件要稍微复杂一些，因为发送端或接收端打开都会导致流在两个方向打开。

本节中显示的状态机提供了大量的信息。本文档使用流状态来描述不同类型帧的发送时间和方式，以及接收不同类型帧时的预期反应。尽管这些状态机在实现 QUIC 时很有用，但这些状态并不是用来约束实现的。实现可以定义不同的状态机，只要它的行为与实现这些状态的实现一致。
