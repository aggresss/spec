# An Unreliable Datagram Extension to QUIC

> 原文 [https://datatracker.ietf.org/doc/html/rfc9221](https://datatracker.ietf.org/doc/html/rfc9221)

## 摘要

本文档定义了对 QUIC 传输协议的扩展，以增加对通过 QUIC 连接发送和接收不可靠数据报的支持。

## 1. Introduction

QUIC 传输协议 [RFC9000] 为传输可靠的应用数据流提供了一种安全的多路复用连接。QUIC 使用各种帧类型在数据包中传输数据，每种帧类型都定义了它所包含的数据是否会被重新传输。使用STREAM 帧发送可靠的应用程序数据流。
一些应用程序，特别是那些需要传输实时数据的应用程序，更喜欢不可靠地传输数据。过去，这些应用程序直接建立在 UDP [RFC0768] 作为传输的基础上，并经常通过 DTLS [RFC6347] 添加安全性。扩展 QUIC 以支持传输不可靠的应用程序数据，为安全数据报提供了另一种选择，并增加了共享用于可靠流的加密和身份验证上下文的好处。
本文档定义了两种新的 DATAGRAM QUIC 帧类型，它们携带应用程序数据而不需要重传。

### 1.1. Specification of Requirements

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

## 2. Motivation

通过 QUIC 传输不可靠的数据提供了优于现有解决方案的优势：

- 想要同时使用可靠流和不可靠流到同一对等端的应用程序可以通过在可靠的 QUIC 流和不可靠性的 QUIC 数据报流之间共享单个握手和身份验证上下文而受益。与打开 TLS 连接和 DTLS 连接相比，这可以减少握手所需的延迟。
- QUIC 使用了比 DTLS 握手更细致的丢失恢复机制。这可以使 QUIC 数据的丢失恢复更快。
- QUIC 数据报受 QUIC 拥塞控制。为可靠和不可靠的数据提供单一的拥塞控制可以更有效和高效。

这些功能可用于优化音频/视频流应用程序、游戏应用程序和其他实时网络应用程序。

不可靠的 QUIC 数据报也可以用于在 QUIC 上实现 IP 数据包隧道，例如用于虚拟专用网络（VPN）。互联网层隧道协议通常需要可靠且经过身份验证的握手，然后是 IP 分组的不可靠安全传输。例如，这可能需要用于控制数据的 TLS 连接和用于隧道传输 IP 数据包的 DTLS。一个单一的 QUIC 连接可以支持这两个部分，除了使用可靠的流之外，还可以使用不可靠的数据报。

## 3. Transport Parameter

通过 QUIC 传输参数（name＝max_DATAGRAM_frame_size，value＝0x20）来通告对接收 DATAGRAM 帧类型的支持。max_datagram_frame_size 传输参数是以字节为单位表示端点愿意接收的 datagram 帧的最大大小（包括帧类型、长度和有效载荷）的整数值（表示为可变长度整数）。

此参数的默认值为 0，表示端点不支持 DATAGRAM 帧。大于 0 的值表示端点支持 DATAGRAM 帧类型并且愿意在该连接上接收这样的帧。

端点在握手期间（或者如果使用 0-RTT，则在上一次握手期间）接收到具有非零值的 max_datagram_frame_size 传输参数之前，不得发送DATAGRAM帧。端点不得发送大于其从对等端接收到的 max_datagram_frame_size 值的 DATAGRAM 帧。当未通过传输参数指示支持时，接收 DATAGRAM 帧的端点必须以 PROTOCOL_VIOLATION 类型的错误终止连接。类似地，接收到大于其在 max_datagram_frame_size 传输参数中发送的值的 DATAGRAM 帧的端点必须以 PROTOCOL_VIOLATION 类型的错误终止连接。

对于大多数 DATAGRAM 帧的使用，建议在 max_datagram_frame_size 传输参数中发送 65535 的值，以指示该端点将接受适合 QUIC 数据包内的任何 DATAGRAM 框架。

max_datagram_frame_size 传输参数是对 datagram 帧的支持的单向限制和指示。使用 DATAGRAM 帧的应用程序协议可以选择仅在单个方向上协商和使用它们。
当客户端使用 0-RTT 时，它们可以存储服务器的 max_datagram_frame_size 传输参数的值。这样做允许客户端在 0-RTT 分组中发送 DATAGRAM 帧。当服务器决定接受 0-RTT 数据时，它们必须发送一个 max_datagram_frame_size 传输参数，该参数大于或等于它们在向其发送 NewSessionTicket 消息的连接中发送给客户端的值。如果客户端将max_dataram_frame_size 传输参数的值与其 0-RTT 状态一起存储，则客户端必须验证服务器在握手中发送的 max_datagram_frame 传输参数的新值是否大于或等于存储的值；否则，客户端必须终止连接，并出现错误 PROTOCOL_VIOLATION。

使用数据报的应用程序协议必须定义它们对缺少 max_datagram_frame_size 传输参数的情况如何反应。如果数据报支持是应用程序不可或缺的，那么如果不存在 max_datagram_frame_size 传输参数，则应用程序协议可能会使握手失败。

## 4. Datagram Frame Types

DATAGRAM 帧用于以不可靠的方式传输应用程序数据。DATAGRAM 帧中的 "Type" 字段的形式为 0b0011000X（或值 0x30 和 0x31）。DATAGRAM 帧中 Type 字段的最低有效位是 LEN 位（0x01），表示是否存在 Length 字段：如果该位设置为 0，则 Length 字段不存在，DATAGRAM Data 字段延伸到数据包的末尾；如果该位设置为 1，则存在 "Length" 字段。

```text
DATAGRAM frames are structured as follows:

DATAGRAM Frame {
  Type (i) = 0x30..0x31,
  [Length (i)],
  Datagram Data (..),
}

               Figure 1: DATAGRAM Frame Format
```

DATAGRAM 帧包含以下字段：

- **Length**: 一个可变长度整数，以字节为单位指定数据报数据字段的长度。仅当 LEN 位设置为 1 时，此字段才出现。当 LEN 位设置为 0 时，数据报数据字段扩展到 QUIC 数据包的末尾。请注意，允许使用空（即零长度）数据报。
- **Datagram Data**: 要传递的数据报的字节数。

## 5. Behavior and Usage

当应用程序通过 QUIC 连接发送数据报时，QUIC 将生成一个新的 datagram 帧，并在第一个可用的数据包中发送。该帧应尽快发送（由拥塞控制等因素决定，如下所述），并可与其他帧合并。

当 QUIC 端点接收到有效的 DATAGRAM 帧时，它应该立即将数据传递给应用程序，只要它能够处理帧并将内容存储在内存中。

与 STREAM 帧一样，DATAGRAM 帧包含应用程序数据，必须使用 0-RTT 或 1-RTT 密钥进行保护。

注意，虽然 max_datagram_frame_size 传输参数对 datagram 帧的最大大小设置了限制，但是该限制可以通过 max_udp_payload_size 传输参量和端点之间的路径的最大传输单元（MTU）来进一步减小。DATAGRAM 帧不能被分割；因此，应用程序协议需要处理最大数据报大小受到其他因素限制的情况。

### 5.1. Multiplexing Datagrams

DATAGRAM 帧作为一个整体属于 QUIC 连接，并且不与 QUIC 层的任何流 ID 相关联。然而，预计应用程序将希望通过使用标识符来区分特定的 DATAGRAM 帧，例如用于数据报的逻辑流或区分不同种类的数据报。

定义用于复用不同类型的数据报或数据报流的标识符是在 QUIC 上运行的应用程序协议的责任。应用程序定义数据报数据字段的语义以及如何解析它。

如果应用程序需要支持多个数据报流的共存，一种推荐的模式是在数据报数据字段的开头使用可变长度整数。这是一种简单的方法，允许使用最小的空间对大量流进行编码。

QUIC 实现应该向应用程序提供一个 API，以便为 DATAGRAM 帧和 QUIC 流分配相对优先级。

### 5.2. Acknowledgement Handling

尽管 DATAGRAM 帧在检测到丢失时不会重传，但它们是 ack-eliciting（[RFC9002]）。接收器应该支持延迟 ACK 帧（在 max_ack_delay 指定的限制范围内），以响应接收到的仅包含 DATAGRAM 帧的数据包，因为如果这些数据包暂时未被确认，发送器不会采取任何行动。当条件指示分组可能丢失时，接收器将继续发送 ACK 帧，因为分组的有效载荷对接收器来说是未知的，并且当由 max_ack_delay 或其他协议组件指示时。

与任何 ack-eliciting 帧一样，当发送方怀疑仅包含 DATAGRAM 帧的数据包丢失时，它发送探测数据包以引发更快的确认，如 [RFC9002] 第 6.2.4 节所述。

如果发送器检测到包含特定 DATAGRAM 帧的数据包可能已经丢失，则实现可以通知应用程序它认为数据报已经丢失。

类似地，如果包含 DATAGRAM 帧的分组被确认，则该实现可以通知发送器应用数据报被成功发送和接收。由于重新排序，这可能包括一个 DATAGRAM 帧，该帧被认为是丢失的，但在稍后被接收并确认。需要注意的是，DATAGRAM 帧的确认仅指示接收器上的传输层处理处理了该帧，而不能保证接收器上的应用程序成功地处理了数据。因此，该信号不能取代指示成功处理的应用层信号。

### 5.3. Flow Control

DATAGRAM 帧不提供任何明确的流控制信令，并且不对任何每个流或连接范围的数据限制作出贡献。

与不为 DATAGRAM 帧提供流控制相关联的风险是接收器可能无法提交必要的资源来处理帧。例如，它可能无法将帧内容存储在内存中。然而，由于 DATAGRAM 帧本质上是不可靠的，如果接收机不能处理它们，则它们可能被接收机丢弃。

### 5.4. Congestion Control

DATAGRAM 帧采用了 QUIC 连接的拥塞控制器。因此，在拥塞控制器允许之前，连接可能无法发送应用程序生成的 DATAGRAM 帧 [RFC9002]。发送器必须延迟发送帧直到控制器允许，或者在不发送帧的情况下丢弃帧（此时它可能会通知应用程序）。使用分组定步的实现（[RFC9002] 的第 7.7 节）也可以延迟 DATAGRAM 帧的发送，以保持一致的分组定步。

实现可以可选地支持允许应用程序指定发送到期时间，超过该发送到期时间应当丢弃拥塞控制的 DATAGRAM 帧而不进行传输。

## 6. Security Considerations

DATAGRAM 帧与 QUIC 连接内传输的其余数据共享相同的安全属性，因此 [RFC9000] 的安全考虑也适用。使用 DATAGRAM 帧传输的所有应用程序数据，如 STREAM 帧，必须由 0-RTT 或 1-RTT 密钥保护。
允许在 0-RTT 中发送 DATAGRAM 帧的应用协议需要定义 0-RTT 的可接受使用的简档；参见 [RFC9001] 第 5.6 节。

DATAGRAM 帧的使用可能被路径上能够丢弃分组的对手检测到。由于 DATAGRAM 帧不使用传输级重传，因此使用 DATAGRAM 框架的连接可能与其他连接不同，因为它们对分组丢失的响应不同。

## 7. IANA Considerations

### 7.1. QUIC Transport Parameter

This document registers a new value in the "QUIC Transport Parameters" registry maintained at <https://www.iana.org/assignments/quic>.

- **Value**: 0x20
- **Parameter Name**: max_datagram_frame_size
- **Status**: permanent
- **Specification**: RFC 9221

### 7.2. QUIC Frame Types

This document registers two new values in the "QUIC Frame Types" registry maintained at <https://www.iana.org/assignments/quic>.

- **Value**: 0x30-0x31
- **Frame Name**: DATAGRAM
- **Status**: permanent
- **Specification**: RFC 9221

## 8. References

### 8.1. Normative References

### 8.2. Informative References

## Acknowledgments
