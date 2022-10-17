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

## 3. 概述

WebRTC-HTTP Egress Protocol (WHEP) 使用 HTTP POST 请求执行单次 SDP Offer/Answer，以便在 WHEP 播放器和流媒体服务端点(媒体服务器)之间建立 ICE/DTLS 会话。

一旦 ICE/DTLS 会话建立，媒体将从媒体服务器单向流向 WHEP 播放器。为了降低复杂性，不支持 SDP 重新协商，因此在完成通过 HTTP 的初始 SDP Offer/Answer 后，不能添加或删除任何 track 或 stream。

```txt

 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP Player |    | WHEP Endpoint | | Media Server | | WHEP Resource |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |
    |                         |              |                  |
    |HTTP POST (SDP Offer)    |              |                  |
    +------------------------>+              |                  |
    |201 Created (SDP answer) |              |                  |
    +<------------------------+              |                  |
    |          ICE REQUEST                   |                  |
    +--------------------------------------->+                  |
    |          ICE RESPONSE                  |                  |
    |<---------------------------------------+                  |
    |          DTLS SETUP                    |                  |
    |<======================================>|                  |
    |          RTP/RTCP FLOW                 |                  |
    +<-------------------------------------->+                  |
    | HTTP DELETE                                               |
    +---------------------------------------------------------->+
    | 200 OK                                                    |
    <-----------------------------------------------------------x

```

图 1. WHEP 会话建立和注销

或者，在有些情况下，WHEP Player 可能希望服务提供SDP Offer (例如，在只支持音频时避免设置音频和视频会话)，因此在这种情况下，初始的 HTTP POST 请求将不包含正文，而响应将包含来自服务的SDP Offer。WHEP 播放器必须在后续的 HTTP PATCH 请求中向 WHEP 资源提供 SDP Answer。

```txt

 +-------------+    +---------------+ +--------------+ +---------------+
 | WHEP Player |    | WHEP Endpoint | | Media Server | | WHEP Resource |
 +--+----------+    +---------+-----+ +------+-------+ +--------|------+
    |                         |              |                  |
    |                         |              |                  |
    |HTTP POST (empty)        |              |                  |
    +------------------------>+              |                  |
    |201 Created (SDP offer)  |              |                  |
    +<------------------------+              |                  |
    | HTTP PATCH (SDP answer)                |                  |
    +---------------------------------------------------------->+
    | 200 OK                                 |                  |
    <-----------------------------------------------------------x
    |          ICE REQUEST                   |                  |
    +--------------------------------------->+                  |
    |          ICE RESPONSE                  |                  |
    |<---------------------------------------+                  |
    |          DTLS SETUP                    |                  |
    |<======================================>|                  |
    |          RTP/RTCP FLOW                 |                  |
    +<-------------------------------------->+                  |
    | HTTP DELETE                                               |
    +---------------------------------------------------------->+
    | 200 OK                                                    |
    <-----------------------------------------------------------x

```

图 2. WHEP 会话建立和注销

## 4. 协议操作

### 4.1. 由 WHEP 播放器生成的 SDP Offer

为了建立一个流会话，WHEP 播放器将根据 JSEP 规则生成一个 SDP Offer，并对配置的 WHEP 端点 URL 执行一个 HTTP POST 请求。

HTTP POST 请求的内容类型为 "application/sdp"，并包含作为主体的 SDP Offer。WHEP 端点将生成一个 SDP Answer 并返回一个 "201 Created" 响应，内容类型为 "application/sdp"，SDP answer 作为主体，Location 报头字段指向新创建的资源。

SDP Offer 应该使用 "recvonly" 属性，SDP Answer 必须使用 "sendonly" 属性。

```txt
POST /whep/endpoint HTTP/1.1
Host: whep.example.com
Content-Type: application/sdp
Content-Length: 1326

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 201 Created
ETag: "38sdf4fdsf54:EsAw"
Content-Type: application/sdp
Content-Length: 1400
Location: https://whep.example.org/resource/id

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=msid-semantic: WMS *
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b

```

图 3. HTTP POST 和 PATCH 请求交换 SDP Offer/Answer 示例

### 4.2. 由 WHEP 端点生成的 SDP Offer

如果 WHEP 播放器倾向于由 WHEP 端点生成 SDP offer, WHEP 播放器将发送一个 POST 请求，但不包含 HTTP BODY 和一个 Accept HTTP Header "application/sdp" 到配置的 WHEP 端点 URL。

