# WebRTC-HTTP ingestion protocol (WHIP)

> 原文 [https://www.ietf.org/archive/id/draft-ietf-wish-whip-04.html](https://www.ietf.org/archive/id/draft-ietf-wish-whip-04.html)

## 摘要

本文档描述了一个简单的基于 HTTP 的协议，该协议允许基于 WebRTC 的内容输入到流媒体服务或 CDN 中。

## 1. 介绍

虽然 WebRTC 在广泛的场景中非常成功，但它在广播/流媒体行业的应用还比较落后。

IETF RTCWEB 工作组标准化了 JSEP([RFC8829])，这是一种用于控制多媒体会话的设置、管理和关闭的机制。它还描述了如何使用 Offer/Answer 模型与会话描述协议(SDP)[RFC3264] 协商媒体流，以及通过网络发送的数据的格式(例如，媒体类型、编解码器参数和加密信息)。WebRTC 有意不指定应用程序级别的信令传输协议。这种灵活性允许实现范围更广泛的服务。然而，这些服务通常是独立的孤岛，不需要与其他服务互操作，也不需要可以与它们之间进行通信的工具存在。

在广播/流媒体领域，硬件编码器的使用非常简单，可以插入传输原始媒体的电缆，就地编码，并将其推送到任何流媒体服务或 CDN 入口。为每个 WebRTC 服务采用自定义信令传输协议阻碍了作为接入协议的广泛采用。

虽然有些标准信令协议可以与 WebRTC 集成，如 SIP[RFC3261] 或 XMPP[RFC6120]，但它们并不是为广播/流媒体服务而设计的，也没有在该行业采用的迹象。基于 RTP 的 RTSP[RFC7826] 可能是功能上最接近 WebRTC 的，但与 SDP[RFC3264] Offer/Answer 机制不兼容。



### 1.1.
