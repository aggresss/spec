# CUBIC for Fast Long-Distance Networks

> 原文 [https://datatracker.ietf.org/doc/html/rfc8312](https://datatracker.ietf.org/doc/html/rfc8312)

## 摘要

CUBIC 是当前 TCP 标准的扩展。 它与当前的 TCP 标准的区别仅在于发送端的拥塞控制算法。 特别是，它使用三次函数代替当前 TCP 标准的线性窗口增加函数，以提高快速和长距离网络下的可扩展性和稳定性。 CUBIC 及其前身算法已被 Linux 默认采用并已使用多年。 本文档提供了 CUBIC 规范，以支持第三方实现并通过 CUBIC 性能实验来征求社区反馈。

## 1. Introduction

TCP 在快速长距离网络中的低利用率问题在 [K03] 和 [RFC3649] 中有详细记录。 此问题是由于在具有大带宽延迟积 (BDP) 的网络中发生拥塞事件后拥塞窗口缓慢增加而引起的。 [HKLRX06]表明，即使在数百个数据包的拥塞窗口大小范围内，也经常观察到此问题。 这个问题同样适用于所有 Reno 风格的 TCP 标准及其变体，包括 TCP-RENO [RFC5681]、TCP-NewReno [RFC6582] [RFC6675]、SCTP [RFC4960] 和 TFRC [RFC5348]，它们使用相同的线性 用于窗口增长的 increase 函数，我们下面将其统称为 “标准TCP”。

CUBIC 最初在 [HRX08] 中提出，是对标准 TCP 拥塞控制算法的修改，以解决这个问题。 本文档描述了 CUBIC 的最新规范。 具体来说，CUBIC使用三次函数代替标准TCP的线性窗口增加函数，以提高快速和长距离网络下的可扩展性和稳定性。

二进制递增拥塞控制 (BIC-TCP) [XHR04] 是 CUBIC 的前身，于 2005 年被 Linux 选为默认的 TCP 拥塞控制算法，并已被整个互联网社区使用了数年。 CUBIC 使用与 BIC-TCP 类似的窗口增加功能，其设计目的是在带宽使用方面比 BIC-TCP 更不激进且更公平，同时保持 BIC-TCP 的优势，如稳定性、窗口可扩展性和 RTT 公平性。 CUBIC已经取代BIC-TCP成为Linux中默认的TCP拥塞控制算法，并已被Linux在全球范围内部署。 通过在各种互联网场景中的广泛测试，我们相信CUBIC在全球互联网上的测试和部署是安全的。

在接下来的章节中，我们首先简要解释CUBIC的设计原理，然后提供CUBIC的准确规范，最后按照[RFC5033]中指定的指南讨论CUBIC的安全特性。

## 2. Conventions
## 3. Design Principles of CUBIC
## 4. CUBIC Congestion Control
### 4.1. Window Increase Function
### 4.2. TCP-Friendly Region
### 4.3. Concave Region
### 4.4. Convex Region
### 4.5. Multiplicative Decrease
### 4.6. Fast Convergence
### 4.7. Timeout
### 4.8. Slow Start
## 5. Discussion
### 5.1. Fairness to Standard TCP
### 5.2. Using Spare Capacity
### 5.3. Difficult Environments
### 5.4. Investigating a Range of Environments
### 5.5. Protection against Congestion Collapse
### 5.6. Fairness within the Alternative Congestion Control Algorithm
### 5.7. Performance with Misbehaving Nodes and Outside Attackers
### 5.8. Behavior for Application-Limited Flows
### 5.9. Responses to Sudden or Transient Events
### 5.10. Incremental Deployment
## 6. Security Considerations
## 7. IANA Considerations
## 8. References
### 8.1. Normative References
### 8.2. Informative References