WHEP 端点将根据 JSEP 规则生成一个 SDP offer，并返回一个 "201 Created" 响应，内容类型为 "application/sdp"，SDP offer 作为主体，Location报头字段指向新创建的资源，以及一个 Expire 报头，指示 WHEP 播放器允许向 WHEP 资源发送 SDP Answer 的最长时间。

WHEP 播放器必须对 WHEP 端点提供的 SDP Offer 生成一个SDP Answer，并向 WHEP 资源的 Location 头中提供的 URL 发送一个 HTTP PATCH 请求。HTTP PATCH 请求的内容类型为 "application/sdp"，并包含作为主体的 SDP answer。如果 SDP Offer 不被 WHEP 播放器接受，它必须执行一个 HTTP DELETE 操作来终止到 WHEP 资源 URL 的会话。

在这种情况下，SDP Offer 应该使用 "sendonly" 属性，SDP Answer 必须使用 "recvonly" 属性。

```txt
POST /whep/endpoint HTTP/1.1
Host: whep.example.com
Content-Length: 0
Accept: application/sdp

HTTP/1.1 201 Created
Content-Type: application/sdp
Content-Length: 1400
Location: https://whep.example.com/resource/id
Expires: Wed, 27 Jul 2022 07:28:00 GMT

v=0
o=- 5228595038118931041 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=msid-semantic: WMS
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=sendonly
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
a=rtcp-mux
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:zjkk
a=ice-pwd:bP+XJMM09aR8AiX1jdukzR6Y
a=ice-options:trickle
a=fingerprint:sha-256 DA:7B:57:DC:28:CE:04:4F:31:79:85:C4:31:67:EB:27:58:29:ED:77:2A:0D:24:AE:ED:AD:30:BC:BD:F1:9C:02
a=setup:actpass
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=sendonly
a=msid:- d46fb922-d52a-4e9c-aa87-444eadc1521b
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

PATCH /resource/id HTTP/1.1
Host: whep.example.com
Content-Type: application/sdp
Content-Length: 1326

v=0
o=- 1657793490019 1 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0 1
a=extmap-allow-mixed
a=ice-lite
a=msid-semantic: WMS *
m=audio 9 UDP/TLS/RTP/SAVPF 111
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:0
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
m=video 9 UDP/TLS/RTP/SAVPF 96 97
c=IN IP4 0.0.0.0
a=rtcp:9 IN IP4 0.0.0.0
a=ice-ufrag:526be20a538ee422
a=ice-pwd:2e13dde17c1cb009202f627fab90cbec358d766d049c9697
a=fingerprint:sha-256 F7:EB:F3:3E:AC:D2:EA:A7:C1:EC:79:D9:B3:8A:35:DA:70:86:4F:46:D9:2D:CC:D0:BC:81:9F:67:EF:34:2E:BD
a=candidate:1 1 UDP 2130706431 198.51.100.1 39132 typ host
a=setup:passive
a=mid:1
a=bundle-only
a=extmap:4 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:10 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=extmap:11 urn:ietf:params:rtp-hdrext:sdes:repaired-rtp-stream-id
a=recvonly
a=rtcp-mux
a=rtcp-rsize
a=rtpmap:96 VP8/90000
a=rtcp-fb:96 ccm fir
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtpmap:97 rtx/90000
a=fmtp:97 apt=96

HTTP/1.1 204 No Content
ETag: "38sdf4fdsf54:EsAw"

```
图 4. HTTP POST 和 PATCH 请求交换 SDP Offer/Answer 示例

如果 WHEP 资源在 HTTP POST Expire header 响应中指定的时间之前没有收到 HTTP PATCH 请求，它应该删除该资源，并对之后收到的 WHEP 资源 URL 上的任何请求响应一个 "404 not Found" 响应。

### 4.3. 常规的流程

WHEP 资源可能需要进行实时发布，以便允许 WHEP 播放器开始查看流。在这种情况下，WHEP 资源将对 WHEP 客户端发出的 POST 请求返回一个 409 冲突响应，并带有一个 Retry-After 报头，指示发送新请求前的秒数。WHEP玩家可以周期性地尝试连接到 WHEP 资源，以指数回调周期，初始值为 409 冲突响应 Header 中的 Retry-After 值。

一旦建立了会话，ICE consent freshness [RFC7675] 将用于检测会话的任何一方的突然断开和 DTLS 销毁。

要显式终止会话，WHEP 播放器必须对初始 HTTP POST 的 Location 报头字段中返回的资源 URL 执行一个 HTTP DELETE 请求。在接收到 HTTP DELETE 请求时，将删除媒体服务器上的 WHEP 资源并释放资源，终止ICE 和 DTLS 会话。

