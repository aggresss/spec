# WebRTC-HTTP ingestion protocol (WHIP)

> 原文 [https://www.ietf.org/archive/id/draft-ietf-wish-whip-04.html](https://www.ietf.org/archive/id/draft-ietf-wish-whip-04.html)

## 摘要

本文档描述了一个简单的基于 HTTP 的协议，该协议允许基于 WebRTC 的内容输入到流媒体服务或 CDN 中。

## 1. 介绍

虽然 WebRTC 在广泛的场景中非常成功，但它在广播/流媒体行业的应用还比较落后。

IETF RTCWEB 工作组标准化了 JSEP([RFC8829])，这是一种用于控制多媒体会话的设置、管理和关闭的机制。它还描述了如何使用 Offer/Answer 模型与会话描述协议(SDP)[RFC3264] 协商媒体流，以及通过网络发送的数据的格式(例如，媒体类型、编解码器参数和加密信息)。WebRTC 有意不指定应用程序级别的信令传输协议。这种灵活性允许实现范围更广泛的服务。然而，这些服务通常是独立的孤岛，不需要与其他服务互操作，也不需要可以与它们之间进行通信的工具存在。

在广播/流媒体领域，硬件编码器的使用非常简单，可以插入传输原始媒体的电缆，就地编码，并将其推送到任何流媒体服务或 CDN 入口。为每个 WebRTC 服务采用自定义信令传输协议阻碍了作为接入协议的广泛采用。

虽然有些标准信令协议可以与 WebRTC 集成，如 SIP[RFC3261] 或 XMPP[RFC6120]，但它们并不是为广播/流媒体服务而设计的，也没有在该行业采用的迹象。基于 RTP 的 RTSP[RFC7826] 可能是功能上最接近 WebRTC 的，但与 SDP[RFC3264] Offer/Answer 机制不兼容。

因此，目前还没有为使用 WebRTC 将媒体导入流媒体服务而设计的标准协议，因此内容提供商仍然严重依赖 RTMP 等协议来实现这一点。这些协议大多不是基于 RTP 的，在通过 WebRTC 进行出口时需要媒体协议转换。避免这种媒体协议转换是可取的，因为在这些协议和 WebRTC 之间没有功能上的对等，这增加了媒体服务器端的实现复杂性。

此外，这些协议中使用的媒体编解码器往往是有限的，没有经过协商，并不总是与WebRTC支持的媒体编码匹配。因为需要在接收节点上进行转码，所以会带来延迟、降低媒体质量并增加服务器端所需的处理工作负载。在自适应比特率流(ABR)实现中，服务器端转码传统上用于呈现多个版本，现在可以用 WebRTC 客户端很好地支持的 Simulcast[RFC8853] 和 SVC 编解码器替代。此外，WebRTC客户端还可以根据 RTCP 的反馈调整客户端编码参数，使编码质量最大化。

本文提出了一个支持 WebRTC 作为媒体接入方法的简单协议:

- 容易实现；
- 像流行的基于 IP 的广播协议一样容易使用；
- 完全符合 WebRTC 和 RTCWEB 规范；
- 允许在传统媒体平台和 WebRTC 端到端平台以尽可能低的延迟接入；
- 降低对硬件编码器和广播服务的要求，以支持 WebRTC；
- 可在 web 浏览器和本机编码器中使用；

## 2. 术语

本文档中的关键词“必须”、“不得”、“必需”、“应”、“不应”、“建议”、“不建议”、“可”和“可选”在所有大写字母出现时（如图所示）应按照 BCP 14[RFC2119] [RFC8174] 所述进行解释。

- WHIP 客户端：WebRTC 媒体编码器或生产者，作为 WHIP 协议的客户端，通过对媒体进行编码并将其发送到远程媒体服务器。
- WHIP 端点：接收初始 WHIP 请求的接收服务器。
- WHIP 端点 URL：创建 WHIP 资源的 WHIP 端点的 URL。
- 媒体服务器：WebRTC 媒体服务器或消费者，与 WHIP 客户端建立媒体会话并接收它产生的媒体。
- WHIP 资源：由 WHIP 端点为正在进行的接入会话分配的资源，WHIP 客户端可以发送更改会话的请求(例如 ICE 操作或终止)。
- WHIP 资源 URL：WHIP 端点分配给特定媒体会话的 URL，可用于终止会话或重新启动 ICE 等操作。

## 3. 概述

WebRTC-HTTP 接入协议(WHIP) 使用 HTTP POST 请求执行单次 SDP Offer/Answer，以便在编码器/媒体生产者(WHIP 客户端)和广播接收端点(媒体服务器)之间建立 ICE/DTLS 会话。

一旦 ICE/DTLS 会话建立，媒体将从编码器/媒体生成器(WHIP客户端)单向流向广播接收端点(媒体服务器)。为了降低复杂性，不支持 SDP 重新协商，因此在完成通过 HTTP 的初始 SDP Offer/Answer 后，不能添加或删除任何 track 或 stream 。

```text

 +-------------+    +---------------+ +--------------+ +---------------+
 | WHIP client |    | WHIP endpoint | | Media Server | | WHIP Resource |
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

图1: WHIP 会话建立和注销

## 4. 协议操作

为了建立一个接入会话，WHIP 客户机将根据 JSEP 规则生成一个 SDP Offer，并对配置的 WHIP 端点 URL 执行一个 HTTP POST 请求。

HTTP POST 请求的内容类型为 "application/sdp"，并包含作为主体的 SDP Offer。WHIP 端点将生成一个 SDP Answer 并返回一个 "201 Created" 响应，内容类型为 "application/ SDP"，SDP Answer 作为主体，Location 报头字段指向新创建的资源。

SDP Offer 应该使用 "sendonly" 属性，SDP Answer 必须使用 "recvonly" 属性。

```text
POST /whip/endpoint HTTP/1.1
Host: whip.example.com
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

HTTP/1.1 201 Created
ETag: "38sdf4fdsf54:EsAw"
Content-Type: application/sdp
Content-Length: 1400
Location: https://whip.example.com/resource/id

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
```

图2：HTTP POST 进行 Offer/Answer 操作示例

一旦建立了会话，ICE consent freshness[RFC7675] 机制将用于检测会话两端的突然断开和 DTLS 中断。

要显式终止会话，WHIP 客户端必须对初始 HTTP POST 的 Location 报头字段中返回的资源 URL 执行一个 HTTP DELETE 请求。当收到 HTTP DELETE 请求时，WHIP 资源将被移除，Media Server 上的资源将被释放，ICE 和 DTLS 会话将被终止。

媒体服务器终止会话必须遵循[RFC7675] 5.2 节中立即撤销同意的程序。

WHIP 端点必须对资源 URL 上的任何 HTTP GET、HEAD 或 PUT 请求返回 HTTP 405 响应，以便为本协议规范的未来版本保留其使用。

WHIP 资源必须对资源 URL 上的任何 HTTP GET、HEAD、POST 或 PUT 请求返回 HTTP 405 响应，以便为本协议规范的未来版本保留其使用。

### 4.1 ICE 和 NAT 支持

WHIP 客户的 Initial Offer 可以在 ICE 的完整收集和 ICE 候选人的完整名单完成后发送，也可以只包含本地候选人(甚至是空的候选人名单)。

为了简化协议，一旦发送了 SDP Answer，就不支持从 Media Server ICE 候选对象交换继续收集的 trickle 候选对象。在响应客户请求之前，WHIP 端点应收集媒体服务器的所有 ICE 候选人，SDP Answer 应包含媒体服务器的 ICE 候选人的完整列表。媒体服务器可以使用 ICE lite，而 WHIP 客户端必须实现完整的 ICE。

WHIP客户端可以通过发送一个HTTP PATCH 请求到 WHIP 资源 URL，请求正文包含 MIME 类型为 "application/trickle-ice-sdpfrag" 的 SDP 片段，如 [RFC8840] 中指定的，来执行 Trickle ICE 或者 ICE Restart [RFC8863]。当用于 Trickle ICE 时，此 PATCH 消息体将包含新的 ICE 候选项；当用于 ICE Restart 时，它将包含一个新的 ICE ufrag/pwd 对。

对于 WHIP 资源，Trickle ICE 和 ICE Restart 支持是可选的。如果 WHIP 资源不支持 Trickle ICE 或 ICE Restart，它必须对任何 HTTP PATCH 请求返回 "405 Method Not Allowed" 响应。如果 WHIP 资源支持 Trickle ICE 或 ICE Restart，但不同时支持两者，它必须对不支持的 HTTP PATCH 请求返回 "501 Not Implemented" 。

由于 WHIP 客户端发送的 HTTP PATCH 请求可能会被 WHIP 资源乱序接收，因此 WHIP 资源必须生成一个唯一的强关联实体标签，根据 [RFC9110] 2.3 节标识 ICE 会话。标识初始 ICE 会话的实体标记的初始值必须在向 WHIP 端点的初始 POST 请求 201 响应的 ETag 报头字段中返回。它还必须在触发 ICE Restart 的任何 PATCH 请求的 200 OK 中返回。

根据 [RFC9110] 3.1 节的规定，WHIP 客户端在发送 PATCH 请求执行 Trickle ICE 时必须包含一个 "If-Match" 报头字段，其中包含最新的已知实体标签。当 WHIP 资源收到 PATCH 请求时，它必须根据 [RFC9110] 3.1 节的规定，将指定的实体标签值与资源的当前实体标签值进行比较，如果不匹配，则返回 "412 Precondition Failed" 响应。

当不需要匹配特定的 ICE 会话时，WHIP 客户端不应该使用实体标记验证，例如在发起 DELETE 请求以终止会话时。

一个 WHIP 资源接收到一个带有新的 ICE 候选项的 PATCH 请求，但不执行ICE Restart，必须返回一个 "204 No Content" 的无正文响应。如果 Media Server 不支持候选传输或无法解析连接地址，它必须接受带有 204 响应的 HTTP 请求，并静默丢弃候选。

```text
PATCH /resource/id HTTP/1.1
Host: whip.example.com
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

图3: Trickle ICE 请求

根据 [RFC9110] 3.1 节的规定，WHIP 客户端发送执行 ICE Restart 的 PATCH 请求必须包含一个 "If-Match" 报头字段，字段值为 "*" 。

如果 HTTP PATCH 请求导致 ICE Restart，WHIP 资源将返回一个 "200 OK"，其中包含 "application/trickle-ice-sdpfrag" 主体，其中包含新的 ICE 用户名片段和密码。响应可以有选择地包含 Media Server 的新 ICE 候选集，新的实体标记对应 ETag 响应报头字段中的新 ICE 会话。

如果 ICE 请求不能被 WHIP 资源满足，资源必须返回适当的 HTTP 错误码，并且绝对不能立即终止会话。WHIP 客户端可以通过发出一个 HTTP DELETE 请求来重新执行 ICE 重启或终止会话。在这两种情况下，如果 ICE 协议由于 ICE 重启失败而失效(根据 [RFC7675] 第5.1节)，会话必须被终止。

```text
PATCH /resource/id HTTP/1.1
Host: whip.example.com
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

图4: ICE Restart 请求

因为 WHIP 客户端需要知道与 ICE 会话相关联的实体标记，以便发送新的 ICE 候选对象，所以它必须在接收到对初始 POST 请求或带有新实体标记值的 PATCH 请求的HTTP响应之前缓冲所有收集的候选对象。一旦知道了实体标签的值，WHIP 客户端应该发送一个聚合的 HTTP PATCH 请求，包含它目前缓冲过的所有候选 ICE。

在网络不稳定的情况下，ICE Restart 的 HTTP PATCH 请求和响应可能会被乱序接收。为了缓解这种情况，当客户端执行 ICE Restart 时，它必须丢弃任何先前的 ICE username/pwd frag，并忽略从挂起的 HTTP PATCH 请求接收到的任何进一步的 HTTP PATCH 响应。客户端必须只应用在响应中收到的 ICE 信息来响应最后发送的请求。如果客户端和服务器上的 ICE 信息不匹配(因为一个无序的请求)，那么 STUN 请求将包含无效的 ICE 信息，并且将被服务器拒绝。当 WHIP 客户端检测到这种情况时，它应该向服务器发送一个新的ICE Restart 请求。

### 4.3. 负载均衡和重定向

WHIP 端点和 Media Servers 可能不在同一服务器上，因此可以对传入的请求进行负载均衡，发送到不同的 Media Servers。WHIP 客户端应通过 "307 Temporary Redirect" 支持 HTTP 重定向，如 [RFC9110] 第 6.4.7 节所述。WHIP 资源 URL 必须是最终 URL，发送给它的 PATCH 和 DELETE 请求不需要支持重定向。

在高负载的情况下，WHIP 端点可能返回一个 "503 Service Unavailable" 状态码，表明服务器目前由于临时过载或计划维护而无法处理请求，这种情况可能会在一段延迟后缓解。WHIP 端点可能会发送一个 "Retry-After" 报头字段，指示用户代理在发出后续请求之前应该等待的最短时间。

### 4.4. STUN/TURN 服务器配置

WHIP 端点可以返回 STUN/TURN 服务器配置 URL 和客户端可以使用的凭据，在 "201 Created" 响应中向 WHIP 端点 URL 发送 POST 请求。

每个 STUN/TURN 服务器将使用 "Link" 报头字段 [RFC8288] 返回，其中 "rel" 属性值为 "ice-server"。链接目标 URI 是 [RFC7064] 和 [RFC7065] 中定义的服务器 URL。凭据编码在 Link 目标属性中如下所示:

- username: 如果 Link 报头字段表示一个 TURN 服务器，并且凭据类型为 "password"，那么该属性指定与该 TURN 服务器一起使用的用户名。
- credential: 如果 "credential-type" 属性缺失或有一个 "password" 值，credential 属性表示一个长期的身份验证密码，如 [RFC8489] 第 10.2 节所述。
- credential-type: 如果 Link 报头字段表示一个 TURN 服务器，那么该属性指定在 TURN 服务器请求授权时应该如何使用凭据属性值。如果该属性不存在，则默认值为 "password"。

```text
 Link: <stun:stun.example.net>; rel="ice-server"
     Link: <turn:turn.example.net?transport=udp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turn:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
     Link: <turns:turn.example.net?transport=tcp>; rel="ice-server";
           username="user"; credential="myPassword"; credential-type="password"
```

图5: ICE 服务配置示例

注意: "ice-server" 的 "rel" 属性值和目标属性的命名遵循 W3C WebRTC 推荐使用的名称。[REC-webrtc-20210126] RTCConfiguration dictionary 4.2.1。"ice-server" 的 "rel" 属性值没有加上 "urn:ietf:params:whip:" 的前缀，因此它可以被其他可能使用这种机制来配置 STUN/TURN 服务器使用的规范重用。

有一些 WebRTC 实现不支持在本地 Offer 创建后更新 STUN/TURN 服务器配置，如 [RFC8829] 第 4.1.18 节所述。为了支持这些客户端，在 POST 请求发送之前，WHIP 端点还可以在发送到 WHIP 端点 URL 的 OPTIONS 请求的响应中包含 STUN/TURN 服务器配置。

生成 TURN 服务器凭据可能需要执行对外部提供者的请求，这既会增加 OPTION 请求处理的延迟，又会增加处理该请求所需的处理。为了防止这种情况，如果 OPTION 请求是 CORS 的预发送请求，也就是说，如果 OPTION 请求不包含具有 "POST" 值的访问控制请求方法，并且访问控制请求头部 HTTP 头不包含 "Link" 值，则 WHIP 端点不应返回 STUN/TURN 服务器配置。

也可以用广播服务或 WHIP 客户机上的外部 TURN 提供程序提供的长期凭证配置 STUN/TURN 服务器 URL，覆盖 WHIP 端点提供的值。

### 4.5 身份验证和授权

