# WebRTC-HTTP Egress Protocol (WHEP)

> 原文 https://www.ietf.org/archive/id/draft-murillo-whep-00.html](https://www.ietf.org/archive/id/draft-murillo-whep-00.html)

## 摘要

本文档描述了一个简单的基于 HTTP 的协议，该协议允许基于 WebRTC 的观看者观看来自流媒体服务和/或内容分发网络(CDN)或 WebRTC 传输网络(WTN)的内容。

## 1. 介绍

IETF RTCWEB 工作组标准化了 JSEP ([RFC8829])，这是一种用于控制多媒体会话的设置、管理和关闭的机制。它还描述了如何使用 Offer/Answer 模型与会话描述协议(SDP) [RFC3264] 协商媒体流，以及通过网络发送的数据的格式(例如，媒体类型、编解码器参数和加密)。WebRTC 有意不指定应用程序级别的信令传输协议。这种灵活性允许实现范围广泛的服务。然而，这些服务通常是孤岛，不需要与其他服务互操作性，也不需要利用可以与它们通信的工具的存在。

虽然有些标准信令协议可以与 WebRTC 集成，如 SIP [RFC3261] 或 XMPP [RFC6120]，但它们并不是为广播/流媒体服务而设计的，也没有在该行业采用的迹象。基于 RTP 的 RTSP [RFC7826] 可能是功能上最接近WebRTC 的，但与 SDP Offer/Answer 模型 [RFC3264] 不兼容。

因此，目前还没有为使用 WebRTC 从流媒体服务中消费媒体而设计的标准协议。

在许多情况下，缺乏使用 WebRTC 从流媒体服务中消费媒体的标准协议已经成为一个问题:

- WebRTC 服务和产品之间的互操作性。
- 重用易于集成的播放器软件。
- 与 HTTP 上的 Dynamic Adaptive Streaming over HTTP (DASH) 集成，通过 WebRTC 提供实时流媒体，同时通过 DASH 提供时移版本。
- 在不支持自定义 javascript 的设备(如电视)上播放 WebRTC 流。

本文档模仿了 WebRTC HTTP Ingest Protocol (WHIP) [I-D.draft-ietf-wish-whip]。并指定了一个简单的基于 HTTP 的协议，可用于使用 WebRTC 从流媒体服务中消费媒体。

## 2. 术语

本文档中的关键词“必须”、“不得”、“必需”、“应”、“不应”、“建议”、“不建议”、“可”和“可选”在所有大写字母出现时（如图所示）应按照 BCP 14[RFC2119] [RFC8174] 所述进行解释。

- WHEP 播放器：WebRTC 媒体播放器，通过接收和解码来自远程媒体服务器的媒体，充当 WHEP 协议的客户端。
- WHEP 端点：接收初始 WHEP 请求的出口服务器。
- WHEP 端点 URL：创建 WHEP 资源的 WHEP 端点的 URL。
- 媒体服务器：WebRTC 媒体服务器或消费者，与 WHEP 播放器建立媒体会话并将媒体传递给它。
- WHEP 资源：由 WHEP 端点为正在进行的出口会话分配的资源，WHEP 播放器可以发送更改会话的请求(例如 ICE 操作或终止)。
- WHEP 资源 URL：WHEP 端点分配给特定媒体会话的 URL，可用于执行终止会话或重新启动 ICE 等操作。

