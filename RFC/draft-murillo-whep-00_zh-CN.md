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


