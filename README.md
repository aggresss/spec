# Specification For Media

## ISO/IEC

### ISO/IEC 13818

- Part 1: Systems
- Part 2: Video
- Part 3: Audio
- Part 4: Conformance testing
- Part 5: Software simulation [Technical Report]
- Part 6: Extensions for DSM-CC
- Part 7: Advanced Audio Coding (AAC)
- Part 9: Extension for real time interface for systems decoders
- Part 10: Conformance extensions for Digital Storage Media Command and Control (DSM-CC)
- Part 11: IPMP on MPEG-2 systems

### ISO/IEC 14496

- Part 1: Systems
- Part 2: Visual
- Part 3: Audio
- Part 4: Conformance testing
- Part 5: Reference software
- Part 6: Delivery Multimedia Integration Framework (DMIF)
- Part 7: Optimized reference software for coding of audio-visual objects
- Part 8: Carriage of ISO/IEC 14496 contents over IP networks
- Part 9: Reference hardware description
- Part 10: Advanced Video Coding (AVC)
- Part 11: Scene description and application engine
- Part 12: ISO base media file format
- Part 13: Intellectual Property Management and Protection (IPMP) extensions
- Part 14: MP4 file format
- Part 15: Advanced Video Coding (AVC) file format
- Part 16: Animation Framework eXtension (AFX)

## RFC

### SIP/SDP

