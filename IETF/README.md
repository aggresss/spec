# RFC

## Reference

- [RFC Index](https://datatracker.ietf.org/doc/html-index.html)
- [RFC Search](https://www.rfc-editor.org/search/rfc_search.php)
- [RFC Mirrors](http://mirrors.nju.edu.cn/rfc/)
- [IETF Datatracker](https://datatracker.ietf.org/)
- [中国科学院计算技术研究所 · RFC文档中心](http://www.rfc.ac.cn/)


## Network Layer Protocls

RFC | 名称
---|---
[RFC791](https://datatracker.ietf.org/doc/html/rfc791) | INTERNET PROTOCOL

## Tansport Layer Protocols

[Protocol Registry](https://www.iana.org/assignments/protocol-numbers/protocol-numbers.xhtml)

RFC | 名称
---|---
[RFC768](https://datatracker.ietf.org/doc/html/rfc768) | User Datagram Protocol
[RFC9293](https://datatracker.ietf.org/doc/html/rfc9293) | Transmission Control Protocol (TCP)
[RFC9260](https://datatracker.ietf.org/doc/html/rfc9260) | Stream Control Transmission Protocol

### TCP

RFC | 名称
---|---
[RFC5681](https://datatracker.ietf.org/doc/html/rfc5681) | TCP Congestion Control // RENO
[RFC5682](https://datatracker.ietf.org/doc/html/rfc5682) | Forward RTO-Recovery (F-RTO)
[RFC6582](https://datatracker.ietf.org/doc/html/rfc6582) | The NewReno Modification to TCP's Fast Recovery Algorithm
[RFC6937](https://datatracker.ietf.org/doc/html/rfc6937) | Proportional Rate Reduction for TCP
[RFC8985](https://datatracker.ietf.org/doc/html/rfc8985) | The RACK-TLP Loss Detection Algorithm for TCP
[rfc9438](https://datatracker.ietf.org/doc/html/rfc9438) | CUBIC for Fast and Long-Distance Networks

### UDP

RFC | 名称
---|---
[RFC8085](https://datatracker.ietf.org/doc/html/rfc8085) | UDP Usage Guidelines

### Congestion Control

RFC | 名称
---|---
[RFC8312](https://datatracker.ietf.org/doc/html/rfc8312) | CUBIC for Fast Long-Distance Networks
[draft-cardwell-iccrg-bbr-congestion-control-02](https://datatracker.ietf.org/doc/html/draft-cardwell-iccrg-bbr-congestion-control-02) | BBR Congestion Control

## Upper Protocols

### Session

RFC | 名称
---|---
[RFC3264](https://datatracker.ietf.org/doc/html/rfc3264) | An Offer/Answer Model with the Session Description Protocol (SDP)
[RFC4566](https://datatracker.ietf.org/doc/html/rfc4566) | SDP: Session Description Protocol
[RFC2326](https://datatracker.ietf.org/doc/html/rfc2326) | Real Time Streaming Protocol (RTSP)
[RFC8829](https://datatracker.ietf.org/doc/html/rfc8829) | JavaScript Session Establishment Protocol (JSEP)

### SIP

RFC | 名称
---|---
[RFC3261](https://datatracker.ietf.org/doc/html/rfc3261) | SIP: Session Initiation Protocol
[RFC2976](https://datatracker.ietf.org/doc/html/rfc2976) | The SIP INFO Method
[RFC3428](https://datatracker.ietf.org/doc/html/rfc3428) | Session Initiation Protocol (SIP) Extension for Instant Messaging


### RTP/RTCP

RFC | 名称
---|---
[RFC3550](https://datatracker.ietf.org/doc/html/rfc3550) | RTP: A Transport Protocol for Real-Time Applications
[RFC4585](https://datatracker.ietf.org/doc/html/rfc4585) | Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF)
[RFC4588](https://datatracker.ietf.org/doc/html/rfc4588) | RTP Retransmission Payload Format
[RFC3611](https://datatracker.ietf.org/doc/html/rfc3611) | RTP Control Protocol Extended Reports (RTCP XR)
[RFC3711](https://datatracker.ietf.org/doc/html/rfc3711) | The Secure Real-time Transport Protocol (SRTP)
[RFC8285](https://datatracker.ietf.org/doc/html/rfc8285) | A General Mechanism for RTP Header Extensions

### RTP Payload

Codec Name | RFC
---|---
PCMU | https://datatracker.ietf.org/doc/html/rfc7655
PCMA | https://datatracker.ietf.org/doc/html/rfc7655
G722 | https://datatracker.ietf.org/doc/html/rfc5577
AAC ADTS | https://datatracker.ietf.org/doc/html/rfc3640
AAC LATM | https://datatracker.ietf.org/doc/html/rfc6416
OPUS | https://datatracker.ietf.org/doc/html/rfc7587
VP8 | https://datatracker.ietf.org/doc/html/rfc7741
H264 | https://datatracker.ietf.org/doc/html/rfc6184
VP9 | https://datatracker.ietf.org/doc/html/draft-ietf-payload-vp9-16
HEVC | https://datatracker.ietf.org/doc/html/rfc7798
AV1 | https://aomediacodec.github.io/av1-rtp-spec/

- [https://en.wikipedia.org/wiki/RTP_payload_formats](https://en.wikipedia.org/wiki/RTP_payload_formats)

### Transfer Encoding

RFC | 名称
-- | --
[RFC5109](https://datatracker.ietf.org/doc/html/rfc5109) | RTP Payload Format for Generic Forward Error Correction
[RFC5510](https://datatracker.ietf.org/doc/html/rfc5510) | Reed-Solomon Forward Error Correction (FEC) Schemes
[RFC8627](https://datatracker.ietf.org/doc/html/rfc8627) | RTP Payload Format for Flexible Forward Error Correction (FEC)

### WebRTC

RFC | 名称
---|---
[RFC5245](https://datatracker.ietf.org/doc/html/rfc5245) | Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal for Offer/Answer Protocols
[RFC5389](https://datatracker.ietf.org/doc/html/rfc5389) | Session Traversal Utilities for NAT (STUN)
[RFC5766](https://datatracker.ietf.org/doc/html/rfc5766) | Traversal Using Relays around NAT (TURN): Relay Extensions to Session Traversal Utilities for NAT (STUN)
[RFC4960](https://datatracker.ietf.org/doc/html/rfc4960) | Stream Control Transmission Protocol (SCTP)
[RFC8854](https://datatracker.ietf.org/doc/html/rfc8854) | WebRTC Forward Error Correction Requirements

### QUIC

RFC | 名称
---|---
[RFC8999](https://datatracker.ietf.org/doc/html/rfc8999) | Version-Independent Properties of QUIC
[RFC9000](https://datatracker.ietf.org/doc/html/rfc9000) | QUIC: A UDP-Based Multiplexed and Secure Transport
[RFC9001](https://datatracker.ietf.org/doc/html/rfc9001) | Using TLS to Secure QUIC
[RFC9002](https://datatracker.ietf.org/doc/html/rfc9002) | QUIC Loss Detection and Congestion Control
[RFC9221](https://datatracker.ietf.org/doc/html/rfc9221) | An Unreliable Datagram Extension to QUIC
[RFC9287](https://datatracker.ietf.org/doc/html/rfc9287) | Greasing the QUIC Bit
[RFC9368](https://datatracker.ietf.org/doc/html/rfc9368) | Compatible Version Negotiation for QUIC
[RFC9369](https://datatracker.ietf.org/doc/html/rfc9369) | QUIC Version 2
[RFC9308](https://datatracker.ietf.org/doc/html/rfc9308) | Applicability of the QUIC Transport Protocol
[RFC9312](https://datatracker.ietf.org/doc/html/rfc9312) | Manageability of the QUIC Transport Protocol

### HTTP

RFC | 名称
---|---
[RFC9110](https://datatracker.ietf.org/doc/html/rfc9110) | HTTP Semantics
[RFC9111](https://datatracker.ietf.org/doc/html/rfc9111) | HTTP Caching
[RFC9112](https://datatracker.ietf.org/doc/html/rfc9112) | HTTP/1.1
[RFC9113](https://datatracker.ietf.org/doc/html/rfc9113) | HTTP/2
[RFC9114](https://datatracker.ietf.org/doc/html/rfc9114) | HTTP/3

### TLS

RFC | 名称
---|---
[RFC2246](https://datatracker.ietf.org/doc/html/rfc2246) | The TLS Protocol Version 1.0
[RFC4346](https://datatracker.ietf.org/doc/html/rfc4346) | The Transport Layer Security (TLS) Protocol Version 1.1
[RFC5246](https://datatracker.ietf.org/doc/html/rfc5246) | The Transport Layer Security (TLS) Protocol Version 1.2
[RFC8446](https://datatracker.ietf.org/doc/html/rfc8446) | The Transport Layer Security (TLS) Protocol Version 1.3

### Web

[RFC6455](https://datatracker.ietf.org/doc/html/rfc6455) | The WebSocket Protocol

