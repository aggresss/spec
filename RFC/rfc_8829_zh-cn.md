# JavaScript Session Establishment Protocol (JSEP)

> 原文 [https://datatracker.ietf.org/doc/html/rfc8829](https://datatracker.ietf.org/doc/html/rfc8829)
## 摘要

本文档描述了 JavaScript 应用程序通过 W3C RTCPeerConnection API 指定的接口来控制多媒体会话信令的机制并讨论与现有信令协议的关系。

## 1 介绍

本文档介绍了通过 WebRTC 中 RTCPeerConnection 接口控制多媒体会话建立、管理和关闭的方法。

### 1.1 JSEP 总体设计

WebRTC 呼叫建立的设计重点关注于媒体层面，而信令层面则尽可能的留给应用程序。