媒体服务器终止会话必须遵循 [RFC7675] 5.2 节中立即撤销同意的操作流程。

WHEP 端点必须对资源 URL 上的任何 HTTP GET、HEAD 或 PUT 请求返回 HTTP 405 响应，以便为本协议规范的未来版本保留其使用。

WHEP 资源必须对资源 URL 上的任何 HTTP GET、HEAD、POST 或 PUT 请求返回 HTTP 405 响应，以便为本协议规范的未来版本保留其使用。

### 4.4 ICE 和 NAT 支持

WHEP Player 提供的 SDP 可以在完整的 ICE 集合和完整的 ICE 候选人列表完成后发送，也可以只包含本地候选人(甚至是空的候选人列表)。

为了简化协议，一旦发送了 SDP Answer，就不支持从 Media Server ICE 候选对象交换聚集的 trickle 候选对象。在响应客户端请求之前，WHEP 端点应收集媒体服务器的所有 ICE 候选者，SDP Answer 应包含媒体服务器的 ICE 候选者的完整列表。媒体服务器可以使用 ICE lite，而 WHEP 播放器必须实现完整的 ICE。

WHEP 播放器可以通过向 WHEP 资源 URL 发送 HTTP PATCH 请求来执行 trickle ICE 或 ICE restart [RFC8863]，请求的主体包含 MIME 类型为 "application/trickle-ice-sdprag" 的 SDP 片段，如 [RFC8840] 所指定的。当用于 trickle ICE 时，此 PATCH 消息体将包含新的 ICE 候选项；当用于重新启动 ICE 时，它将包含一个新的 ICE ufrag/pwd 对。

WHEP 播放器在 SDP Offer/Answer 完成之前不能发送任何 ICE 候选或重新启动 ICE。因此，如果 WHEP 播放器在 SDP Offer/Answer 中不作为 Offer 提供者，在收到包含SDP Answer 的 200 OK HTTP PATCH 请求响应之前，它绝对不能发送任何用于 trickle ICE 或重启 ICE 的 HTTP PATCH 请求。

对于 WHEP 资源，trickle ICE 和 ICE restart 支持是可选的。如果 WHEP 资源不支持 trickle ICE或 ICE restart，它必须对任何 HTTP PATCH 请求返回 405 Method not Allowed 响应。如果 WHEP 资源支持 trickle ICE 或 ICE restart，但不同时支持两者，它必须为不支持的 HTTP PATCH 请求返回 501 not Implemented。

由于 WHEP 播放器发送的 HTTP PATCH 请求可能会被 WHEP 资源乱序接收，因此 WHEP 资源必须根据 [RFC9110] 2.3 节生成一个唯一的强实体标签来标识 ICE 会话。标识初始 ICE 会话的实体标签的初始值必须在对 WHEP 端点的初始 POST 请求的 201 响应中的 ETag 报头字段中返回，如果 WHEP 播放器作为 SDP Offer 提供者，或者在包含 SDP Answer 的 HTTP PATCH 响应中返回。它还必须在触发 ICE 重新启动的任何 PATCH 请求的 200 OK 中返回。

WHEP 播放器发送执行 trickle ICE 的PATCH请求时，必须包含一个 "If-Match" 报头字段，根据 [RFC9110] 3.1 节的规定，包含最新的已知实体标签。当 WHEP 资源收到 PATCH 请求时，它必须根据 [RFC9110] 3.1 节将指定的实体标签值与资源的当前实体标签值进行比较，如果不匹配，则返回 "412 Precondition Failed" 响应。

WHEP player 不应该在不需要匹配特定的 ICE 会话时使用实体标签验证，例如在发起 DELETE 请求终止会话时。

WHEP 资源收到一个带有新的 ICE 候选项的 PATCH 请求，但不执行 ICE 重新启动，必须返回一个 "204 No Content" 的无正文响应。如果 Media Server 不支持候选传输或无法解析连接地址，它必须接受带有 204 响应的 HTTP 请求，并静默丢弃候选。

