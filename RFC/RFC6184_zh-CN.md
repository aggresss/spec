# RTP Payload Format for H.264 Video

> 原文 [https://datatracker.ietf.org/doc/html/rfc6184](https://datatracker.ietf.org/doc/html/rfc6184)

## 摘要

本备忘录描述了 ITU-T 建议 H.264 视频编解码器和技术上相同的 ISO/IEC 国际标准 14496-10 视频编解码器的 RTP 有效载荷格式，不包括可伸缩视频编码(SVC)扩展和多视图视频编码扩展，其中 RTP 有效载荷格式在其他地方定义。RTP 有效载荷格式允许在每个 RTP 有效载荷中对一个或多个网络抽象层单元(NALUs)进行分组，这些单元由 H.264 视频编码器生成。有效载荷格式具有广泛的适用性，因为它支持从简单的低比特率会话使用，到交互传输的互联网视频流，到高比特率视频点播的应用程序。

本备忘录废止 [RFC3984]。第14节总结了 [RFC3984] 的变化。关于 [RFC3984] 的向后兼容性的问题将在第 15 节中讨论。
