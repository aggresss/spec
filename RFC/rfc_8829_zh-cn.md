# JavaScript Session Establishment Protocol (JSEP)

> 原文 [https://datatracker.ietf.org/doc/html/rfc8829](https://datatracker.ietf.org/doc/html/rfc8829)

## 摘要

本文档描述了 JavaScript 应用程序通过 W3C RTCPeerConnection API 指定的接口来控制多媒体会话信令的机制并讨论与现有信令协议的关系。

## 1. 介绍

本文档介绍了通过 WebRTC 中 RTCPeerConnection 接口控制多媒体会话建立、管理和关闭的方法。

### 1.1. JSEP 总体设计

WebRTC 呼叫建立的设计重点关注于媒体层面，而信令层面的行为则尽可能的留给应用程序。其根本原因是不同的应用程序在信令层会使用不同的协议，例如现存的 SIP 呼叫协议或者为特定应用程序定制化的协议（可能是一个新的用例）。在这种实现中，需要交换的关键信息是媒体回话描述，它指定了传输参数和媒体配置信息。

考虑到这些因素，本文档描述了 JavaScript 会话建立协议(JSEP)，它允许通过 JavaScript 完全控制信令状态机。如上所述，JSEP 假设存在一个模型，在这个模型中，JavaScript 应用程序在包含 WebRTC API 的运行时中执行 “JSEP 实现”。JSEP 的实现几乎完全脱离了核心信令流，它由 JavaScript 使用两个接口来处理:

1. 传递本地和远程会话描述；
2. 与 ICE 状态机交互。JSEP 实现和 JavaScript 应用程序的组合在整个文档中被称为“JSEP 端点”；

在本文档中描述了JSEP的使用默认为发生在两个 JSEP 端点之间。但请注意，在许多情况下它实际上是在 JSEP 端点和某些类型的服务器(如网关或多点控制单元(MCU))之间。这种区别对于JSEP 端点是透明的；它只是遵循通过 API 给出的指令。

JSEP 对会话描述的处理简单而直接。每当需要交换 offer/answer 时，发起端通过调用 createOffer API 来创建 offer，然后应用程序使用这个 offer 通过 setLocalDescription API 来设置其本地配置。 offer 最终通过其首选的信令机制（如 websocket）发送到远端；在收到 offer 后，远端使用 setRemoteDescription API 来设置这个 offer。

为了完成 offer/answer 交换，远端使用 createAnswer API 生成适应的应答，使用 setLocalDescription API 来应用它，并通过信令通道将应答发送回发起方，当发起方获得这个应答时，它使用 setRemoteDescription API 应用它，初始化就完成了。这个过程可以重复进行额外的 offer/answer 交换。

对于 ICE，JSEP 将 ICE 状态机从整个信令状态机中解耦出来。ICE 状态机必须保留在 JSEP 实现中，因为只是实现具有候选对象和其他传输信息的必要信息。通过这种分离增加了协议的灵活性，可以将会话描述从传输中解耦。例如，在传统的 SIP 中，每个 offer 和 answer 都是自包含的，包括会话描述和传输参数。然而，[RFC8840] 允许 SIP 与 Trickle ICE 一起使用，其中会话描述可以立即发送，传输参数可以在可用时发送。单独发送传输参数可以更快地启动 ICE 和 DTLS，因为 ICE 检查可以在任一传输信息可用时立即启动而不是等待所有的传输参数。JSEP 对 ICE 和信令状态机的解耦使其能够适应以上两种类型。

尽管 JSEP 抽象了信令，但它要求应用程序知道信令的处理过程。虽然应用程序不需要理解会话描述的内容，但应用程序必须在正确的时机掉用正确的 API，将会话描述和 ICE 信息转换为其信令协议所定义的消息，并对它从另一端接收到的消息执行解析。

简化应用程序的一种方法是提供一个 JavaScript 库，向开发人员隐藏这种复杂性；该库将实现信令协议及其状态机并且完成代码序列化，从而为应用程序开发人员提供更高级别的面相调用的接口。例如，库可以在 JSEP API 之上提供 SIP 和 XMPP 信令协议的支持。因此，JSEP 为有经验的开发人员提供了更大的控制，而不会给新手带来任何额外的复杂性。

### 1.2. 其他方法考量

一种替代 JSEP 的方法是包含一个轻量级的信令协议。API 将产生并使用来自该协议的消息，而不是向 API 提供会话描述。这样虽然提供了一个更高级的API，但在 JSEP 实现中增加了对信令的更多控制，迫使它必须理解和处理各种异常状态(参见 [RFC3264]，第4节)。

第二种考虑但没有选择的方法是将媒体控制对象的管理与会话描述分离，取而代之的是提供可以直接控制每个组件的 API。这一提议被否决了，理由是要求应用程序开发人员暴露这种级别的复杂性对他们没有好处：

1. 这种方法会产生一个即使是一个简单的例子也需要大量代码来协调所有交互的 API；
2. 其次还会创建一个非常大的并且需要维护一致性的 API 层。此外，可以以任何顺序调用这些 API，导致与媒体子系统的交互集比 JSEP 方法更复杂，而JSEP 方法指定了如何评估和应用会话描述；

JSEP 的一种变体是保留基本的面向会话描述的 API，但将生成 offer 和 answer 的机制移出 JSEP 实现。该方法将开放 getCapabilities API，而不提供 createOffer/createAnswer 方法，getCapabilities API 将向应用程序提供生成自己的会话描述所需的信息。这增加了应用程序的工作量；它需要知道如何从功能集生成会话描述，特别是如何从任意 offer 和支持的功能集生成正确的 answer。虽然这可以通过使用类似于上面提到的库来解决，但它基本上迫使我们使用该库，即使是一个简单的例子。提供 createOffer/createAnswer 可以避免这个问题。

### 1.3. 关于 "bundle-only" 和 "m=" 的矛盾

自从 WebRTC 规范文档被批准以来，IETF 已经意识到指定 JSEP 的文档和指定 BUNDLE 的文档(该 RFC 和 [RFC8843])之间的不一致。这些文档不是为了达成一项决议而进一步推迟公布，而是按照最初批准的方式予以公布。IETF 打算重新启动这些工作，一旦有了解决方案，这些文档的修订版将会发布。

具体的问题涉及到 "m=" 部分被指定为 "bundle-only"，将在本 RFC 4.1.1 节讨论。目前，JSEP 和 BUNDLE 之间存在分歧，这些规范和现有的浏览器实现之间也存在分歧:

- JSEP 规定，"m=" 应该使用端口 0，并在初始 offer 中添加 "a=bundle-only" 属性，而不是在 answer 或后续 offer 中；
- BUNDLE 规定，"m=" 部分应该像前面所描述的那样标记，但是是在所有的 offer 和 answer 中。
- 当前大多数浏览器不标记任何端口为 0 的 "m="，而是为所有捆绑的 "m=" 使用相同的端口；其他则遵循 JSEP 定义。

## 2. 术语

本文件中的关键词“必须”、“不得”、“必需”、“应”、“不应”、“建议”、“不建议”、“可”和“可选”在所有大写字母出现时（如图所示）应按照 BCP 14[RFC2119] [RFC8174] 所述进行解释。

## 3. 语义和语法

### 3.1. 信令模型

JSEP 没有指定特定的信令模型或状态机，除了一般需要以 [RFC3264] (offer/answer)描述的方式交换会话描述，以便会话双方知道如何进行会话。JSEP 提供了创建 offer 和 answer 以及将它们应用到会话的机制。然而，JSEP 实现完全与实际机制解耦，这些 offer 和 answer 通过这种机制传递到远程端，包括寻址、重传、分叉和冲突处理。这些问题完全取决于应用程序；应用程序可以完全控制哪些 offer 和 answer 将提交给实现，以及何时提交。

```text
      +-----------+                               +-----------+
      |  Web App  |<--- App-Specific Signaling -->|  Web App  |
      +-----------+                               +-----------+
            ^                                            ^
            |  SDP                                       |  SDP
            V                                            V
      +-----------+                                +-----------+
      |   JSEP    |<----------- Media ------------>|   JSEP    |
      |   Impl.   |                                |   Impl.   |
      +-----------+                                +-----------+
```

图1：JSEP 信令模型

### 3.2. 会话描述和状态机

为了建立媒体交互，JSEP 实现需要特定的参数来指示要向远端发送什么，以及如何处理接收到的媒体。这些参数是由 offer 和 answer 中会话描述的交换决定的，并且这个过程的某些细节必须在 JSEP API 中处理。

会话描述是否适用于本地或远端将影响该描述的含义。例如，发送给远端的编解码列表表明了本地方愿意接收的内容，当与远端支持的编解码集相交叉时，该列表指定了远程方应该发送的内容。然而，并不是所有的参数都遵循这一规则；有些参数是声明性的，远端必须接受或完全拒绝它们。这种参数的一个例子是 TLS 指纹 [RFC8122]，在 DTLS [RFC6347] 的上下文中使用；这些指纹是根据提供的本地证书计算的，不受协商的影响。

此外，不同的 RFC 对 offer 和 answer 的格式提出了不同的条件。例如，offer 可以提出任意数量的 “m=” 部分(即，媒体描述如 [RFC4566]，5.14 节所述)，但 answer 必须包含与要约完全相同的数字。

最后，虽然确切的媒体参数只有在一个 offer 和一个 answer 交换之后才知道，但 offer 方可能会在收到 answer 之前收到 ICE 检查和可能的媒体(例如，在一个连接建立后的重新 offer)。在这种情况下，为了正确处理传入的媒体，offer 方的媒体处理程序必须在 answer 到达之前了解 offer 的细节。

因此，为了正确处理会话描述，JSEP实现需要:

1. 了解会话描述是属于本端还是远端。
2. 了解一个会话描述是一个 offer 还是一个 answer。
3. 允许 offer 独立于 answer 而指定。

JSEP 通过添加 setLocalDescription 和 setRemoteDescription 方法来解决这个问题，并让会话描述对象包含一个类型字段，用来指示所提供的会话描述的类型。这满足了上面列出的要求，发起者首先调用 setLocalDescription(sdp [offer])，然后再调用 setRemoteDescription(sdp [answer])。对于应答者，首先调用 setRemoteDescription(sdp [offer])，然后再调用 setLocalDescription(sdp [answer])。

在交换 offer/answer 的过程中，未完成的 offer 在发起方和应答方看来是 “未决的”，因为它可能被接受或拒绝。如果这是一个重新 offer，每一方还将有“当前”的本地和远程描述，这反映了最后的 offer/answer 交换的结果。章节 4.1.14、4.1.16、4.1.13 和4.1.15 提供了关于待定和当前描述的更多细节。

JSEP 还允许应用程序将 answer 视为临时 answer。临时 answer 为应答方提供了一种将初始会话参数反馈给发起者的方法，以允许会话开始，同时允许稍后指定最终 answer。最终 answer 的概念对于 offer/answer 模式非常重要；当接收到这样一个 answer 时，调用者分配的任何额外资源都可以被释放，并且确切的会话配置已经知道了。这些“资源”可以包括额外的 ICE 组件、候选 TURN 或视频解码集。另一方面，临时的 answer 没有这样的重新分配；因此，多个不同的临时 answer，有自己的编解码选择，传输参数等，可以在呼叫设置期间接收和应用。请注意，最终 answer 本身可能与任何收到的 answer 不同。

在 [RFC3264] 中，信令级别的约束是一个给定的会话只能有一个 offer，但是在 JSEP 级别，一个新的 offer 可以在任何点生成。例如，当使用 SIP 信令时，如果发送了一个 offer，然后使用 SIP CANCEL 取消，则可以生成另一个 offer，即使没有收到第一个 offer 的答复。为了支持这一点，当 JavaScript 应用程序需要一个 offer 作为信令时，JSEP 媒体层可以通过 createOffer 方法提供一个 offer。应答者可以返回零个或多个临时 answer，然后通过发送一个最终 answer 来结束 offer/answer 交换。这个的状态机如下:

```text
                    setRemote(OFFER)               setLocal(PRANSWER)
                        /-----\                               /-----\
                        |     |                               |     |
                        v     |                               v     |
         +---------------+    |                +---------------+    |
         |               |----/                |               |----/
         |  have-        | setLocal(PRANSWER)  | have-         |
         |  remote-offer |------------------- >| local-pranswer|
         |               |                     |               |
         |               |                     |               |
         +---------------+                     +---------------+
              ^   |                                   |
              |   | setLocal(ANSWER)                  |
setRemote(OFFER)  |                                   |
              |   V                  setLocal(ANSWER) |
         +---------------+                            |
         |               |                            |
         |               |<---------------------------+
         |    stable     |
         |               |<---------------------------+
         |               |                            |
         +---------------+          setRemote(ANSWER) |
              ^   |                                   |
              |   | setLocal(OFFER)                   |
setRemote(ANSWER) |                                   |
              |   V                                   |
         +---------------+                     +---------------+
         |               |                     |               |
         |  have-        | setRemote(PRANSWER) |have-          |
         |  local-offer  |------------------- >|remote-pranswer|
         |               |                     |               |
         |               |----\                |               |----\
         +---------------+    |                +---------------+    |
                        ^     |                               ^     |
                        |     |                               |     |
                        \-----/                               \-----/
                    setLocal(OFFER)               setRemote(PRANSWER)
```

图2：JSEP 状态机

除了这些状态转换，处理 临时 answer 和最终 answer 没有区别。

### 3.3. 回话描述格式

JSEP 的会话描述使用会话描述协议(session Description Protocol, SDP)语法进行内部表示。虽然这种格式对于 JavaScript 操作不是最佳的，但它被广泛接受并经常更新新特性；任何会话描述的替代表示都必须与 SDP 的变化保持同步，至少在这种新的表示取代 SDP 流行之前是如此。

然而，为了提供未来的灵活性，SDP 语法被封装在 SessionDescription 对象中，该对象可以从 SDP 构造并序列化到 SDP。如果未来的规范对会话描述采用 JSON 格式达成一致，我们可以很容易地让这个对象生成并使用 JSON。

如下所述，大多数应用程序应该能够将这些各种 API 调用产生和使用的 sessiondescription 视为不透明的 blob；也就是说，应用程序不需要读取或更改它们。

### 3.4. 回话描述控制

为了让应用程序控制各种公共会话参数，JSEP 提供了控制面，告诉 JSEP 实现如何生成会话描述。在大多数情况下，这避免了 JavaScript 修改会话描述的需要。

对这些对象的更改将导致对后续 createOffer/createAnswer 调用生成的会话描述的更改。

#### 3.4.1. RtpTransceivers

RtpTransceivers 允许应用程序控制与一个 “m=” section 相关联的 RTP media。每个 RtpTransceiver 有一个 RtpSender 和 一个RtpReceiver，应用程序可以使用它们来控制 RTP media 的发送和接收。应用程序也可以直接修改 RtpTransceiver，例如停止操作。

RtpTransceivers 通常与 "m=" section 是 1:1 的映射，尽管可能会有比 "m=" section 更多的 RtpTransceivers ，例如：当RtpTransceivers 被创建但还没有关联到"m=" section，或者如果 RtpTransceivers 已经停止并从 "m=" section 中分离。如果 RtpTransceiver 的 mid (media identification) 属性是非空的，则 RtpTransceiver 被认为与 “m=”section 相关联；否则它被认为是游离的。关联的 “m=” section 是在创建一个 offer 或应用一个远程 offer 时使用收发器和 “m=” section 索引之间的映射确定的。

一个 RtpTransceiver 从来不会与一个以上的 "m=" section 相关联，并且一旦会话描述被应用，一个 "m=" section 总是与一个 RtpTransceiver 相关联。然而，在某些情况下，当一个 “m=” section 被拒绝时，正如下面 5.2.2 节中讨论的那样，“m=” section 将被“回收”，RtpTransceiver 将与一个新的 MID 值相关联。

RtpTransceivers 可以由应用程序显式创建，也可以通过调用 setRemoteDescription 来隐式创建，并提供一个新的 “m=” section。

#### 3.4.2. RtpSenders

RtpSenders 允许应用程序控制 RTP 媒体的发送方式。RtpSender 在概念上负责由 “m=” section 描述输出的 RTP 流。这包括编码附加的 MediaStreamTrack，发送 RTP 媒体包，以及关联的 RTCP。

#### 3.4.3. RtpReceivers

RtpReceivers 允许应用程序检查如何接收 RTP 媒体。RtpReceiver 在概念上负责传入的 RTP 流，由 “m=” section 描述。这包括处理接收到的RTP媒体包，解码传入的流以产生远程MediaStreamTrack，并为传入的 RTP 流生成和处理 RTCP。

### 3.5. ICE

#### 3.5.1. ICE Gathering 概述

JSEP 根据应用程序的需要收集 ICE candidates。ICE candidates 的收集被称为收集阶段，这是由添加一个新的或回收的 “m=” section 到本地会话描述或描述中新的 ICE 凭证触发的，表明 ICE 重启。新 ICE 凭据的使用可以由应用程序显式触发，也可以由 JSEP 实现隐式触发，以响应 ICE 配置中的更改。

当 ICE 配置发生变化，需要一个新的收集阶段时，就会设置一个 “needs-ice-restart” 标识。设置了这个标识后，调用 createOffer API 将生成新的 ICE 凭证。这个标识通过调用setLocalDescription API 来清除，该 API 带有来自 offer 或 answer 的新 ICE 凭证，即本地或远程发起的ICE重启。

当一个新的收集阶段开始时，ICE 代理将通过状态更改事件通知应用程序正在进行收集。然后，当每个新的 ICE candidates 可用时，ICE 代理将通过一个 oniccandidate 事件通知应用程序；这些候选对象也将自动添加到当前或者挂起的本地会话描述中。最后，当收集完所有的 ICE candidates 后，将发送一个最后的 oniccandidate 事件，以表明收集过程已经完成。

注意，收集阶段只收集 new/recycled/restaring 状态的 "m=" section 所需的候选；其他 “m=” section 继续使用它们现有的候选项。同样，如果一个 "m=" section 被绑定(通过一个成功的bundle 协商或者被标记为 bundle-only)，那么当且仅当它的 MID 项是一个 bundle 标签时，候选的 "m=" section 将被收集并交换，如 [RFC8843] 所述。