```txt
PATCH /resource/id HTTP/1.1
Host: whep.example.com
If-Match: "38sdf4fdsf54:EsAw"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 548

a=ice-ufrag:EsAw
a=ice-pwd:P2uYro0UCOQ4zxjKXaWCBui1
m=audio RTP/AVP 0
a=mid:0
a=candidate:1387637174 1 udp 2122260223 192.0.2.1 61764 typ host generation 0 ufrag EsAw network-id 1
a=candidate:3471623853 1 udp 2122194687 198.51.100.1 61765 typ host generation 0 ufrag EsAw network-id 2
a=candidate:473322822 1 tcp 1518280447 192.0.2.1 9 typ host tcptype active generation 0 ufrag EsAw network-id 1
a=candidate:2154773085 1 tcp 1518214911 198.51.100.2 9 typ host tcptype active generation 0 ufrag EsAw network-id 2
a=end-of-candidates

HTTP/1.1 204 No Content

```
图 5: Trickle ICE 请求

WHEP 播放器发送用于执行 ICE 重启的 PATCH 请求时，必须包含一个字段值为 "*" 的 "If-Match" 报头字段，按照 [RFC9110] 3.1 节的规定。

如果 HTTP PATCH 请求导致 ICE 重新启动，WHEP 资源将返回一个 "200 OK"，其中包含 "application/trickle-ice-sdprag" 主体，其中包含新的 ICE 用户名片段和密码。响应可以有选择地包含 Media Server的新 ICE 候选集，新的实体标记对应 ETag 响应报头字段中的新 ICE 会话。

如果 ICE 请求不能被 WHEP 资源满足，WHEP 资源必须返回一个适当的 HTTP 错误码，并且绝对不能立即终止会话。WHEP 播放器可以重试执行一个新的 ICE 重启或通过发出一个 HTTP DELETE 请求来终止会话。在这两种情况下，如果 ICE 协议由于 ICE 重启失败而失效(根据RFC7675第5.1节)，会话必须被终止。

```txt
PATCH /resource/id HTTP/1.1
Host: whep.example.com
If-Match: "*"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 54

a=ice-ufrag:ysXw
a=ice-pwd:vw5LmwG4y/e6dPP/zAP9Gp5k

HTTP/1.1 200 OK
ETag: "289b31b754eaa438:ysXw"
Content-Type: application/trickle-ice-sdpfrag
Content-Length: 102

a=ice-lite
a=ice-ufrag:289b31b754eaa438
a=ice-pwd:0b66f472495ef0ccac7bda653ab6be49ea13114472a5d10a

```

图 6：ICE 重启请求

因为 WHEP 播放器需要知道与 ICE 会话相关联的实体标记，以便发送新的 ICE 候选对象，所以它必须在接收到对初始 POST 请求或带有新实体标记值的 PATCH 请求的 HTTP 响应之前缓冲所有收集的候选对象。一旦它知道了实体标签的值，WHEP 播放器应该发送一个聚合的 HTTP PATCH 请求，包含它目前缓冲的所有 ICE 候选。

在网络不稳定的情况下，ICE 重启 HTTP PATCH 请求和响应可能会被乱序接收。为了缓解这种情况，当客户端执行 ICE 重新启动时，它必须丢弃任何先前的 ICE用户名/pwd片段，并忽略从挂起的 HTTP PATCH 请求接收到的任何进一步的 HTTP PATCH 响应。客户端必须只应用在响应中收到的 ICE 信息来响应最后发送的请求。如果客户端和服务器上的 ICE 信息不匹配(因为一个无序的请求)，那么 STUN 请求将包含无效的 ICE 信息，并且将被服务器拒绝。当 WHEP 播放器检测到这种情况时，它应该向服务器发送一个新的 ICE 重启请求。

### 4.5. WebRTC 约束

在特定情况下的媒体消费从流媒体服务,可以进行一些假设的服务器端简化了 WebRTC 合规负担，WebRTC-gateway 文档中详细说明 [I-D.draft-ietf-rtcweb-gateways]。

为了减少在播放器和媒体服务器中实现 WHEP 的复杂性，WHEP 对 WebRTC 的使用施加了以下限制:

WHEP 播放器和 WHEP 端点都应该使用 SDP bundle [RFC9143]。每个 "m=" Section 必须是单个 BUNDLE 组的一部分。因此，当 WHEP 播放器或 WHEP 端点发送 SDP Offer 时，它必须在每个绑定的 "m=" Section 包含 "bundle-only" 属性。WHEP 播放器和媒体服务器必须根据 [RFC9143] 第 9 节的规定支持与 BUNDLE 组关联的多路媒体。此外，根据 [RFC9143]，WHEP 播放器和媒体服务器将对所有捆绑的媒体使用RTP/RTCP 多路复用。WHEP 播放器和媒体服务器应该包括 "rtcp-mux-only" 属性在每个绑定的 "m=" section。

