# JavaScript Session Establishment Protocol (JSEP)

> 原文 [https://datatracker.ietf.org/doc/html/rfc8829](https://datatracker.ietf.org/doc/html/rfc8829)

## 摘要

本文档描述了 JavaScript 应用程序通过 W3C RTCPeerConnection API 指定的接口来控制多媒体会话信令的机制并讨论与现有信令协议的关系。

## 1 介绍

本文档介绍了通过 WebRTC 中 RTCPeerConnection 接口控制多媒体会话建立、管理和关闭的方法。

### 1.1 JSEP 总体设计

WebRTC 呼叫建立的设计重点关注于媒体层面，而信令层面的行为则尽可能的留给应用程序。其根本原因是不同的应用程序在信令层会使用不同的协议，例如现存的 SIP 呼叫协议或者为特定应用程序定制化的协议（可能是一个新的用例）。在这种实现中，需要交换的关键信息是媒体回话描述，它指定了传输参数和媒体配置信息。

考虑到这些因素，本文档描述了 JavaScript 会话建立协议(JSEP)，它允许通过 JavaScript 完全控制信令状态机。如上所述，JSEP 假设存在一个模型，在这个模型中，JavaScript 应用程序在包含 WebRTC API 的运行时中执行 “JSEP 实现”。JSEP 的实现几乎完全脱离了核心信令流，它由 JavaScript 使用两个接口来处理:

- (1) 传递本地和远程会话描述；
- (2) 与 ICE 状态机交互。JSEP 实现和 JavaScript 应用程序的组合在整个文档中被称为“JSEP 端点”；

在本文档中描述了JSEP的使用默认为发生在两个 JSEP 端点之间。但请注意，在许多情况下它实际上是在 JSEP 端点和某些类型的服务器(如网关或多点控制单元(MCU))之间。这种区别对于JSEP 端点是透明的；它只是遵循通过 API 给出的指令。

JSEP 对会话描述的处理简单而直接。每当需要交换 offer/answer 时，发起端通过调用 createOffer API 来创建 offer，然后应用程序使用这个 offer 通过 setLocalDescription API 来设置其本地配置。 offer 最终通过其首选的信令机制（如 websocket）发送到远端；在收到 offer 后，远端使用 setRemoteDescription API 来设置这个 offer。

为了完成 offer/answer 交换，远端使用 createAnswer API 生成适应的应答，使用 setLocalDescription API 来应用它，并通过信令通道将应答发送回发起方，当发起方获得这个应答时，它使用 setRemoteDescription API 应用它，初始化就完成了。这个过程可以重复进行额外的 offer/answer 交换。

对于 ICE，JSEP 将 ICE 状态机从整个信令状态机中解耦出来。ICE 状态机必须保留在 JSEP 实现中，因为只是实现具有候选对象和其他传输信息的必要信息。通过这种分离增加了协议的灵活性，可以将会话描述从传输中解耦。例如，在传统的 SIP 中，每个 offer 和 answer 都是自包含的，包括会话描述和传输参数。然而，RFC8840 允许 SIP 与 Trickle ICE 一起使用，其中会话描述可以立即发送，传输参数可以在可用时发送。单独发送传输参数可以更快地启动 ICE 和 DTLS，因为 ICE 检查可以在任一传输信息可用时立即启动而不是等待所有的传输参数。JSEP 对 ICE 和信令状态机的解耦使其能够适应以上两种类型。