RFC | 名称 | 备注
---|---|---
[2976](https://tools.ietf.org/html/rfc2976) | The SIP INFO Method | SIP INFO方法
[3261](https://tools.ietf.org/html/rfc3261) | SIP: Session Initiation Protocol | SIP 协议
[3265](https://tools.ietf.org/html/rfc3265) | Session Initiation Protocol (SIP)-Specific Event Notification | SIP 事件通知
[3428](https://tools.ietf.org/html/rfc3428) | Session Initiation Protocol (SIP) Extension for Instant Messaging | SIP 即时消息拓展
[3264](https://tools.ietf.org/html/rfc3264) | An Offer/Answer Model with the Session Description Protocol (SDP) | SDP 关于邀请/响应的格式定义
[4566](https://tools.ietf.org/html/rfc4566) | SDP: Session Description Protocol | SDP协议

### RTSP

RFC | 名称 | 备注
---|---|---
[2326](https://tools.ietf.org/html/rfc2326) | Real Time Streaming Protocol (RTSP) | RTSP

### Connection
RFC | 名称 | 备注
---|---|---
[5245](https://tools.ietf.org/html/rfc5245) | Interactive Connectivity Establishment (ICE): A Protocol for Network Address Translator (NAT) Traversal for Offer/Answer Protocols | ICE
[5389](https://tools.ietf.org/html/rfc5389) | Session Traversal Utilities for NAT (STUN) | STUN
[5766](https://tools.ietf.org/html/rfc5766) | Traversal Using Relays around NAT (TURN): Relay Extensions to Session Traversal Utilities for NAT (STUN) | TURN

### RTP/RTCP

RFC | 名称 | 备注
---|---|---
[1889](https://tools.ietf.org/html/rfc1889) | RTP: A Transport Protocol for Real-Time Applications | 早期RTP协议，RTP v1
[1890](https://tools.ietf.org/html/rfc1890) | RTP Profile for Audio and Video Conferences with Minimal Control | RTP的负载类型定义，对应于RTP v1
[2198](https://tools.ietf.org/html/rfc2198) | RTP Payload for Redundant Audio Data | 发送音频冗余数据的机制，FEC的雏形
[3550](https://tools.ietf.org/html/rfc3550) | RTP: A Transport Protocol for Real-Time Applications | 现用RTP协议，RTP v2
[3551](https://tools.ietf.org/html/rfc3551) | RTP Profile for Audio and Video Conferences with Minimal Control | RTP的负载类型定义，对应于RTP v2
[3611](https://tools.ietf.org/html/rfc3611) | RTP Control Protocol Extended Reports (RTCP XR) | RTCP的拓展报文即XR报文定义
[3640](https://tools.ietf.org/html/rfc3640) | RTP Payload Format for Transport of MPEG-4 Elementary Streams | RTP负载为MPEG-4的格式定义
[3711](https://tools.ietf.org/html/rfc3711) | The Secure Real-time Transport Protocol (SRTP) | RTP媒体流采用AES-128对称加密
[3984](https://tools.ietf.org/html/rfc3984) | RTP Payload Format for H.264 Video | RTP负载为H264的格式定义，已被6184取代
[4103](https://tools.ietf.org/html/rfc4103) | RTP Payload for Text Conversation | RTP负载为文本或者T.140的格式定义
[4571](https://tools.ietf.org/html/rfc4571) | Framing Real-time Transport Protocol (RTP) and RTP Control Protocol (RTCP) Packets over Connection-Oriented Transport | 面向连接的传输数据包帧 RTP 和 RTCP 协议
[4585](https://tools.ietf.org/html/rfc4585) | Extended RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/AVPF) | NACK定义，通过实时的RTCP进行丢包重传
[4587](https://tools.ietf.org/html/rfc4587) | RTP Payload Format for H.261 Video Streams | H261的负载定义
[4588](https://tools.ietf.org/html/rfc4588) | RTP Retransmission Payload Format | RTP重传包的定义
[4961](https://tools.ietf.org/html/rfc4961) | Symmetric RTP / RTP Control Protocol (RTCP) | 终端收发端口用同一个，叫做对称的RTP，便于DTLS加密
[5104](https://tools.ietf.org/html/rfc5104) | Codec Control Messages in the RTP Audio-Visual Profile with Feedback (AVPF) | 基于4585实时RTCP消息，来控制音视频编码器的机制
[5109](https://tools.ietf.org/html/rfc5109) | RTP Payload Format for Generic Forward Error Correction | Fec的通用规范。
[5124](https://tools.ietf.org/html/rfc5124) | Extended Secure RTP Profile for Real-time Transport Control Protocol (RTCP)-Based Feedback (RTP/SAVPF) | SRTP的丢包重传
[5285](https://tools.ietf.org/html/rfc5285) | A General Mechanism for RTP Header Extensions | RTP 扩展头定义，可以扩展1或2个字节，比如CSRC，已被8285协议替代
[5450](https://tools.ietf.org/html/rfc5450) | Transmission Time Offsets in RTP Streams | 计算RTP的时间差，可以配合抖动计算
[5484](https://tools.ietf.org/html/rfc5484) | Associating Time-Codes with RTP Streams | RTP和RTCP中时间格式的定义
[5506](https://tools.ietf.org/html/rfc5506) | Support for Reduced-Size Real-Time Transport Control Protocol (RTCP): Opportunities and Consequences | RTCP压缩
[5669](https://tools.ietf.org/html/rfc5669) | The SEED Cipher Algorithm and Its Use with the Secure Real-Time Transport Protocol (SRTP) | SRTP的对称加密算法的种子使用方法
[5691](https://tools.ietf.org/html/rfc5691) | RTP Payload Format for Elementary Streams with MPEG Surround Multi-Channel Audio | 对于MPEG-4中有多路音频的RTP负载格式的定义
[5760](https://tools.ietf.org/html/rfc5760) | RTP Control Protocol (RTCP) Extensions for Single-Source Multicast Sessions with Unicast Feedback | RTCP对于单一源进行多播的反馈机制
[5761](https://tools.ietf.org/html/rfc5761) | Multiplexing RTP Data and Control Packets on a Single Port | RTP和RTCP在同一端口上传输
[6051](https://tools.ietf.org/html/rfc6051) | Rapid Synchronisation of RTP Flows | 多RTP流的快速同步机制，适用于MCU的处理
[6128](https://tools.ietf.org/html/rfc6128) | RTP Control Protocol (RTCP) Port for Source-Specific Multicast (SSM) Sessions | RTCP对于多播中特定源的反馈机制
[6184](https://tools.ietf.org/html/rfc6184) | RTP Payload Format for H.264 Video | H264的负载定义
[6188](https://tools.ietf.org/html/rfc6188) | The Use of AES-192 and AES-256 in Secure RTP | SRTP拓展定义AES192和AES256
[6189](https://tools.ietf.org/html/rfc6189) | ZRTP: Media Path Key Agreement for Unicast Secure RTP | ZRTP的定义，非对称加密，用于密钥交换
[6190](https://tools.ietf.org/html/rfc6190) | RTP Payload Format for Scalable Video Coding | H264-SVC的负载定义
[6222](https://tools.ietf.org/html/rfc6222) | Guidelines for Choosing RTP Control Protocol (RTCP) Canonical Names (CNAMEs) | RTCP的CNAME的选定规则，可根据RFC 4122的UUID来选取
[6798](https://tools.ietf.org/html/rfc6798) | RTP Control Protocol (RTCP) Extended Report (XR) Block for Packet Delay Variation Metric Reporting | RTCP的XR报文，关于数据包延迟变化度量报告的定义
[6843](https://tools.ietf.org/html/rfc6843) | RTP Control Protocol (RTCP) Extended Report (XR) Block for Delay Metric Reporting | RTCP的XR报文，关于延迟指标报告的定义
[6958](https://tools.ietf.org/html/rfc6958) | RTP Control Protocol (RTCP) Extended Report (XR) Block for Burst/Gap Loss Metric Reporting | RTCP的XR报文，关于突发/间隙损失度量报告的定义
[7002](https://tools.ietf.org/html/rfc7002) | RTP Control Protocol (RTCP) Extended Report (XR) Block for Discard Count Metric Reporting | RTCP的XR报文，关于丢弃计数度量的定义
[7003](https://tools.ietf.org/html/rfc7003) | RTP Control Protocol (RTCP) Extended Report (XR) Block for Burst/Gap Discard Metric Reporting | RTCP的XR报文，关于破裂/丢弃指标差距的定义
[7097](https://tools.ietf.org/html/rfc7097) | RTP Control Protocol (RTCP) Extended Report (XR) for RLE of Discarded Packets | RTCP的XR报文，关于RLE丢弃的的定义
[6904](https://tools.ietf.org/html/rfc6904) | Encryption of Header Extensions in the Secure Real-time Transport Protocol | SRTP的RTP头信息加密
[7022](https://tools.ietf.org/html/rfc7022) | Guidelines for Choosing RTP Control Protocol (RTCP) Canonical Names (CNAMEs) | RTCP的CNAME的选定规则，修订6222
[7160](https://tools.ietf.org/html/rfc7160) | Support for Multiple Clock Rates in an RTP Session | RTP中的码流采样率变化的处理规则，音频较常见
[7164](https://tools.ietf.org/html/rfc7164) | RTP and Leap Seconds | RTP时间戳的校准机制
[7201](https://tools.ietf.org/html/rfc7201) | Options for Securing RTP Sessions | RTP的安全机制的建议，什么时候用DTLS，SRTP，ZRTP或者RTP over TLS等
[7202](https://tools.ietf.org/html/rfc7202) | Securing the RTP Framework: Why RTP Does Not Mandate a Single Media Security Solution | RTP的安全机制的补充说明
[7656](https://tools.ietf.org/html/rfc7656) | A Taxonomy of Semantics and Mechanisms for Real-Time Transport Protocol (RTP) Sources | RTP在webrtc中的应用场景
[7667](https://tools.ietf.org/html/rfc7667) | RTP Topologies | 在MCU等复杂系统中，RTP流的设计规范
[7741](https://tools.ietf.org/html/rfc7741) | RTP Payload Format for VP8 Video | 负载为vp8的定义
[7798](https://tools.ietf.org/html/rfc7798) | RTP Payload Format for High Efficiency Video Coding (HEVC) | 负载为HEVC的定义
[8082](https://tools.ietf.org/html/rfc8082) | Using Codec Control Messages in the RTP Audio-Visual Profile with Feedback with Layered Codecs | 基于4585实时RTCP消息，来控制分层的音视频编码器的机制，对于5104协议的补充
[8083](https://tools.ietf.org/html/rfc8083) | Multimedia Congestion Control: Circuit Breakers for Unicast RTP Sessions | RTP的拥塞处理之码流环回的处理
[8108](https://tools.ietf.org/html/rfc8108) | Sending Multiple RTP Streams in a Single RTP Session | 单一会话，单一端口传输所有的RTP/RTCP码流，对现有RTP/RTCP机制的总结
[8285](https://tools.ietf.org/html/rfc8285) | A General Mechanism for RTP Header Extensions | RTP 扩展头定义，可以同时扩展为1或2个字节