作为一个给定的流的编解码器可能不会被媒体服务器当 WHEP player 开始接收媒体流，如果作为 SDP Answer 提供者 WHEP 端点，它必须包括所有的编解码器提供支持的 SDP Answer 关于将不做任何假设的编解码器发送。

trickle ICE 和 ICE restart 的支持是可选的，如 WHEP player 和媒体服务器的 4.1 节中所解释的。

### 4.6. 负载平衡和重定向

WHEP 端点和媒体服务器可能不在同一服务器上，因此可以对传入的请求进行负载平衡，发送到不同的媒体服务器。WHEP 播放器应通过 "307 Temporary Redirect response code" 支持 HTTP 重定向，如 [RFC9110] 第 6.4.7 节所述。WHEP 资源 URL 必须是最终的 URL，对于发送给它的 PATCH 和 DELETE 请求，不需要支持重定向。

在高负载的情况下，WHEP 端点可能返回一个 503(Service Unavailable) 状态码，表明服务器目前由于临时过载或计划维护而无法处理请求，这种情况可能会在一段延迟后缓解。WHEP 端点可能会发送一个 Retry-After 报头字段，指示用户代理在发出后续请求之前应该等待的最小时间。

### 4.7. STUN/TURN 服务配置

WHEP 端点可以在 "201 Created" 响应中返回 STUN/TURN 服务器配置 URL 和客户端可用的凭证，以响应对 WHEP 端点 URL 的 HTTP POST 请求。

每个 STUN/TURN 服务器将使用 "Link" 报头字段 [RFC8288] 返回，其中 "rel" 属性值为 "ice-server"，如 [I-D.draft-ietf-wish-whip] 所指定。

也可以用广播服务或 WHEP 播放器上的外部 TURN 提供程序提供的长期凭证配置 STUN/TURN 服务器 URL，覆盖 WHEP 端点提供的值。

### 4.8. 身份验证和授权

WHEP 端点和资源可能要求使用 HTTP 授权报头字段和承载令牌对 HTTP 请求进行身份验证，如 [RFC6750] 2.1 节所述。WHEP 播放器必须实现这种身份验证和授权机制，并在发送到 WHEP 端点或资源的所有 HTTP 请求中发送 HTTP 授权报头字段，但 CORS 的 preflight OPTIONS 请求除外。

承载标记的性质、语法和语义，以及如何将其分发到客户机，超出了本文档的范围。可以使用的令牌类型的一些例子是，但不限于，根据 [RFC6750] 和 [RFC8725] 的JWT令牌，或存储在数据库中的共享秘密。

WHEP 端点和资源可以通过在 WHEP 端点或资源的 URL 中编码身份验证令牌来执行身份验证和授权。在 WHEP 播放器没有配置为使用承载令牌的情况下，HTTP 授权报头字段在任何请求中都不能被发送。

### 4.9. 协议拓展

为了支持为 WHEP 协议定义的未来扩展，定义了一个用于注册和宣布新扩展的通用过程。

WHEP 服务器支持的协议扩展必须在发送到 WHEP 端点的初始 HTTP POST 请求的 "201 Created" 响应中向 WHEP 播放器发布。WHEP 端点必须为每个扩展返回一个 "Link" 报头字段，带有扩展 "rel" 类型属性和用于接收与该扩展相关的请求的 HTTP 资源的 URI。

协议扩展对于 WHEP 播放器和 WHEP 端点和资源都是可选的。WHEP 玩家必须忽略任何具有未知 "rel" 属性值的链接属性，WHEP 端点和资源必须不要求使用任何扩展。

每个协议扩展必须在 IANA 注册一个唯一的 "rel" 属性值，前缀为: "urn:ietf:params:whep:ext"，如 6.2 节所述。

例如，考虑到使用 https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events 中指定的服务器发送事件的服务器到客户端通信的潜在扩展，连接到已发布流的服务器端事件资源的 URL 可以在初始 HTTP "201 Created" 响应中返回，该响应带有 "Link" 报头字段和 "urn:ietf:params:whep:ext:example:server-sent-events" 的 "rel" 属性。(本文档不指定该扩展名，仅作为示例。)

在这种理论上的情况下，HTTP POST 请求的 HTTP 201 响应应该如下所示:

```txt
HTTP/1.1 201 Created
Content-Type: application/sdp
Location: https://whep.example.org/resource/id
Link: <https://whep.ietf.org/publications/213786HF/sse>;
      rel="urn:ietf:params:whep:ext:example:server-side-events"
```
