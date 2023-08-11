# TCP Congestion Control

> 原文 [https://datatracker.ietf.org/doc/html/rfc5681](https://datatracker.ietf.org/doc/html/rfc5681)

## 摘要

本文档定义了 TCP 的四种相互交织的拥塞控制算法：慢启动、避免拥塞、快速重传和快速恢复。此外，该文档规定了TCP在相对较长的空闲期后应如何开始传输，并讨论了各种确认生成方法。此文档废弃了RFC 2581。

## 1. Introduction

本文档规定了四种 TCP[RFC793] 拥塞控制算法：慢启动、避免拥塞、快速重传和快速恢复。这些算法是在 [Jac88] 和 [Jac90] 中设计的。它们与 TCP 的使用在 [RFC1122] 中进行了标准化。 [CJ89] 中给出了在加性增加、乘性减少拥塞控制方面的额外早期工作。

请注意，[Ste94] 提供了这些算法的示例，而 [WS95] 提供了对这些算法的 BSD 实现的源代码的解释。

除了指定这些拥塞控制算法外，本文档还指定了 TCP 连接在相对较长的空闲期后应该做什么，并指定和澄清了与 TCP ACK 生成有关的一些问题。

本文件废止了 [RFC2581]，而后者又废止了 [RFC2001]。

本文件的组织结构如下。第 2 节提供了将在整个文件中使用的各种定义。第 3 节提供了拥塞控制算法的规范。第 4 节概述了与拥塞控制算法相关的问题，最后，第 5 节概述了安全考虑因素。

## 2. Definitions

本节提供了几个术语的定义，这些术语将在本文件的其余部分中使用。

- **SEGMENT**：一个 SEGMENT 是任何 TCP/IP 数据或确认数据包（或两者兼有）。
- **SENDER MAXIMUM SEGMENT SIZE (SMSS)**：SMSS 是发件人可以发送的最大分段的大小。该值可以基于网络的最大传输单元、路径 MTU 发现 [RFC1191，RFC4821] 算法、RMSS（见下一项）或其他因素。大小不包括 TCP/IP 标头和选项。
- **RECEIVER MAXIMUM SEGMENT SIZE (RMSS)**：RMSS 是接收器愿意接受的最大分段的大小。这是接收器在连接启动期间发送的 MSS 选项中指定的值。或者，如果未使用 MSS 选项，则为 536 字节 [RFC1122]。大小不包括 TCP/IP 标头和选项。
- **FULL-SIZED SEGMENT**：包含允许的最大数据字节数的段（即，包含 SMSS 数据字节的段）。
- **RECEIVER WINDOW (rwnd)**：最近发布的接收器窗口。
- **CONGESTION WINDOW (cwnd)**：一个 TCP 状态变量，用于限制 TCP 可以发送的数据量。在任何给定时间，TCP 都不得发送序列号高于最高确认序列号与 cwnd 和 rwnd 最小值之和的数据。
- **INITIAL WINDOW (IW)**：初始窗口是三方握手完成后发送方拥塞窗口的大小。
- **LOSS WINDOW (LW)**：丢失窗口是 TCP 发送器使用其重传计时器检测到丢失后拥塞窗口的大小。
- **RESTART WINDOW (RW)**：重新启动窗口是TCP在空闲期后重新启动传输后的拥塞窗口的大小（如果使用慢启动算法；请参阅第4.1节了解更多讨论）。
- **FLIGHT SIZE**：已发送但尚未累计确认的数据量。
- **DUPLICATE ACKNOWLEDGMENT**：
  - (a) ACK 的接收器有未完成的数据
  - (b) 传入的确认没有携带数据
  - (c) SYN和FIN位都关闭时，确认被视为“重复”
  - (d) 确认号码等于在给定连接上接收到的最大确认（来自 [RFC793] 的 TCP.UNA），
  - (e) 传入确认中的通告窗口等于最后传入确认的通告窗口。或者，利用选择性确认（SACK）[RFC2018，RFC2883] 的 TCP 可以利用 SACK 信息来确定传入ACK何时是 “重复的”（例如，ACK 是否包含以前未知的 SACK 信息）。

## 3. Congestion Control Algorithms

本节定义了 [Jac88] 和 [Jac90] 中开发的四种拥塞控制算法：慢启动、拥塞避免、快速重传和快速恢复。在某些情况下，TCP 发送器比算法允许的更保守可能是有益的；但是，TCP 不得比以下算法允许的更激进（即，当以下算法计算的 cwnd 值不允许发送数据时，不得发送数据）。

此外，请注意，本文档中指定的算法将 丢包 用作拥塞信号。显式拥塞通知（ECN）也可以按照 [RFC3168] 中的规定使用。

### 3.1. Slow Start and Congestion Avoidance

TCP 发送器必须使用慢速启动和拥塞避免算法来控制注入网络的未处理数据量。为了实现这些算法，在 TCP 的每个连接状态中添加两个变量。拥塞窗口（cwnd）是发送方在接收到确认（ACK）之前可以向网络发送的数据量的限制，接收方的通告窗口（rwnd）是接收方对未完成数据量的极限。发送时使用 cwnd 和 rwnd 的最小值控制数据传输。

另一个状态变量，慢启动阈值（ssthresh），用于确定是否使用慢启动或拥塞避免算法来控制数据传输，如下所述。

在未知条件下开始传输到网络需要 TCP 缓慢地探测网络以确定可用容量，以避免不适当的大数据突发堵塞网络。慢启动算法在传输开始时或在修复由重传定时器检测到的丢失之后用于此目的。慢速启动还用于启动 TCP 发送器使用的 “ACK时钟”，以在慢速启动、拥塞避免和丢失恢复算法中将数据释放到网络中。

IW，cwnd的初始值，必须使用以下准则作为上限进行设置。

```
   If SMSS > 2190 bytes:
       IW = 2 * SMSS bytes and MUST NOT be more than 2 segments
   If (SMSS > 1095 bytes) and (SMSS <= 2190 bytes):
       IW = 3 * SMSS bytes and MUST NOT be more than 3 segments
   if SMSS <= 1095 bytes:
       IW = 4 * SMSS bytes and MUST NOT be more than 4 segments
```

### 3.2. Fast Retransmit/Fast Recovery



## 4. Additional Considerations


### 4.1. Restarting Idle Connections


### 4.2. Generating Acknowledgments


### 4.3. Loss Recovery Mechanisms

## 5. Security Considerations


## 6. Changes between RFC 2001 and RFC 2581


## 7. Changes Relative to RFC 2581
## 8. Acknowledgments

