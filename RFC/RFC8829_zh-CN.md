# JavaScript Session Establishment Protocol (JSEP)

> 原文 [https://datatracker.ietf.org/doc/html/rfc8829](https://datatracker.ietf.org/doc/html/rfc8829)

## 摘要

本文档描述了 JavaScript 应用程序通过 W3C RTCPeerConnection API 指定的接口来控制多媒体会话信令的机制并讨论与现有信令协议的关系。

## 1. 介绍

本文档介绍了通过 WebRTC 中 RTCPeerConnection 接口控制多媒体会话建立、管理和关闭的方法。

### 1.1. JSEP 总体设计

WebRTC 呼叫建立的设计重点关注于媒体层面，而信令层面的行为则尽可能的留给应用程序。其根本原因是不同的应用程序在信令层会使用不同的协议，例如现存的 SIP 呼叫协议或者为特定应用程序定制化的协议（可能是一个新的用例）。在这种实现中，需要交换的关键信息是媒体描述，它指定了传输参数和媒体配置信息。

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

具体的问题涉及到 m-section 被指定为 "bundle-only"，将在本 RFC 4.1.1 节讨论。目前，JSEP 和 BUNDLE 之间存在分歧，这些规范和现有的浏览器实现之间也存在分歧:

- JSEP 规定，m-section 应该使用端口 0，并在初始 offer 中添加 "a=bundle-only" 属性，而不是在 answer 或后续 offer 中；
- BUNDLE 规定，m-section 应该像前面所描述的那样标记，但是是在所有的 offer 和 answer 中。
- 当前大多数浏览器不标记任何端口为 0 的 m-section，而是为所有捆绑的 m-section 使用相同的端口；其他则遵循 JSEP 定义。

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

此外，不同的 RFC 对 offer 和 answer 的格式提出了不同的条件。例如，offer 可以提出任意数量的 m-section(即，媒体描述如 [RFC4566]，5.14 节所述)，但 answer 必须包含与要约完全相同的数字。

最后，虽然确切的媒体参数只有在一个 offer 和一个 answer 交换之后才知道，但 offer 方可能会在收到 answer 之前收到 ICE 检查和可能的媒体(例如，在一个连接建立后的重新 offer)。在这种情况下，为了正确处理传入的媒体，offer 方的媒体处理程序必须在 answer 到达之前了解 offer 的细节。

因此，为了正确处理会话描述，JSEP 实现需要:

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

### 3.3. 会话描述格式

JSEP 的会话描述使用会话描述协议(session Description Protocol, SDP)语法进行内部表示。虽然这种格式对于 JavaScript 操作不是最佳的，但它被广泛接受并经常更新新特性；任何会话描述的替代表示都必须与 SDP 的变化保持同步，至少在这种新的表示取代 SDP 流行之前是如此。

然而，为了提供未来的灵活性，SDP 语法被封装在 SessionDescription 对象中，该对象可以从 SDP 构造并序列化到 SDP。如果未来的规范对会话描述采用 JSON 格式达成一致，我们可以很容易地让这个对象生成并使用 JSON。

如下所述，大多数应用程序应该能够将这些各种 API 调用产生和使用的 sessiondescription 视为不透明的 blob；也就是说，应用程序不需要读取或更改它们。

### 3.4. 会话描述控制

为了让应用程序控制各种公共会话参数，JSEP 提供了控制面，告诉 JSEP 实现如何生成会话描述。在大多数情况下，这避免了 JavaScript 修改会话描述的需要。

对这些对象的更改将导致对后续 createOffer/createAnswer 调用生成的会话描述的更改。

#### 3.4.1. RtpTransceivers

RtpTransceivers 允许应用程序控制与一个 m-section 相关联的 RTP media。每个 RtpTransceiver 有一个 RtpSender 和 一个RtpReceiver，应用程序可以使用它们来控制 RTP media 的发送和接收。应用程序也可以直接修改 RtpTransceiver，例如停止操作。

RtpTransceivers 通常与 m-section 是 1:1 的映射，尽管可能会有比 m-section 更多的 RtpTransceivers ，例如：当RtpTransceivers 被创建但还没有关联到 m-section，或者如果 RtpTransceivers 已经停止并从 m-section 中分离。如果 RtpTransceiver 的 mid (media identification) 属性是非空的，则 RtpTransceiver 被认为与 m-section 相关联；否则它被认为是游离的。关联的 m-section 是在创建一个 offer 或应用一个远程 offer 时使用收发器和 m-section 索引之间的映射确定的。

一个 RtpTransceiver 从来不会与一个以上的 m-section 相关联，并且一旦会话描述被应用，一个 m-section 总是与一个 RtpTransceiver 相关联。然而，在某些情况下，当一个 m-section 被拒绝时，正如下面 5.2.2 节中讨论的那样，m-section 将被“回收”，RtpTransceiver 将与一个新的 MID 值相关联。

RtpTransceivers 可以由应用程序显式创建，也可以通过调用 setRemoteDescription 来隐式创建，并提供一个新的 m-section。

#### 3.4.2. RtpSenders

RtpSenders 允许应用程序控制 RTP 媒体的发送方式。RtpSender 在概念上负责由 m-section 描述输出的 RTP 流。这包括编码附加的 MediaStreamTrack，发送 RTP 媒体包，以及关联的 RTCP。

#### 3.4.3. RtpReceivers

RtpReceivers 允许应用程序检查如何接收 RTP 媒体。RtpReceiver 在概念上负责传入的 RTP 流，由 m-section 描述。这包括处理接收到的 RTP 媒体包，解码传入的流以产生远程 MediaStreamTrack，并为传入的 RTP 流生成和处理 RTCP。

### 3.5. ICE

#### 3.5.1. ICE Gathering 概述

JSEP 根据应用程序的需要收集 ICE candidates。ICE candidates 的收集被称为收集阶段，这是由添加一个新的或回收的 m-section 到本地会话描述或描述中新的 ICE 凭证触发的，表明 ICE 重启。新 ICE 凭据的使用可以由应用程序显式触发，也可以由 JSEP 实现隐式触发，以响应 ICE 配置中的更改。

当 ICE 配置发生变化，需要一个新的收集阶段时，就会设置一个 “needs-ice-restart” 标识。设置了这个标识后，调用 createOffer API 将生成新的 ICE 凭证。这个标识通过调用setLocalDescription API 来清除，该 API 带有来自 offer 或 answer 的新 ICE 凭证，即本地或远程发起的ICE重启。

当一个新的收集阶段开始时，ICE 代理将通过状态更改事件通知应用程序正在进行收集。然后，当每个新的 ICE candidates 可用时，ICE 代理将通过一个 oniccandidate 事件通知应用程序；这些候选对象也将自动添加到当前或者挂起的本地会话描述中。最后，当收集完所有的 ICE candidates 后，将发送一个最后的 oniccandidate 事件，以表明收集过程已经完成。

注意，收集阶段只收集 new/recycled/restaring 状态的 m-section 所需的候选；其他 m-section 继续使用它们现有的候选项。同样，如果一个 m-section 被绑定(通过一个成功的bundle 协商或者被标记为 bundle-only)，那么当且仅当它的 MID 项是一个 bundle 标签时，候选的 m-section 将被收集并交换，如 [RFC8843] 所述。

#### 3.5.2. ICE Candidate Trickling

Candidate Trickling 是一种技术方案，通过它呼叫者可以在发出初始 offer 后，逐步地向被呼叫者提供 candidate；“Trickle ICE” 的语义在 [RFC8838] 中定义。这个过程允许被调用方开始对调用开始操作，并立即建立 ICE (可能还有 DTLS )连接，而不必等待调用方收集所有可能的候选连接。在启动调用之前没有执行收集的情况下，这将导致更快的媒体设置。

JSEP 通过提供前文提到的 API 来支持可选的候选传入，这些 API 对 ICE 候选收集过程提供控制和反馈。支持 Trickle ICE 的应用程序可以立即发送初始 offer，并在收到新 candidate 的通知时发送单个 candidate；不支持此功能的应用程序可以简单地等待收集完成的指示，然后创建和发送携带所有 candidate 的 offer。

在收到少量的 candidate 后，接收人将把他们提供给 ICE 代理。这将触发 ICE 代理开始使用新的远程候选连接检查。

##### 3.5.2.1. ICE Candidate 格式

在 JSEP中，ICE 候选对象被一个 iccandidate 对象抽象，与会话描述一样，内部表示使用 SDP 语法。

候选的详细信息在 iccandidate 字段中指定，使用与 [RFC8839] 5.1 节中定义的 "candidate-attribute" 字段相同的 SDP 语法。注意，该字段不包含 "a=" 前缀，如下例所示:

```text
candidate:1 1 UDP 1694498815 192.0.2.33 10000 typ host
```

iccandidate 对象包含一个字段，用来指明它与哪个 ICE 用户名片段(ufrag)相关联，如 [RFC8839] 5.4 节中定义的那样。该值用于确定该 iccandidate 属于哪个会话描述(以及哪个收集阶段)，这有助于解决 ICE 重启时的歧义。如果这个字段在接收到的 iccandidate 中不存在(可能是在与非 JSEP 端点通信时)，则假定为最近接收到的会话描述。

可以通过一个 m-section 索引或者 MID 这两种方式中的一种确认 iceCandidate 对象与 m-section 是相关联的，m-section 索引是一个从零开始的索引,索引 N 指的第 N + 1 m-section 的会话描述引用的这个 IceCandidate。 MID 是一个 “media stream identification” 的值，在[RFC5888] 第4章中定义，它提供了一种更健壮的方法来标识会话描述中的 m-section，使用关联的 RtpTransceiver 对象的MID(它可能是由应答者在与不支持 MID 属性的非 JSEP 端点交互时本地生成的，如下面 5.10 节所讨论的)。如果在接收到的 iccandidate 中有 MID 字段，它必须被用于识别;否则，将使用 m-section 索引。如上所述，实现时必须考虑到接收对象缺少某些字段的情况。

#### 3.5.3. ICE Candidate 策略

通常，在收集 ICE Candidate 对象时，JSEP 实现将收集初始候选对象的所有可能形式 —— host/server-reflexive/relay。但在某些情况下，出于隐私或相关考虑，应用程序可能希望对收集过程拥有更具体的控制。例如，可能希望只使用中继候选以尽可能少地泄漏位置信息(需要注意，这种选择会带来相应的操作成本)。为了实现这一点，JSEP 允许应用程序限制在会话中使用哪些 ICE Candidate。请注意，这个过滤是在任何限制的基础上应用的，如 [RFC8828] 中所讨论的，该实现可以强制选择哪些 IP 地址用于应用程序。

在某些情况下，应用程序可能希望更改会话处于活动状态时使用的候选类型。一个主要的例子是，接受会话者最初可能希望只使用中继候选者，以避免将位置信息泄露给任意的呼叫者，一旦用户表示他们想接这个呼叫，就会更改为使用所有的 Candidate(为了降低操作成本)。对于这个场景，JSEP 实现必须允许在会话中期更改候选策略，这取决于前面提到的与本地策略的交互。

为了管理 ICE Candidate 策略，JSEP 实现将在每个收集阶段开始时确定当前设置。然后，在收集阶段，实现不可以将当前策略不允许的候选对象公开给应用程序，使用它们作为连接检查的源，或者通过其他字段间接公开它们，例如其他 ICE Candidate 对象的 raddr/rport 属性。之后，如果应用程序指定了不同的策略，应用程序可以通过 ICE 重启来启动一个新的收集阶段并应用它。

#### 3.5.4. ICE Candidate Pool

JSEP 应用程序通常通过提供给 setLocalDescription 的信息通知 JSEP 实现开始收集 ICE，因为本地描述指示了需要的 ICE 组件的数量，以及必须为哪些候选组件收集。但是，为了加速应用程序提前知道要使用的 ICE 组件数量的情况，它可能会要求实现收集潜在的 ICE 候选组件池，以确保快速设置媒体。

当 setLocalDescription 最终被调用并且 JSEP 实现准备收集所需的 ICE Candidate 时，它应该首先检查池中是否有可用的候选。如果候选池中有 Candidate，他们应该通过ICE Candidate 活动立即提交申请。如果池耗尽，要么是因为使用的 ICE 组件数量超过预期，要么是因为池没有足够的时间收集候选项，那么剩余的候选项将像往常一样收集。这只发生在第一次 offer/answer 交换，之后候选池被清空并且不再使用。

这个概念有用的一个例子是，一个应用程序希望在将来的某个时间点收到一个传入的呼叫，并希望将建立连接所需的时间最小化，以避免剪切初始媒体。通过将候选对象预先聚集到池中，它可以在收到呼叫后几乎立即交换并开始从这些候选对象发送连接检查。不过请注意，通过保留这些预先收集的候选对象(只要需要它们，它们就会一直保持活跃)，应用程序将消耗它正在使用的 STUN/TURN 服务器上的资源。

#### 3.5.5. ICE 版本

虽然该规范在形式上依赖于 [RFC8445]，但在其发布时，大多数 WebRTC 实现都支持 [RFC5245] 中描述的 ICE 版本。在 [RFC8445] 中定义的 “ice2” 属性可以用来检测远程终端使用的版本，并提供从旧规范到新规范的平稳过渡。实现必须能够接受没有 "ice2" 属性的远程描述。

### 3.6. 视频大小协商

视频大小协商是接收端可以使用 "a=imageattr" SDP属性 [RFC6236] 来指示它能够接收的视频帧大小的过程。接收端可以对其视频解码能处理的内容有硬性限制，或者它可能有一些策略设置的最大值。通过在 "a=imageattr" 属性中指定这些限制，JSEP 端点可以尝试确保远程发送方以可接受的分辨率传输视频。然而，当与不支持此属性的非 JSEP 端点通信时，可能会超过任何信令限制，而 JSEP 实现必须合理地处理此限制，例如，丢弃视频。

需要注意，某些编解码器支持传输宽高比不是1.0(即非正方形像素)的样本。JSEP 实现将不发送非正方形像素，但应该接收和渲染这样的视频与正确的宽高比。然而，样本宽高比对尺寸协商没有影响；无论是否正方形，所有尺寸都以像素为单位测量。

#### 3.6.1. 创建 imageattr 属性

接收器将首先结合任何已知的本地限制(例如，硬件解码器能力或本地策略)，以确定它可以接收的绝对最小和最大尺寸。如果没有已知的局部限制，"a=imageattr" 属性应该被省略。如果这些局部限制排除了接收任何视频，例如，不允许分辨率的简并情况下，"a=imageattr" 属性必须被省略，如果合适的话，m-section 必须被标记为 sendonly/inactive。

否则，一个 "a=imageattr" 属性被创建为一个 "recv" 方向，并且由前面提到的交集形成的分辨率空间被用来指定它的最小和最大的 "x=" 和 "y=" 值。

这里的规则表示一组单独的首选项，因此，"a=imageattr" 中 "q=" 值并不重要。应该设置为 "1.0"。

"a=imageattr" 字段是特定于负载类型的。当所有支持的视频编解码器具有相同的功能时，建议使用通配符有效负载类型(*)的单一属性。然而，当受支持的视频编解码器有不同的限制时，必须为每种负载类型插入特定的 "a=imageattr" 属性。

例如，考虑一个具有多格式视频解码器的系统，它能够解码从 48x48 到 720p 的任何分辨率。在这种情况下，实现将生成这个属性:

```text
 a=imageattr:* recv [x=[48:1280],y=[48:720],q=1.0]
```

这个声明表明接收器能够解码从 48x48 到 1280x720 像素分辨率的任何图像。

#### 3.6.2. 解析 imageattr 属性

[RFC6236] 将 "a=imageattr" 定义为一个建议的字段。这意味着它并不绝对限制发送方可以使用的视频格式，而是给出了首选值的指示。

这个规范规定了更具体的行为。MediaStreamTrack 的视频分辨率被附加到一个 RtpSender，追踪编码的视频在同一或低分辨率(s)(“编码器分辨率”)，应用和远程描述引用发送方和包含有效的 "= imageattr recv" 属性,它必须遵循以下规则，以确保发送方不会传输超出属性中指定的大小标准的分辨率。只要属性在远程描述中仍然存在，就必须遵循这些规则，包括在 track 改变其分辨率或被不同 track 替换的情况下。

根据 RtpSender 是如何配置的，它可能会产生一个特定分辨率的单一编码，或者，如果同时发送(3.7节)多个已经协商的编码，每个都有自己的特定分辨率。此外，根据配置的不同，每种编码都可以在需要时灵活地减少分辨率，或者锁定到特定的输出分辨率。

对于由 RtpSender 产生的每个编码，远程描述中相应的 m-section 中的 "a=imageattr recv" 属性集将被处理，以确定应该传输什么。仅考虑引用为编码选择的媒体格式的属性；每个这样的属性都是单独计算的，从 "q=" 值最高的属性开始。如果多个属性具有相同的 "q=" 值，则按照它们在包含 "m=" 部分中出现的顺序计算它们。请注意，虽然 JSEP 端点每一种媒体格式最多包含一个 "a=imageattr recv"属性，但 JSEP 端点可以从非 JSEP 端点接收包含多个此类属性的 m-section 的会话描述。

对于每个 "a=imageattr recv" 属性，应用以下规则。如果此处理成功，则相应地传输编码，并且不再考虑该编码的其他属性。否则，将按照前面提到的顺序计算下一个属性。如果所提供的所有属性都不能被成功处理，那么就不能传输编码，并且应该向应用程序抛出一个错误。

- 将该属性的限制与编码器分辨率进行比较。只考虑以下提到的具体限制；任何其他值，比如图片的宽高比，都必须被忽略。当考虑生成旋转视频的 MediaStreamTrack 时，必须使用未旋转的分辨率进行检查。无论接收机是否支持执行接收侧旋转(例如，通过协调视频定向(CVO) [TS26.114])，这都是必需的，因为这大大简化了匹配逻辑。
- 如果属性包含一个“sar=”(样本宽高比)值设置为“1.0”以外的值，表明接收者想要接收非正方形像素，这不能满足，属性绝对不能被使用。
- 如果编码器的分辨率超过属性允许的最大大小，并且允许编码器调整其分辨率，编码器应该应用降尺度以满足限制。一定不要改变图片的宽高比的编码，忽略任何微小的差异，由于舍入。例如，如果编码器的分辨率是1280x720，而属性的最大指定值是640x480，那么预期的输出分辨率将是640x360。如果不能应用降尺度，则绝对不能使用该属性。
- 如果编码器分辨率小于该属性允许的最小大小，则一定不要使用该属性;编码器一定不能应用升级。JSEP实现应该通过允许接收任意小的分辨率(可能通过回退到软件解码器)来避免这种情况。
- 如果编码器分辨率在最大和最小尺寸范围内，则不需要任何操作。

#### 3.7. Simulcast

JSEP 支持 MediaStreamTrack 的 Simulcast 传输，其中媒体源的多个编码可以在一个 m-section 的上下文中传输。当前的 JSEP API 设计用于允许应用程序发送 Simulcast 媒体，但只接收单一编码。这允许在多用户场景中，每个发送客户端向服务器发送多个编码，然后服务器为每个接收客户端选择要转发的适当编码。

应用程序通过在 RtpSender上配置多个编码来请求支持 Simulcast。在生成 offer 或 answer 时，这些编码通过相应的 m-section 上的 SDP 标记表示，如下所述。理解并愿意接收 Simulcast 的接收器也将包括 SDP 标记来表示他们的支持，而 JSEP 端点将使用这些标记来确定是否允许给定的 RtpSender 进行 Simulcast。如果没有协商同步传输支持，RtpSender 将只使用第一个配置的编码。

请注意，Simulcast 的确切参数取决于发送程序。虽然前面提到的 SDP 标记是为了确保远端可以接收和分解多个 Simulcast 的编码，但在 JSEP 中，用于每个编码的具体分辨率和比特率仅由发送端决定。

JSEP 目前还没有提供一种机制来配置 Simulcast 的接收。这意味着，如果远端提供了 simulcast，则 JSEP端点生成的 answer 将不指示是否支持接收simulcast，因此远端将只发送每个 m-section 的单个编码。

此外，JSEP 没有提供处理来自 JSEP 端点 simulcast offer 请求的机制。这意味着，在 JSEP 端点收到初始 offer 的情况下，设置 simulcast 需要带外信令或 SDP 检查。然而，如果 JSEP 端点在其初始 offer 中设置了 simulcast，则任何已建立的 simulcast 流将在收到传入的重新 offer 后继续工作。该规范的未来版本可能会添加额外的 API 来处理传入的初始 offer 场景。

当使用 JSEP 从 RtpSender 发送多个编码时，使用来自 [RFC8853] 和 [RFC8851] 的技术。具体来说，当一个 RtpSender 被配置了多个编码时，RtpSender 的 m-section 将包含一个 "a=simulcast" 属性，正如在 [RFC8853] 5.1 节中定义的那样，带有一个 "send" simulcast 描述并列出了每个期望的编码，而没有 "recv" simulcast 描述。m-section 还将包含每个编码的 "a=rid" 属性，如 [RFC8851] 4 节中指定的；使用限制标识符(rid，也称为 rid-ids 或 RtpStreamIds )可以消除各个编码的歧义，即使它们都属于同一个 m-section。

#### 3.8. 与 forking 的相互作用

一些呼叫信令系统允许各种类型的 forking ，一个 SDP offer 可以提供给多个设备。例如，SIP [RFC3261] 同时定义了 “并行搜索” 和 “顺序搜索”。尽管这些主要是信令层问题，不在 JSEP 的范围内，但它们确实对媒体相关的配置有一些影响。当 forking 发生在信令层时，负责信令的 JavaScript 应用程序需要决定在什么时间点应该发送或接收什么媒体，以及应该与哪个远程端点通信；JSEP 用于确保媒体引擎能够按照应用程序的要求生成 RTP stream 并执行操作。应用程序可以让媒体引擎做的基本操作如下:

- 开始与给定的远程对等端交换媒体，但保留 offer 中的所有资源。
- 开始与给定的远程对等点交换媒体，并释放 offer 中未被使用的任何资源。

##### 3.8.1. 顺序 Forking

顺序 forking 涉及一个调用被分派到多个远程被调用方，其中每个被调用方可以接受调用，但每次只有一个活动会话存在；不执行媒体合流操作。

JSEP 可以很好地处理顺序 forking ，允许应用程序轻松地控制选择所需远程端点的策略。当来自其中一个调用者的 answer 到达时，应用程序可以选择将其应用为 provisional answer 或者 final answer。

在 "first-one-wins" 的情况下，第一个 answer 将被应用为 final answer，并且应用程序将拒绝任何随后的 answer。在 SIP 中可以理解为 "ACK + BYE"。

在 "last-one-wins" 的情况下，所有的 answer 将作为 provisional answer，任何先前的呼叫阶段将被终止。在某些时候，应用程序将结束建立过程，可能会使用一个计时器；此时，应用程序可以重新应用挂起的远程描述作为 final answer。

##### 3.8.2. 并行 Forking

并行 forking 包括一个调用被分派到多个远程被调用方，其中每个被调用方可以接受该调用从而可以建立多个同时活动的信令会话。如果多个被调用方同时发送媒体，处理的可能性在[RFC3960] 3.1节中描述。如今的大多数 SIP 设备一次只支持与单个设备交换媒体，而不是像早期那样尝试混合多个媒体音频源，因为这可能会导致混乱的情况。例如，考虑将欧洲的回铃音和北美的回铃音混合在一起——产生的声音将不像这两种音调，会让用户感到困惑。如果信令应用程序希望一次只与一个远程端点交换媒体，那么从媒体引擎的角度来看，这与顺序 forking 的情况完全相同。

在并行 forking 的情况下，JavaScript 应用程序希望同时与多个对等体交换媒体，流程稍微复杂一些，但JavaScript应用程序可以遵循 [RFC3960] 描述的策略，使用 UPDATE。UPDATE 方法允许信令为它希望与之交换媒体的每个对等体建立单独的媒体流。在 JSEP 中，UPDATE 使用的这个 offer 将通过简单地创建一个新的 PeerConnection (参见 4.1 节)来形成，并确保相同的本地媒体流已经被添加到这个新的 PeerConnection 中。然后新的 PeerConnection 对象将产生一个 SDP offer，该 offer 可以被信令用来执行 [RFC3960] 中讨论的 UPDATE 策略。

由于是共享媒体流，应用程序将以 N 个并行 PeerConnection 会话结束，每个会话都有一个本地和远程描述以及它们自己的本地和远程地址。来自这些会话的媒体流可以使用setDirection 来管理(参见 4.2.3 节)，或者应用程序可以选择从所有会话混合播放媒体。当然，如果应用程序只想保留单个会话，它可以简单地终止不再需要的会话。

## 4. 接口

本节详细介绍了实现 JSEP 功能必须具备的基本操作。W3C API 中公开的实际 API 可能有一些不同的语法，但应该很容易映射到这些概念。

### 4.1. PeerConnection

#### 4.1.1. 构造函数

PeerConnection 构造函数允许应用程序为媒体会话指定全局参数，例如在收集候选对象时使用的 STUN/TURN 服务器和凭证，以及初始 ICE 候选策略和池大小，以及使用的 bundle 策略。

如果指定了 ICE 候选策略，它的功能如 3.5.3 节所述，导致 JSEP 实现只向应用程序显示被允许的候选策略(包括任何实现内部过滤)，并且只使用这些候选策略进行连接检查。可用的策略集合如下:

- all:
所有被执行政策允许的候选人将被收集和使用。
- relay:
除中继候选外，所有其他候选将被过滤掉。这将隐藏远程 peer 可能从接收到的候选对象确定的位置信息。根据应用程序部署和选择中继服务器的方式，这可能会将位置混淆到城市级甚至全球级。
默认的 ICE 候选策略必须设置为 "all"，因为这通常是理想的策略，而且通常也会显著减少应用程序对 TURN 服务器资源的使用。

如果指定了 ICE 候选池的大小，则表示预收集候选的 ICE 组件的数量。因为预收集的结果是在很长一段时间内使用 STUN/TURN 服务器资源，这必须只发生在应用程序请求时，因此默认的候选池大小必须为零。

应用程序可以指定使用 bundle 的首选策略，bundle 的多路复用机制定义在 [RFC8843] 中。无论策略如何，应用程序将总是尝试协商捆绑到一个单一的传输，并将提供一个单一 bundle 组的所有 m-section；这个单一传输的使用取决于接受 bundle 的应答器。但是，通过从下面的列表中指定策略，应用程序可以精确地控制它将媒体流捆绑在一起的积极程度，这将影响它与 "non-bundle-aware" 端点的互操作方式。当与 "non-bundle-aware" 终端协商时，只有未标记为 "bundle-only" 的流将被建立。

可用的策略集合如下:

- balanced:
每个类型(音频、视频或应用程序)的第一个 m-section 将包含传输参数，这将允许应答者拆分该部分。每个类型的第二个和后续的 m-section 将被标记为 "bundle-only"。结果是，如果有N种不同的媒体类型，那么将为 N 个媒体流收集候选媒体。这一政策平衡了实现多元化的期望和确保基本音频和视频仍然可以在遗留方案中协商的需要。当作为应答者时，如果 offer 中没有bundle 组，实现将拒绝所有类型的 m-section 。
- max-compat:
所有 m-section 将包含传输参数但不标记为 "bundle-only"。这个策略将允许所有的流都被 "non-bundle-aware" 的端点接收，但是需要为每个媒体流收集单独的候选。
- max-bundle:
只有第一个 m-section 将包含传输参数，除第一个流之外的所有流将被标记为 "bundle-only"。该策略旨在最小化候选集合和最大化复用，以降低与历史端点的兼容性为代价。当作为应答者时，实现将拒绝除第一个 m-section 区段之外的任何 m-section，除非它们与 m-section 在同一个 bundle 组中。

由于提供了性能和与遗留端点兼容性之间的最佳权衡，默认 bundle 策略必须设置为 "balanced"。

应用程序可以指定使用 RTP/RTCP 传输复用 [RFC5761] 的首选策略，使用以下策略之一:

- negotiate:
JSEP实现将收集 RTP 和 RTCP 候选，但也将提供 "a=rtcp-mux"，从而允许兼容多路或非多路复用终端。
- require:
JSEP 实现将只收集 RTP 候选，并将在它生成的 offer 中任何新的 m-section 中插入 "a=rtcp-mux-only" 。这样一来，候选收集者需要收集的候选数量就减少了一半。使用不包含"a=rtcp-mux" 属性的 m-section 描述将导致返回错误。

默认的复用策略必须设置为 "require"。实现可以选择拒绝应用程序设置多路复用策略为 "negotiate" 的尝试。

#### 4.1.2. addTrack

addTrack 方法为 PeerConnection 添加一个 MediaStreamTrack，使用 MediaStream 参数将该 track 与同一 MediaStream 中的其他 track 关联起来，这样当创建一个 offer 或 answer 时，它们可以被添加到相同的 "LS"(Lip Synchronization)组。将 track 添加到相同的 "LS" 组表明，这些 track 的播放应该同步以进行正确的lip sync，如[RFC5888]，第 7 节所述。addTrack 试图最小化收发器的数量，如下所示：如果 PeerConnection 处于 "have-remote-offer" 状态，该 track 将被附加到第一个兼容的 transceiver 上，该 transceiver 是由最近的 setRemoteDescription 调用创建的，并且没有本地 track。否则，将创建一个新的 transceiver，如 4.1.4 节所述。

#### 4.1.3. removeTrack

removeTrack 方法从 PeerConnection 中删除 MediaStreamTrack，使用 RtpSender 参数来指示哪个 sender 的 track 应该被删除。清除 sender 的 track 后，sender 停止发送。调用 createOffer 后，如果是 transceiver，方向由 sendrecv 变为 recvonly，或者由 sendonly 变为 inactive 。

#### 4.1.4. addTransceiver

addTransceiver 方法增加一个新的 RtpTransceiver 到 PeerConnection。如果提供了 MediaStreamTrack 参数，则 transceiver 将被配置为该媒体类型，并且 track 将被附加到 transceiver 上。否则，应用程序必须显式地指定类型；这种模式对于创建 recvonly transceiver 非常有用，对于创建可以在以后附加 track 的 transceiver 也非常有用。

应用程序可以在创建的时候指定 transceiver 方向属性、关联的 MediaStreams (允许 "LS" 组分配)以及一组媒体编码(用于在 3.7 节中描述的 simulcast)。

#### 4.1.5. onaddtrack 事件

当 setRemoteDescription 调用后生成了一个新的 remote track 时，onaddtrack 事件被通知给应用程序。新 track 在事件中被作为 MediaStreamTrack 对象，与该 track 所在的 MediaStream 一起包含在事件参数中。

#### 4.1.6. createDataChannel

createDatachannel 方法创建一个新的数据通道并将其附加到 PeerConnection。如果当前 PeerConnection 没有数据通道，则需要一次新的 offer/answer 交换。在一个给定的PeerConnection 上的所有数据通道共享相同的 SCTP/DTLS 关联("SCTP" 代表 "Stream Control Transmission Protocol")，因此相同的 m-section，后续的数据通道的创建不会对 JSEP 状态产生任何影响。

createdatchannel 方法也包含了 PeerConnection 使用的一些参数(例如 maxPacketLifetime)，但不会在 SDP 中反映，也不会影响 JSEP 状态。

#### 4.1.7. ondatachannel 事件

当远端协商了一个新的数据通道时，ondatchannel 事件被通知给应用程序，这可以在底层 SCTP/DTLS 关联建立之后的任何时间发生。在事件参数中包含新的数据通道对象。

#### 4.1.8. createOffer

createOffer 方法生成一个 描述其支持的会话配置的 SDP [RFC3264]，包括添加到当前 PeerConnection 媒体的描述 、codec、RTP/RTCP、由 ICE 代理收集的 ICE candidate。可以提供一个选项参数来对生成的 offer 提供额外的控制。这个选项参数允许应用程序触发 ICE 重启，以重新建立连接。

在最初的 offer 中，生成的 SDP 将包含会话所需的所有功能(默认支持但不希望使用的功能可以省略)；对于 SDP 的每一行，SDP 的生成规则都将遵循已定义的 SDP 规则。初始 offer 生成的具体处理详见下文 5.2.1 节。

如果在会话建立后调用 createOffer，createOffer 将根据会话的任何更改生成一个修改当前会话的 offer，例如，添加或停止 RtpTransceivers，或请求 ICE 重启。对于每个现有的流，每个 SDP line 的生成规则必须遵循已有的 RFC 定义。对于每个新媒体流，SDP 的生成必须遵循生成初始 offer 的过程。如果没有更改，或SDP line 不受请求更改的影响，offer 将只包含由最后的 offer/answer 交换协商的参数。后续 offer 生成的具体处理详见下文 5.2.2 节。

由 createOffer 生成的会话描述必须立即被 setLocalDescription 使用；如果一个系统有有限的资源(例如，解码器的数量有限)，createOffer 应该返回一个反映系统当前状态的 offer，这样 setLocalDescription 在尝试获取这些资源时就会成功。

调用这个方法可以做一些事情，比如生成新的 ICE 凭证，但它不会改变 PeerConnection 状态，触发候选收集，或导致媒体流开始或停止。具体地说，在调用 setLocalDescription 之前，offer 不会被应用，也不会成为临时的本地描述。

#### 4.1.9. createAnswer

createAnswer 方法生成一个 SDP，应答在最近的 setRemoteDescription 调用中每个 [RFC3264] 所支持的会话配置兼容的参数，setRemoteDescription 必须在调用 createAnswer 之前调用。像 createOffer 一样，返回的blob 包含了对添加到 PeerConnection 的媒体的描述，为该会话协商的codec/RTP/RTCP 选项，以及 ICE 代理收集的任何候选项，并且可以提供一个选项参数来提供对生成 answer 的额外控制。

作为一个 answer，生成的SDP将包含一个特定的配置，指定媒体传输应该如何建立；对于每一条SDP line，SDP 的生成必须遵循已有的规范。answer 生成的具体处理将在下面的 5.3 节中详细介绍。

由 createAnswer 生成的会话描述必须立即被对端 setLocalDescription 使用；像 createOffer 一样，返回的描述应该反映系统的当前状态。

调用这个方法可以做一些事情，比如生成新的 ICE 凭据，但它不会改变 PeerConnection 状态，不会触发候选收集，也不会导致媒体状态改变。具体来说，answer 不会被应用，也不会成为当前端点的本地描述，直到 setLocalDescription被调用。

#### 4.1.10. SessionDescriptionType

会话描述对象(RTCSessionDescription)的类型可以是 "offer"、"pranswer"、"answer" 或 "rollback"。这些类型提供了关于应该如何解析描述参数以及应该如何更改媒体状态的信息。

"offer" 表示一个描述必须被解析为一个 offer；描述可能包括许多可能的媒体配置。当 PeerConnection 处于 "stable" 状态时，作为 offer 的描述可以被应用，或者作为先前提供但未回复的 offer 的更新被应用。

"pranswer" 表示描述必须被解析为一个 answer，而不是最终的 answer，因此不可以释放已分配资源。如果 answer 没有指定方向为 inactive ，则可能导致媒体传输开始。作为 "pranswer" 的描述可以作为对 offer 的回应，也可以作为对先前发送的 "pranswer" 的更新。

"answer" 表示一个描述必须被解析为一个 answer，offer/answer 交换将被认为是完整的，并且任何不再需要的资源(解码器，候选)应该被释放。作为 "answer" 的描述可以作为对 "answer" 的回应，也可以作为对先前发送的 "pranswer" 的更新。

provisional answer 和 final answer 之间的唯一区别是，final answer 的结果是释放因 offer 而分配的任何未使用的资源。因此，应用程序可以自行决定 answer 是临时的还是最终的，并可以根据需要更改会话描述的类型。例如，在串行 forking 场景中，应用程序可能会收到多个 final answer，每个远程端点都有一个。应用程序可以选择接受最初的 answer 作为临时 answer，只有当它收到一个满足其标准的 answer 时(例如，一个活跃的用户而不是语音邮件)，才应用这个 answer 作为最终 answer。

"rollback" 是一种特殊的会话描述类型，它指示状态机必须回滚到以前的 "stable" 状态，如 4.1.10.2 节所述。内容必须为空。

##### 4.1.10.1. Provisional Answers 使用方式

大多数应用程序不需要使用 "pranswer" 类型创建 answers。虽然 这样可以对 offer 立即响应，为了尽快建立会话运输，防止媒体裁剪发生，JSEP 优先处理应用程序创建和发送一个 "sendonly" 最终 answer (MediaStreamTrack 为空) 后立即收到 answer，这将阻止媒体由呼叫者发送，并允许媒体在被呼叫者回答后立即发送。稍后，当被调用方实际接受该调用时，应用程序可以插入真正的 MediaStreamTrack 并创建一个新的 "sendrecv" offer 来更新之前的 offer/answer对，并启动双向媒体流。虽然这也可以通过 "sendonly" pranswer 后接 "sendrecv" answer 来完成，但最初的 pranswer 保留 offer/answer 交换开放，这意味着呼叫者不能在这段时间内发送更新的 offer。

例如，考虑一个典型的 JSEP 应用程序，它希望尽可能快地设置音频和视频。当接收到一个 offer 包含了音频和视频 MediaStreamTracks，它将发送一个直接的 answer 并将这些 track 设置为 sendonly (这意味着接受者将不会发送任何媒体，因为尚未添加自己的 MediaStreamTracks，发起者也不会发送任何媒体)。然后，它将等待用户接受呼叫并获取所需的本地 track。在用户接受后，应用程序将载入它获得的 track，这时 ICE 握手和 DTLS 握手可能已经完成，可以立即开始传输。应用程序还将向远端发送一个新的 offer，指示呼叫接受，并将音频和视频设置为 "sendrecv"。7.3 节给出了一个详细的用例流程。

当然，一些应用程序可能无法执行这种双重 offer/answer 交换，特别是那些使用历史信令协议的应用程序。在这些情况下，pranswer 仍然可以为应用程序提供一种机制来进行传输预连接。

##### 4.1.10.2. Rollback

在某些情况下，可能需要 "rollback" 对 setLocalDescription 或 setRemoteDescription 调用后进行更改。考虑这样一种情况，一个呼叫正在进行，一方希望更改一些会话参数，该端生成一个更新的 offer，然后调用 setLocalDescription。但是，远端(在 setRemoteDescription 之前或之后)决定它不想接受新参数，并向提供程序发送拒绝消息。现在，提供者(可能还有回答者)需要返回到 "stable" 状态和先前的 offer/answer 描述。为了支持这一点，我们引入了 "rollback" 的概念，它丢弃对会话的任何建议更改，将状态机返回到 "stable" 状态。回滚是通过向 setLocalDescription 或 setRemoteDescription 提供带有空内容的类型为 "rollback" 的会话描述来执行的。

#### 4.1.11. setLocalDescription

setLocalDescription 方法指示 PeerConnection 应用提供的会话描述作为它的本地配置。type 字段指示该描述应被处理为 offer 、provisional answer、final answer 还是 rollback；offer 和 answer 是不同的检查，与已存在的每个SDP line 规则兼容。

这个 API 改变了本地媒体的状态；除此之外它还创建了接收和解码媒体的本地资源。为了使应用程序可以支持改变媒体格式的场景，PeerConnection 必须能够同时支持使用当前和等待本地描述(例如,支持所有已存在的编解码器格式)。这种双重处理开始于 PeerConnection 进入 "have-local-offer" 状态时，并一直持续到 setRemoteDescription 传入一个 final answer 或者 rollback。

这个API间接地控制候选的收集过程。当提供了本地描述，并且当前使用的传输数量与本地描述所需的传输数量不匹配时，PeerConnection 将根据需要创建传输，并开始为每个传输收集候选，如果可用，使用候选池中的候选。

如果(1)setRemoteDescription 被提前调用并带有一个 offer，(2)setLocalDescription被调用带有一个 answer(临时或最终)，(3)媒体方向是兼容的，(4)媒体可用来发送，这将导致媒体传输的开始。

#### 4.1.12. setRemoteDescription

setRemoteDescription 方法指示PeerConnection 应用提供的会话描述作为所需的远程配置。与 setLocalDescription 一样，描述的类型字段指示应该如何处理它。

这个 API 改变了本地媒体的状态；除此之外，它还设置了用于发送和编码媒体的本地资源。

如果(1)setLocalDescription 被提前调用并带有一个 offer，(2)setRemoteDescription 被调用并带有一个 answer (临时或最终)，(3)媒体方向是兼容的，(4)媒体可用来发送，这将导致媒体传输的开始。

#### 4.1.13. currentLocalDescription

currentLocalDescription 方法返回当前协商的本地描述，即来自最后一次成功的 offer/answer 交换的本地描述，不包括本地描述被设置之后由 ICE 代理生成的任何本地候选。

如果 offer/answer 交换尚未完成，将返回一个空对象。

#### 4.1.14. pendingLocalDescription

pendingLocalDescription 方法返回当前正在协商的本地描述的副本，即一个没有任何相应远程 answer 的本地 offer集，除了自本地描述被设置以来由 ICE 代理生成的任何本地候选对象之外。

如果 PeerConnection 的状态是 "stable" 或 "have-remote-offer"，则返回空对象。

#### 4.1.15. currentRemoteDescription

currentRemoteDescription 方法返回当前协商的远程描述的副本，即来自最后一次成功 offer/answer 交换的远程描述，以及自远程描述设置后通过 processIceMessage 提供的任何远程候选描述。

如果 offer/answer 交换尚未完成，将返回一个空对象。

#### 4.1.16. pendingRemoteDescription

pendingRemoteDescription 方法返回当前正在协商的远程描述的副本，即一个没有任何相应本地 answer 的远程 offer 集合，除了自远程描述被设置以来通过 processIceMessage 提供的任何远程候选对象之外。

如果 PeerConnection 的状态是 "stable" 或 "have-local-offer"，则返回空对象。

#### 4.1.17. canTrickleIceCandidates

cantrickleiceccandidates 属性表示远端是否支持接收 Trickle ICE 项。有三个可能的值:

- null:
没有从另一方收到 SDP，所以不知道它是否能处理 Trickle ICE。这是调用 setRemoteDescription 之前的初始值。
- true:
SDP 已经收到另一方的指示，表明它可以支持 Trickle ICE。
- false:
SDP 已经从另一方收到指示，它不能支持 Trickle ICE。
如 3.5.2 节所述，JSEP 实现总是为应用程序单独提供候选，这与 Trickle ICE 所需要的是一致的。然而，应用程序可以使用 canTrickleIceCandidates 属性来确定他们的对等端是否能够真正地进行 Trickle ICE，也就是说，发送最初的 offer 或 answer 是否安全，然后候选人被收集。因为 "true" 是唯一明确表示远程 Trickle ICE 支持的值，所以一个比较 canTrickleIceCandidates 和 "true" 的应用程序在初始 offer 时默认会尝试 Half Trickle ICE，在与 Trickle ICE 兼容的代理进行后续交互时默认会尝试 Full Trickle ICE。

#### 4.1.18. setConfiguration

setConfiguration 方法允许 PeerConnection 的全局配置(最初由构造函数参数设置)在会话期间更改。调用这个方法的效果取决于它被调用的时间，它们会因特定参数的改变而不同:

- 对 STUN/TURN 服务器的任何更改都会影响到下一个收集阶段。如果一个 ICE 收集阶段已经开始或完成，将设置 3.5.1 节中提到的 "needs-ice-restart" 标识。这将导致下一次调用 createOffer 来生成新的 ICE 凭证，目的是强制 ICE 重启并开始一个新的收集阶段，在这个阶段中将使用新的服务器。如果 ICE 候选池的大小为非零且尚未应用本地描述，则将丢弃任何现有的候选池，并从新服务器收集新的候选池。
- 对 ICE 候选策略的任何更改都会影响下一个收集阶段。如果一个 ICE 收集阶段已经开始或完成，"needs-ice-restart" 标识将被设置。无论哪种方式，对策略的更改都不会对候选池产生影响，因为在出现收集阶段之前，应用程序无法使用池中的候选池，因此仍然可以对任何池中的候选池执行任何必要的过滤。
- 应用本地描述后，不能改变 ICE 候选池的大小。如果本地描述还没有被应用，任何对 ICE 候选池大小的更改都会立即生效；如果增加，则预先收集更多的候选人;如果减少，则丢弃现在多余的候选项。
- 在 PeerConnection 建立后，bundle 和 rtcp 复用策略不能被修改。

调用此方法可能导致对ICE代理的状态进行更改。

#### 4.1.19. addIceCandidate

addiccandidate 方法通过一个 iccandidate 对象来更新 ICE agent(章节3.5.2.1)。如果该 ICE candidate 字段为非空，则该 ICE candidate 将被视为一个新的远程 ICE candidate，根据为 Trickle ICE 定义的规则，将其添加到 current/pending 状态的远程描述中。否则，根据 [RFC8838] 第 14 节的定义，iccandidate 被视为候选结束指示。

在任何一种情况下，提供的 iccandidate 中的 m-section 索引, MID，和 ufrag 字段被用来确定 iccandidate 属于哪个 m-section 和 ICE candidate，如上文 3.5.2.1 节所述。在候选结束指示的情况下，m-section 索引和 MID 字段的空值被解释为表示该指示适用于指定 ICE 候选生成中的所有 m-section 。但是，如果对于新的远程候选对象，这两个字段都为空，则必须将其视为无效条件，如下所述。

如果任何 iccandidate 字段包含无效值或在处理 iccandidate 对象时发生错误，必须忽略提供的 iccandidate 并返回一个错误。

否则，将向 ICE 代理提供新的远程候选项或候选结束项指示。对于一个新的远程候选对象，连接检查将被发送到新的候选对象，假设已经调用了 setLocalDescription 来初始化 ICE 收集过程。

#### 4.1.20. onicecandidate 事件

oniccandidate 事件在两种情况下被发送给应用程序：

1. 当ICE agent 在 ICE 收集过程中发现了一个新的允许的本地 ICE 候选，如 3.5.1 节所述，并受 3.5.3 节所讨论的限制；
2. 当 ICE 收集阶段完成时。事件包含一个单一的 iccandidate 对象，如3.5.2.1 节所定义；

在第一种情况下，新发现的候选对象反映在iccandidate对象中，并且它的所有字段必须是非空的。根据为Trickle ICE定义的规则，这个候选也将被添加到当前和/或待处理的本地描述中。

在第二种情况下，事件的 iccandidate 对象必须将其候选字段设置为 null，以表明当前收集阶段已经完成，也就是说，在这个阶段将不再有其他的候选事件。然而，iccandidate 的 ufrag 字段必须被指定，以表示哪个 ICE 候选的生成正在结束。ICE candidate 的 m-section 和 MID 字段可以指定，表示该事件适用于一个特定的 m-section，或者设置为 null，表示该事件适用于所有在收集阶段的 m-section。这个事件可以被应用程序用来生成一个候选结束的指示，定义在 [RFC8838] 第13节中。

### 4.2. RtpTransceiver

#### 4.2.1. stop

stop 方法停止 RtpTransceiver。这将导致以后调用 createOffer 为关联的 m-section 生成一个零端口。详情见下文。

#### 4.2.2. stopped

stopped 属性指示收发器是否已经停止，可以通过调用停止，也可以通过应用拒绝关联的 m-section 的 answer。在这两种情况下，它都被设置为 "true"，否则将被设置为 "false"。

一个停止的 RtpTransceiver 不发送任何 RTP 或 RTCP，也不处理任何流入的 RTP 或 RTCP，并且无法重启。

#### 4.2.3. setDirection

setDirection 方法设置 RtpTransceiver 的方向，它会在以后调用 createOffer 和 createAnswer 时影响关联的 m-section 方向属性。方向允许的值为 "recvonly"，"sendrecv"，"sendonly" 和 "inactive"，镜像了[RFC4566] 章节 6 中定义的同名方向属性。

当创建 offer 时，RtpTransceiver 方向直接反映在输出，即使是重新 offer。当创建 answer 时，RtpTransceiver 方向为与 offer 方向的交集，如下面的 5.3 节所述。

注意，当 setDirection 设置收发器的 direction 属性时(4.2.4 节)，这个属性不会立即影响收发器的 RtpSender 是否会发送或者 RtpReceiver 是否会接收。有效的方向是由 currentDirection 属性表示的，它只在应用一个 answer 时更新。

#### 4.2.4. direction

direction 属性表示传递给 setDirection 的最后一个值。如果 setDirection 从未被调用，它将被设置为 RtpTransceiver 初始化时的方向。

#### 4.2.5. currentDirection

currentDirection 属性表示 RtpTransceiver 关联的 m-section 最后协商的方向。更具体地说，它指示了最后一个应用 answer (包括临时 answer)中相关的 m-section 的方向属性 [RFC3264]，如果它是远程 answer，则 "send" 和"recv" 方向颠倒。例如，如果远程应答中关联的 m-section 的 direction 属性是 "recvonly"，则 currentDirection 被设置为 "sendonly"。

如果一个引用这个 RtpTransceiver 的 answer 还没有被应用，或者 RtpTransceiver 已经停止，currentDirection 将被设置为 "null"。

#### 4.2.6. setCodecPreferences

setCodecPreferences 方法设置 RtpTransceiver 的编解码器首选项，这反过来影响在以后调用 createOffer 和 createAnswer 时关联的 m-section 出现和编解码器的顺序。请注意，setCodecPreferences 并不直接影响实现决定发送哪个编解码器。它只影响实现表明它更喜欢通过提供或回答接收哪种编解码器。即使一个编解码器被 setCodecPreferences 排除，它仍然可以被用来发送，直到下一个 offer/answer 交换丢弃它。

RtpTransceiver 的编解码器首选项可能导致后续 createOffer 和 createAnswer 调用排除掉相关的编解码器，在这种情况下，关联的 m-section 中相应的媒体格式将被排除。编解码器首选项不能添加不存在的媒体格式。

RtpTransceiver 的编解码器首选项还可以确定 createOffer 和 createAnswer 后续调用中的编解码器顺序，在这种情况下，关联的 m-section 中的媒体格式的顺序将遵循指定的首选项。

## 5. SDP 交互过程

本节介绍创建和解析 SDP 对象时需要遵循的具体步骤。

### 5.1. 需求概述

JSEP 实现必须遵守下面列出的规范，这些规范管理 offer 和 answer 的创建和处理。

#### 5.1.1. Usage Requirements

所有由 JSEP 实现处理的会话描述，无论是本地的还是远程的，都必须表示对以下规范的支持。如果其中任何一项不存在，则必须将此遗漏视为错误。

- 必须使用 [RFC8445] 中规定的 ICE。注意，远程端点可以使用精简实现；实现必须正确处理使用 ICE-lite 的远程端点。远程端点也可以使用旧版本的ICE；实现必须正确处理 [RFC5245] 中规定的使用ICE的远程端点。
- 必须使用 DTLS [RFC6347] 或 DTLS-SRTP [RFC5763]，根据媒体类型的不同，如 [RFC8827] 所述。

SRTP 密钥的 SDP 安全描述机制 [RFC4568] 不能被使用，如在 [RFC8827] 中讨论的那样。

#### 5.1.2. Profile Names and Interoperability

对于媒体 m-section，JSEP 实现必须支持在 [RFC5764] 中指定的 "UDP/TLS/RTP/SAVPF" profile，以及在 [RFC7850] 中指定的 "TCP/DTLS/RTP/SAVPF" profile，并且必须为他们在 offer 中产生的每一个媒体 m-section 指定这些profile 之一。对于数据 m-section，实现必须支持 "UDP/DTLS/SCTP" profile 以及 "TCP/DTLS/SCTP" profile，并且必须为他们在 offer 中产生的每一个数据 m-section 指明这些 profile 中的一个。确切的 profile 是由与当前默认或选择的候选 ICE 相关的协议决定的，如 [RFC8839] 4.2.1.2 节所述。

不幸的是，为了兼容性一些端点会生成不同规则的概要文件字符串，即使它们支持这些概要文件中的一个。例如，端点可能生成 "RTP/AVP"，但提供 "a=fingerprint" 和 "a=rtcp-fb" 属性，表示它愿意支持 "UDP/TLS/RTP/SAVPF" 或 "TCP/TLS/RTP/SAVPF"。为了简化与这些端点的兼容性，JSEP 实现在处理接收到的 offer 中的媒体 m-section 时必须遵循以下规则:

- 以下 offer 中出现的 profile 需要被接受:
  - "RTP/AVP" (defined in [RFC4566], Section 8.2.2)
  - "RTP/AVPF" (defined in [RFC4585], Section 9)
  - "RTP/SAVP" (defined in [RFC3711], Section 12)
  - "RTP/SAVPF" (defined in [RFC5124], Section 6)
  - "TCP/DTLS/RTP/SAVP" (defined in [RFC7850], Section 3.4)
  - "TCP/DTLS/RTP/SAVPF" (defined in [RFC7850], Section 3.5)
  - "UDP/TLS/RTP/SAVP" (defined in [RFC5764], Section 9)
  - "UDP/TLS/RTP/SAVPF" (defined in [RFC5764], Section 9)
- profile 所匹配 m-section 的 answer 必须与 offer 匹配；
- 由于 DTLS-SRTP 是必选项, 所以 SAVP or AVP 并不是必须的; 是否支持 DTLS-SRTP 也可以通过 "a=fingerprint" 属性判断。需要注意，缺少 "a=fingerprint" 属性将会引起协商失败。
- AVPF 或 AVP的使用只是控制用于 RTCP feedback 的间隔规则。如果提供了 AVPF 或 "a=rtcp-fb" 属性，假设存在 AVPF，即默认值为 "trr-int=0"。否则，假设 AVPF 是在AVP 兼容模式下使用的，并使用 "trr-int=4000" 的值。
- 在数据 m-section, 实现中必须支持 "UDP/DTLS/SCTP", "TCP/DTLS/SCTP" 和 "DTLS/SCTP" (为了向后兼容) profile.

### 5.2. Constructing an Offer

当 createOfferv被调用时，必须创建一个新的 SDP 描述，包含在 [RFC8834] 中指定的功能。这个过程的具体细节如下所述。

#### 5.2.1. Initial Offers

当 createOffer 第一次被调用时，其结果被称为初始 offer。

生成初始 offer 的第一步是生成会话级属性，如[RFC4566] 5 节所述。具体地:

- 第一个 SDP line 必须是 "v=0"，定义在[RFC4566]，章节5.1。
- 第二个 SDP line 必须是一个 "o=" 的行，定义在[RFC4566]，章节5.2。\<username> 字段的值应该是 "-"。\<session-id> 必须由64位有符号整数表示，且取值必须小于2^63-1。建议通过生成一个 64 位的随机数来构造 \<session-id>，其中最高位设置为 0，其余 63 位采用加密随机方式。\<nettype> \<addrtype> \<unicast-address> 组合的值应该设置为一个无意义的地址，例如 "IN IP4 0.0.0.0"，以防止本端 IP地址泄露到该字段中，这个问题在 [RFC8828] 中进行了讨论。正如在 [RFC4566] 中提到的，整个 "o=" 行需要是唯一的，通过为 \<session-id> 选择一个随机数就足以实现这一点。
- 第三个 SDP 行必须是 "s=" 的行，在 [RFC4566] 5.3 节中定义；为了匹配 "o=" 行，应该使用 "-" 作为会话名称，例如，"s=-"。请注意，这与 [RFC4566] 中的建议不同，后者建议使用单个空格，但由于 "o=" 和"s=" 在 JSEP 中都是无意义的，因此拥有相同的无意义值似乎更清晰。
- 会话信息("i=")，URI("u=")，电子邮件地址("e=")，电话号码("p=")，重复时间("r=")，和时区("z=")行在这个上下文中是没有用的，不应该被包括。
- 加密密钥(“k=”)行不能提供足够的安全性，绝对不能包含。
- 必须添加 "t=" 行，如 [RFC4566] 5.9 节所述; \<start-time> 和 \<stop-time> 都应该设置为 0，例如"t=0 0"。
- 必须添加 "a=ice-options" 行以及 "trickle" 和 "ice2" 选项，具体请参见 [RFC8840] 4.1.1 节和 [RFC8445] 10 节。
- 如果使用 WebRTC 标识，必须添加 "a=identity" 行，如 [RFC8827] 5 节所述。

下一步是生成 m-section，如 5.14 节中 [RFC4566] 所述。一个 m-section 是一个已被添加到 PeerConnection 中的 RtpTransceiver，排除任何停止的 RtpTransceiver；这是按照 RtpTransceivers 被添加到 PeerConnection 的顺序完成的。如果没有这样的 RtpTransceiver，则没有 m-section 生成；后面可以添加更多内容，如[RFC3264] 5节所述。

对于为 RtpTransceiver 生成的每个 m-section，在 RtpTransceiver 和生成的 m-section 索引之间建立一个映射。

每个 m-section，如果没有标记为 "bundle-only"，必须包含唯一的 ICE 证书集和唯一的 ICE 候选集。标记了 bundle-only 的 m-section 不能包含任何 ICE 证书，也不能收集任何候选。

对于 DTLS，所有的 m-section 必须使用 PeerConnection 指定的任何和所有证书；因此，它们必须都具有相同的指纹值或值 [RFC8122]，或者这些值必须是会话级属性。

每个 m-section 必须按照 [RFC4566] 5.14 节中指定的方式生成。对于 m-section 本身，必须遵守以下规则:

- 如果 m-section 被标记为 bundle-only，那么 \<port> 值必须设置为 0。否则，值将被设置为 m-section 的默认的 ICE 候选端口，但是考虑到没有可用的候选端口，必须使用默认的端口值 9 (Discard)，如 RFC8840 4.1.1 节所示。
- 为了正确地指示使用 DTLS， profile 必须设置为 "UDP/TLS/RTP/SAVPF"，如 [RFC5764] 8节所规定的。
- 如果已为相关的 RtpTransceiver 设置了编解码器首选项，则必须按相应的顺序生成媒体格式，并且必须排除编解码器首选项中不存在的任何编解码器。
- 除上述限制外，媒体格式必须包括 [RFC7874] 第3节和 [RFC7742] 第5节中规定的强制性音频/视频编解码器。

m-section 后面必须紧跟着 "c=" 行，如[RFC4566] 5.7节所述。同样，由于没有候选项可用，"c=" 行必须包含默认值 "IN IP4 0.0.0.0"，如 [RFC8840] 4.1.1 节所定义。

[RFC8859] 将 SDP 属性分类。为了 bundling 时不必要的重复，类别相同或传输属性不需要在 bundling 的 "m-section" 中重复，[RFC8843]7.1.3 节中有说明。这包括 m-section，bundling 已经协商好，仍然是需要的，以及“m=”部分标记为 bundle-only。

以下属性，不是相同或运输的类别，必须包含在每个 m-section：

- "a=mid" 行，如 [RFC5888] 4 节中所述。所有的 MID 值必须以一种不泄漏用户信息的方式生成，例如，随机或使用每个 PeerConnection 计数器，并且应该是3字节或更少，以允许它们有效地适合在[RFC8843]章节15.2中定义的RTP头扩展。注意，这并没有设置RtpTransceiver mid属性，因为这只在应用描述时才会发生。此时，生成的MID值可以被认为是“建议的”MID。
- 方向属性，与关联的 RtpTransceiver 的方向属性相同。
- 对于媒体 m-section，"a=rtpmap" 和 "a=fmtp" 行描述的每一种媒体格式，如 [RFC4566] 6 节和 [RFC3264] 5.1 节所述。
- 对于每个需要使用 RTP 重传的主编解码器，对应的 "a=rtpmap" 指示主编解码器的时钟速率的 "rtx"，对应的 "a=fmtp" 指示主编解码器的有效负载类型，如 [RFC4588] 8.1 节所述。
- 对于每个支持的前向纠错 (FEC) 机制，"a=rtpmap" 和 "a=fmtp" 行，如 [RFC4566] 6 节所述。必须支持的 FEC 机制在 [RFC8854] 7 节中指定，每种媒体类型的具体用法在第 4 节和第 5 节中概述。
- 如果这个 m-section 用于每个包的媒体持续时间可配置的媒体，例如，音频，一个 "a=maxptime" 表示可以封装在每个包中的最大媒体量，以毫秒为单位，如 [RFC4566] 6节中所规定的。该值被设置为 m-section 中包含的所有编解码器的最大持续时间值中的最小值。
- 如果这个“m=”部分用于视频媒体，并且已知可以解码的图像的大小有限制，增加 "a=imageattr" 行，如 3.6 节所述。
- 对于每个支持的 RTP 报头扩展，一个 "a=extmap" 行，如[RFC5285] 5 节中指定的。应该/必须支持的报头扩展列表在 [RFC8834] 5.2 节中指定。任何需要加密的头部扩展必须在 [RFC6904] 4 节中指定。
- 对于每个支持的 RTCP 反馈机制，一个 "a=rtcp-fb”行，如[RFC4585] 4.2 节所述。应该/必须支持的 RTCP 反馈机制列表在 [RFC8834] 5.1 节中指定。
- 如果 RtpTransceiver 是 sendrecv 或 sendonly 方向:
  - 对于通过 addTrack 或 addTransceiver 创建的每个 MediaStream，一个 "a=msid" 行，如 [RFC8830] 2 节中指定的，但省略了 "appdata" 字段。
- 如果 RtpTransceiver 是 sendrecv 或 sendonly 方向，并且应用程序已经为一个编码指定了 rid-id，或者在 RtpSenders 的参数中指定了多个编码，则为每个指定的编码指定一个 "a=rid" 行。"a=rid" 行在 [RFC8851] 中指定，它的方向必须是 "send"。如果应用程序已经选择了 rid-id，它必须被使用；否则 rid-id 必须由实现生成。rid-ids 生成不能泄露用户信息，例如随机或使用 每个 PeerConnection 计数器( [RFC8852] 3.3节)，并且应该 3 个字节或更少，让他们有效地融入RTP报头中定义扩展 [RFC8852] 3.3节。如果没有指定编码，或者只指定了一个编码但没有指定 rid-id，则不会生成 "a=rid" 行。
- 如果 RtpTransceiver 是 sendrecv 或 sendonly 方向，并且产生了一个以上的 "a=rid" 行，一个 "a=simulcast" 行，方向 "send"，定义在 [RFC8853] 5.1 节。关联的 rid-id 集合必须包含 m-section 的 "a=rid"行中使用的所有 rid-id。
- 如果这个 PeerConnection的 bundle policy 被设置为 "max-bundle" 并且这不是第一个 m-section ，或者这个 bundle policy 被设置为 "balanced" 并且这不是这个媒体类型的第一个 m-section，需要 "a=bundle-only" 行。

以下属性，它们属于 IDENTICAL 或 TRANSPORT 类别，但必须只出现在 m-section 中，这些部分要么有唯一的地址，要么与 bundle 标签相关联。(在最初的 offer 中，这意味着那些 m-section 不包含 "a=bundle-only" 属性。)

- "a=ice-ufrag" 和 "a=ice-pwd" 行，参见[RFC8839]章节5.4。
- 对于每个所需的摘要算法，每个端点的证书都有一个或多个 "a=fingerprint" 行，如 [RFC8122] 5节所述。
- "a=setup" 行，如[RFC4145] 4 节中规定的，并在[RFC5763] 5 节中说明了用于 DTLS-SRTP 场景的用法。offer 中的角色值必须为 "actpass"。
- "a=tls-id" 行，如[RFC8842] 5.2 节所述。
- 一个 "a=rtcp" 行，如[RFC3605] 2.1 节，包含默认值 "9 in IP4 0.0.0.0"，因为还没有收集候选。
- "a=rtcp-mux" 行，如 [RFC5761] 5.1.3 节所述。
- 如果 RTP/RTCP 复用策略为 "require"，则在 [RFC8858] 4节中指定 "a=rtcp-mux-only" 行。
- 一个 "a=rtcp-rsize" 行，在[RFC5506] 5 节中指定。

最后，如果创建了数据通道，则必须为数据生成 m-section。\<media> 必须设置为 "application"，必须设置为 "UDP/DTLS/SCTP" [RFC8841]。\<fmt> 的值必须设置为 "webrtc-datchannel"，如 [RFC8841] 4.2.2 节所述。

在数据 m-section，一个 "a=mid" 行必须被生成，并包括在上面描述的，连同一个 "a=sctp-port" 行引用 SCTP 端口号，如 [RFC8841] 5.1 节中定义；并且如果合适的话，一个 "a=max-message-size" 行，在[RFC8841] 6.1 节定义。

如上所述，只有当数据 m-section 部分有一个唯一的地址或与 bundle 标签相关联(例如，如果它是唯一的 m-section)时，下列 IDENTICAL 或 TRANSPORT 属性才会被包含:

- "a=ice-ufrag"
- "a=ice-pwd"
- "a=fingerprint"
- "a=setup"
- "a=tls-id"

一旦所有的 m-section 被生成，一个会话级的 "a=group" 属性必须被添加，在 [RFC5888] 中规定。这个属性必须具有 "BUNDLE" 的语义，并且必须包含每个 m-section 的 MID。这样做的结果是，JSEP 实现将所有 m-section 作为一个 bundle group 提供。然而，m-section 是否为 bundle-only，取决于 bundle 策略。

下一步是生成会话级别的 LS 同步组，定义在[RFC5888] 7 节。对于每个被多个 RtpTransceiver 引用的 MediaStream (通过将这些 MediaStream 作为参数传递给 addTrack 和 addTransceiver 方法)，必须添加一组类型为 "LS" 的 group，该 group 包含每个 RtpTransceiver 的 MID 值。

SDP 允许应该在媒体级的属性既放在会话级又放在媒体级，即使它们是相同的。这有助于开发和调试，使它更容易理解各个媒体部分，特别是当一组最初相同的属性中的一个后来被更改时。然而，实现可以选择在会话级别聚合属性，而 JSEP 实现必须准备在任意位置识别属性。

可以包含除上述规定之外的其他属性，但以下属性除外，它们与 [RFC8834] 的要求不兼容且绝对不能包含:

- "a=crypto"
- "a=key-mgmt"
- "a=ice-lite"

注意，当使用 bundle 时，任何附加的属性必须遵循 [RFC8859] 中关于这些属性如何与 bundle 交互的建议。

请注意，这些要求在某些情况下比 SDP 的要求更为严格。实现必须准备接受兼容的 SDP，即使它不符合本规范中生成 SDP 的要求。

#### 5.2.2. Subsequent Offers

当多次调用 createOffer，或者在本地描述已经设置之后调用，处理过程与初始 offer 略有不同。

如果之前的 offer 没有使用 setLocalDescription 应用，这意味着 PeerConnection 仍然处于 "stable" 状态，必须遵循生成初始 offer 的步骤，受以下限制:

- "o=" 行中的字段必须保持不变，除了 \<session-version> 字段，如果 createOffer 的输出可能与之前 createOffer 的输出不同，则 \<session-version> 字段必须在每次调用 createOffer 时递增 1；实现可以选择在每次调用时递增 \<session-version>。生成的 \<session-version> 的值与当前本地描述的无关；特别注意以下情况，如果当前版本为 N ，一个 offer 被创建并应用于版本 N+1，然后该 offer 被回滚，以便当前版本再次为 N，但是下一个生成的 offer 将为版本 N+2。

请注意，如果应用程序通过读取 currentLocalDescription 而不是 createOffer 来创建一个 offer，返回的 SDP 可能与最初调用 setLocalDescription 时不同，因为添加了收集的 ICE 候选项，但 \<session-version> 将不会改变。没有已知的情况会导致问题，但如果这是一个问题，解决方案是简单地使用 createOffer 来确保唯一的 \<session-version>。

如果前面使用 setLocalDescription 提供应用，但相应的来自远端的 answer 尚未应用，这意味着 PeerConnection 仍在 "have-local-offer" 状态，offer 的生成将遵循以下规则：

- "s=" 和 "t=" 行必须保持不变。
- 如果有任何 RtpTransceiver 被添加，并且在当前本地描述或远程描述中存在一个 "m=" 的 0 端口区域，m-section 必须被回收，为添加的 RtpTransceiver 生成一个 m-section ，就像 m-section 被添加到会话描述中(包括一个新的 MID 值)，并将其放在与 m-section 相同的索引和 0 端口。
- 如果 RtpTransceiver 被停止并且没有与 m-section 相关联，则绝对不能为它生成 m-section 。这阻止了 RtpTransceiver 的 m-section 被回收，并在之前的 offer/answer 交换中用于新的 RtpTransceiver，如上所述。
- 如果 RtpTransceiver 已经停止并与一个 m-section 相关联，并且 m-section 没有被回收，如上所述，一个 m-section 必须为它生成与端口设置为 0 和删除所有 "a=msid" 行。
- 对于没有停止的 RtpTransceiver，"a=msid" 行必须保持不变，如果它们存在于当前的描述中，无论 RtpTransceiver 的方向或 track 的变化。如果在当前描述中没有 "a=msid" 行，"a=msid" 行(s)必须根据与初始 offer 相同的规则生成。
- 每一行 "m=" 和 "c=" 必须填写端口，相关的 RTP profile，和 m-section 的默认候选地址，如[RFC8839]，4.2.1.2 节和 5.1.2 节所述。如果还没有收集 RTP 候选项，则必须仍然使用默认值，如上所述。
- 每一行 "a=mid" 必须保持一致。
- "a=ice-ufrag" 和 "a=ice-pwd" 必须保持不变，除非 ICE 配置发生了改变(例如，改变了支持的 STUN/TURN 服务器或 ICE 候选策略)或 icerstart 选项(5.2.3.1 节)被指定。如果 m-section 被捆绑到另一个 m-section 中，它仍然不能包含任何 ICE 凭证。
- 如果 m-section 没有被捆绑到另一个 m-section 中，它的 "a=rtcp" 属性行必须被填充默认 rtcp 候选的端口和地址，如 RFC5761 5.1.3 节所示。如果还没有收集 RTCP 候选项，则必须使用默认值，如 5.2.1 节所述。
- 如果 m-section 没有捆绑到另一个 m-section，对于在最近的收集阶段(参见章节3.5.1)收集的每个候选，必须添加一个 "a=candidate" 行，如 [RFC8839] 5.1 节所定义。如果该 m-section 的候选收集已经完成，则必须添加一个 "a=end-of-candidates" 属性，如 [RFC8840] 8.2 节所述。如果 m-section 被捆绑到另一个 m-section 中，那么 "a=candidate" 和 "a=end-of-candidates" 都必须被省略。
- 对于仍然存在的 RtpTransceiver，"a=rid" 行必须保持不变。
- 对于仍然存在的 RtpTransceiver，任何 "a=simulcast" 行必须保持不变。

如果之前的 offer 是使用 setLocalDescription 应用的，而来自远端相应的 answer 已经使用 setRemoteDescription 应用，这意味着 PeerConnection 是在 "have-remote-pranswer" 状态或 "stable" 状态，根据协商的会话描述，按照上面提到的 "have-local-offer" 状态的步骤生成 offer。

此外，对于新 offer 中每一个现有的、不可回收的、不可拒绝的 m-section，根据当前本地或远程描述中相应的 m-section 的内容，视情况作出以下调整:

- m-section 和相应的 "a=rtpmap" 和 "a=fmtp" 行必须只包含没有被相关 RtpTransceiver 的编解码器首选项排除的媒体格式，而且必须包括所有当前可用的格式。先前提供但不再提供的媒体格式(例如，共享硬件编解码器)可能被排除在外。
- 除非已为相关联的 RtpTransceiver 设置了编解码器首选项，否则 m-section 上的媒体格式必须按照与最近的 answer 相同的顺序生成。任何媒体格式，没有出现在最近的 answer 必须添加在所有现有的格式之后。
- RTP 头部扩展必须只包含那些在最近的 answer 中出现的。
- RTCP 反馈机制必须只包括最近的 answer 中出现的那些，除了引用新添加的媒体格式的特定于格式的机制。
- 如果最近的 answer 包含了 "a=rtcp-mux" 行，那么 "a=rtcp" 行绝对不能被添加。
- "a=rtcp-mux" 必须与最近的 answer 一致。
- 不能添加 "a=rtcp-mux-only" 行。
- "a=rtcp-rsize" 行不能添加，除非出现在最近的 anwer 中。
- 不能添加 "a=bundle-only" 行，定义在 [RFC8843] 6 节。相反，JSEP 实现必须简单地省略绑定了 m-section 的 IDENTICAL 和 TRANSPORT 类别中的参数，如 [RFC8843] 7.1.3 节所述。
- 注意，如果媒体 m-section 被捆绑到数据 m-section 中，那么某些 TRANSPORT 和 IDENTICAL 的属性可能会出现在数据 m-section 中，即使它们只适合于媒体 m-section (例如，"a=rtcp-mux")。这在初始 offer 中不可能发生，因为在初始 offer 中，JSEP 实现总是在数据 m-section 之前列出媒体 m-section，而且至少有一个媒体 m-section 将没有 "a=bundle-only" 属性。因此，在最初的 offer 中，任何 "a=bundle-only" 的 m-section 将被捆绑到前面的非 bundle-only 媒体 m-section 中。

"a=group:BUNDLE" 属性必须包含在最近的 answer 中 BUNDLE 组中指定的 MID 标识符，减去任何被标记为被拒绝的 m-section，加上任何新添加或重新启用的 m-section。换句话说，bundle 属性必须包含所有之前绑定的 m-section，只要它们还是活动状态，以及任何新的 m-section。

"a=group:LS" 属性生成与首次 offer 相同的方式，有额外的规定，任何 LS 同步组出现在最近的 answer 必须继续存在并必须包含任何以前现有 MID，只要确定 m-section 仍然存在并且没有被拒绝, group 仍然包含至少两个 MID 标识符。这确保了任何同步的 "recvonly" m-section 在新的 offer 中继续同步。

#### 5.2.3. Options Handling

createOffer 方法接受一个 RTCOfferOptions 对象作为参数。如果存在以下选项，则在生成 SDP 描述时执行特殊处理。

##### 5.2.3.1. IceRestart

如果 IceRestart 选项被指定为 true，则 offer 必须通过生成新的 ICE ufrag 和 pwd 属性来指示 ICE 重启，如 [RFC8839] 4.4.3.1.1 节所述。如果在初始 offer 中指定了这个选项，则没有效果(因为已经生成了一个新的 ICE ufrag 和 pwd )。类似地，如果 ICE 配置改变了，这个选项也没有作用，因为新的 ufrag 和 pwd 属性将自动生成。在应用程序检测到故障的情况下，此选项主要用于重新建立连接。

##### 5.2.3.2. VoiceActivityDetection

静音抑制，也称为不连续传输("DTX")，可以在没有检测到语音活动时切换到一种特殊的编码，从而减少音频使用的带宽，但代价是会引起一些失真。

如果指定 "VoiceActivityDetection" 选项为 true，必须注明提供支持沉默抑制的音频接收包括舒适噪声("CN")为每个提供音频编解码器编解码器，按照[RFC3389] 5.1节中定义，除非编解码器有自己的内部静音抑制支持。对于具有自身内部静音抑制支持的编解码器，必须为该编解码器指定适当的 fmtp 参数，以表示需要对接收的音频进行静音抑制。例如，当使用 Opus 编解码器 [RFC6716] 时，在 [RFC7587] 中指定的 "usedtx=1" 参数将被用于 offer。

如果 "VoiceActivityDetection" 选项被指定为 false，则 JSEP 实现不能使用 "CN" 编解码器。对于具有自身内部静音抑制支持的编解码器，必须指定该编解码器的适当的 fmtp 参数，以表示不希望对接收的音频进行静音抑制。例如，当使用 Opus 编解码器时，"usedtx=0" 参数将在 offer 中指定。此外，在 "VoiceActivityDetection" 选项被指定为 false 的场景下，实现绝对不能对它产生的媒体使用静音抑制，无论 "CN" 编解码器或相关的 fmtp 参数是否出现在对等体的描述中。这些规则的影响 JSEP 中的静音抑制依赖于双方的相互协议，这确保了一致的处理，无论使用哪种编解码器。

在[RFC6464] 4 节中描述的客户端到混音器音频电平报头扩展的信令中，"VoiceActivityDetection" 选项对 "vad" 值的设置没有任何影响。

### 5.3. Generating an Answer

当 createAnswer 被调用时，必须创建一个新的 SDP 描述，该描述必须与提供的远程描述以及在 [RFC8834] 中指定的要求兼容。这个过程的具体细节如下所述。

#### 5.3.1. Initial Answers

当提供远程描述后第一次调用 createAnswer 时，结果被称为初始 answer。如果没有安装远程描述，则无法生成 answer，并且必须返回一个错误。

请注意，远程描述 SDP 可能不是由 JSEP 端点创建的，可能不符合 5.2 节中列出的所有要求。在许多情况下，这不是问题。然而，如果任何强制性的 SDP 属性缺失，或者上面列出的强制性使用的功能不存在，这必须被视为一个错误，并且必须导致受影响的 m-section 被标记为拒绝。

生成初始 answer 的第一步是生成会话级属性。这里的过程是相同的，表明在上面 5.2.1 节中，除了 "a=ice-options" 和 "trickle" 选项，在 [RFC8840] 4.1.3 节中被定义, 和 "ice2" 选项在 [RFC8445] 10 节中被定义，包括这样一个选项如果出现在 offer 中。

下一步是生成会话级的 LS group，如 [RFC5888] 7 节中定义的那样。对于 offer 中出现的每一组类型为 "LS" 的 group，选择由指定组中的 MID 值引用的本地 RtpTransceiver，并确定其中哪个引用了一个公共的本地 MediaStream(在 addTrack/addTransceiver 调用中指定用于创建它们)或没有 MediaStream 引用，因为它们不是由 addTrack/addTransceiver 创建的。如果存在至少两个这样的 RtpTransceiver，必须添加一组类型为 "LS" 的 RtpTransceiver 的 MID 值。否则，必须忽略提供的 LS group，answer 中不生成相应的组。

作为一个简单的例子，考虑一下下面提供的同一个 MediaStream 中包含的单个音频和单个视频 track。为了清晰起见，与本例无关的 SDP line 已被删除。正如 5.2 节中所解释的，添加了一组类型为 "LS" 的引用每个 track 的 RtpTransceiver。

```sdp
          a=group:LS a1 v1
          m=audio 10000 UDP/TLS/RTP/SAVPF 0
          a=mid:a1
          a=msid:ms1
          m=video 10001 UDP/TLS/RTP/SAVPF 96
          a=mid:v1
          a=msid:ms1
```

如果 answer 使用一个单一的 MediaStream，当它添加 track，它的两个 RtpTransceiver 将引用这个流，所以随后的 answer 将包含一个 "LS" 组相同的 answer，如下图所示:

```sdp
          a=group:LS a1 v1
          m=audio 20000 UDP/TLS/RTP/SAVPF 0
          a=mid:a1
          a=msid:ms2
          m=video 20001 UDP/TLS/RTP/SAVPF 96
          a=mid:v1
          a=msid:ms2
```

然而，如果 answer 将其 track 分组到单独的 MediaStreams 中，它的 RtpTransceiver 将引用不同的流，因此随后的回答将不包含 "LS" group。

```sdp
          m=audio 20000 UDP/TLS/RTP/SAVPF 0
          a=mid:a1
          a=msid:ms2a
          m=video 20001 UDP/TLS/RTP/SAVPF 96
          a=mid:v1
          a=msid:ms2b
```

最后，如果 answer 没有添加任何 track，它的 RtpTransceiver 将不会引用任何 MediaStream，为了维护 offer 的首选项，随后的 answe 将包含相同的 "LS" group。

```sdp
          a=group:LS a1 v1
          m=audio 20000 UDP/TLS/RTP/SAVPF 0
          a=mid:a1
          a=recvonly
          m=video 20001 UDP/TLS/RTP/SAVPF 96
          a=mid:v1
          a=recvonly
```

7.2 节中的例子展示了一个更复杂的 "LS" group 生成示例。

下一步是为远程 offer 中出现的每个 m-section 生成一个 m-section，如 [RFC3264] 6 节中所述。出于讨论的目的，offer 中的任何会话级属性如果也是有效的媒体级属性则被认为出现在每个 m-section 中。每个提供的 m-section 将有一个相关的 RtpTransceiver，如 5.10 节所述。如果 RtpTransceivers 比 m-section 多，不匹配的 RtpTransceiver 将需要在后续的 offer 关联。

对于每个 offer 的 m-section，如果下列条件符合，对应的 answer 中 m-section 必须通过设置 \<port> 为 0 标记为拒绝(在 [RFC3264] 6节中定义)，进一步处理可以跳过 m-section:

- 关联的 RtpTransceiver 已经停止。
- 没有提供既支持又被编解码器首选项(如果适用的话)允许的媒体格式。
- bundle 策略是 "max-bundle"，不是第一个 m-section，也不是和第一个 m-section 在同一个 bundle group 中。
- bundle 策略是 "balanced"，不是该媒体类型的第一个 m-section，也不是和第一个 m-section 在同一个 bundle group 中。
- 这个 m-section 在一个 bundle group 中，并且 offer 中标记的 m-section 由于上述原因之一被拒绝。这要求 bundle group 中所有 m-section 被拒绝，如 [RFC8843] 7.3.3 节所述。

否则，answer 中的每个 m-section 必须按照 [RFC3264] 6.1 节中的规定生成。对于 m-section 本身，必须遵守以下规则:

- \<port> 值通常会被设置为 m-section 的默认的 ICE 候选端口，但是考虑到还没有可用的候选端口，必须使用默认的值 9 (Discard)，如 [RFC8840] 4.1.1 节所示。
- \<proto> 字段必须被设置为与在 offer 中对应的 m-section 完全匹配。
- 如果已为相关的 RtpTransceiver 设置了编解码器首选项，则媒体格式必须按相应的顺序生成，而不管提供了什么，并且必须排除编解码器首选项中没有出现的任何编解码器。
- 否则，m-section 上的媒体格式必须按照当前远程描述中提供的格式的顺序生成，不包括当前不支持的格式。当前远程描述中不存在的任何当前可用的媒体格式必须添加到所有现有格式之后。
- 在任何一种情况下，answer 中的媒体格式必须包括 offer 中包含的至少一种格式，但可以包括在本地支持但 offer 中不包含的格式，如 [RFC3264] 6.1 节所述。如果不存在通用的格式， m-section 将被拒绝。

m-section 后面必须紧跟着 "c=" 行，如 [RFC4566] 5.7 节所述。同样，由于没有候选项可用，"c=" 行必须包含默认值 "IN IP4 0.0.0.0"，如 [RFC8840] 4.1.3 节所定义。

如果 offer 支持 bundle，所有 bundle 的 m-section 必须使用相同的 ICE 证书和候选人；所有未 bundle 的 m-section 必须使用唯一的 ICE 证书和候选人。每个 m-section 必须包含以下属性(不同于 IDENTICAL 或 TRANSPORT):

- 当且仅当 offer 中出现 "a=mid" 行，如[RFC5888] 9.1 节所述。MID 值必须与 offer 中指定的值匹配。
- 方向属性，通过应用 [RFC3264] 6.1 节中提供的方向规则来确定，然后与相关的 RtpTransceiver 的方向相交。例如，在 m-section 被提供为 "sendonly" 和本地 RtpTransceiver 被设置为 "sendrecv" 的情况下，answer 的结果是 "recvonly" 方向。
- 对于 m-section，"a=rtpmap"和 "a=fmtp" 行上的每一种媒体格式，如 [RFC4566] 6 节和 [RFC3264] 6.1 节所述。
- 如果 "rtx" 存在于 offer，则需要相应的 "a=rtpmap" 指示 "rtx" 主要编解码器的时钟频率和一个 "a=fmtp" 引用声明引用的的负载类型, [RFC4588] 8.1 节定义。
- 对于每个受支持的 FEC 机制，"a=rtpmap" 和 "a=fmtp" 行，如 [RFC4566] 6 节所述。必须支持的 FEC 机制在 [RFC8854] 7 节中指定，每种媒体类型的具体用法在第 4 节和第 5 节中概述。
- 如果这个 m-section 是针对每个包的媒体持续时间可配置的媒体，例如，音频，需要增加 "a=maxptime" 行，5.2 节所述。
- 如果这个 m-section 用于视频媒体，并且已知可以解码的图像的大小有限制，需要增加 "a=imageattr" 行，如 3.6 节所述。
- 对于 offer 中每个支持的 RTP 头部扩展，需要增加 "a=extmap“ 行，如 [RFC5285] 5 节中所规定的。支持的报头扩展列表在 [RFC8834] 5.2 节中指定。任何需要加密的报头扩展在 [RFC6904] 4 节中指定。
- 对于每个支持的 RTCP 反馈机制，在 offer 中，"a=rtcp-fb" 行，如[RFC4585] 4.2 节所述。支持的 RTCP 反馈机制列表在 [RFC8834] 5.1 节中指定。
- 如果 RtpTransceiver 有一个 sendrecv 或 sendonly 方向:
  - 对于通过 addTrack 或 addTransceiver 创建的每个 MediaStream，一个 "a=msid" 行，如在 [RFC8830] 2 节中指定的，但省略了 "appdata" 字段。

如果一个 m-section 没有捆绑到另一个 m-section 必须包含以下属性( IDENTICAL 或 TRANSPORT 类别):

- "a=ice-ufrag" 和 "a=ice-pwd" 行，参见[RFC8839] 5.4 节。
- 对于每个所需的摘要算法，每个端点的证书都有一个或多个 "a=fingerprint" 行，如 [RFC8122] 5 节所述。
- "a=setup" 行，如 [RFC4145] 4 节中规定的，并在 [RFC5763] 5 节中澄清了用于 DTLS-SRTP 场景的用法。answer 中的角色值必须是 "active" 或 "passive"。当 offer 包含 "actpass" 值时，就像 JSEP 端点的情况一样，ansswer 应该使用 "active" 角色。来自非 JSEP 端点的 offer 可以发送 "a=setup" 的其他值，在这种情况下，answer 必须使用与 offer 中的值一致的值。
- "a=tls-id" 行，在[RFC8842] 5.3 节中指定。
- 如果在 offer 中出现 "a=rtcp-mux" 行，如 [RFC5761] 5.1.3 节所述。否则，"a=rtcp" 行，如[RFC3605] 2.1 节中指定的，包含默认值 "9 in IP4 0.0.0.0" (因为还没有收集候选项)。
- 如果在 offer 中出现 "a=rtcp-rsize" 行，如[RFC5506]章节5所述。

如果提供了一个数据通道，则必须为数据 m-section。\<media> 字段必须设置为 "application"，\<fmt> 字段必须设置为与 offer 中的字段完全匹配。

在数据 m-section，一个 "a=mid" 行必须被生成，并包括在上面描述的，连同一个 "a=sctp-port" 行引用 SCTP 端口号，在 [RFC8841] 5.1 节中定义；并且可以增加 "a=max-message-size" 行，在 [RFC8841] 6.1 节中定义。

如上所述，只有当数据 m-section 没有捆绑到另一个 m-section 时，才会包含 IDENTICAL 或 TRANSPORT 类别的以下属性:

- "a=ice-ufrag"
- "a=ice-pwd"
- "a=fingerprint"
- "a=setup"
- "a=tls-id"

注意，如果媒体 m-section 被捆绑到数据 m-section 中，那么某些 TRANSPORT 和 IDENTICAL 属性也可能出现在数据 m-section 中，即使它们只适合于媒体 m-section (例如，"a=rtcp-mux")。

如果提供了具有 "BUNDLE" 语义的 "a=group" 属性，则必须按照 [RFC5888] 的规定添加相应的会话级 "a=group" 属性。这些属性必须具有 "BUNDLE" 语义，并且必须包含所有来自所提供的 BUNDLE 组中没有被拒绝的中间标识符。请注意，不管 offer 中是否存在 "a=bundle-only"，answer 中所有的 m-section 都不能有 "a=bundle-only" 行。

如果在会话级别显式地定义为有效，那么所有 m-section 之间的公共属性可以移动到会话级别。

在创建 offer 时被禁止的属性也在创建 answer 时被禁止。

#### 5.3.2. Subsequent Answers

当 createAnswer 在第二次(或更晚)时被调用，或者在已经安装了本地描述之后被调用时，处理过程与初始 answer 略有不同。

如果之前的 answer 没有使用 setLocalDescription 应用，这意味着 PeerConnection 仍然处于 "have-remote-offer" 状态，则必须遵循生成初始 answer 的步骤，受以下限制:

- "o=" 行的字段必须保持不变，除了 \<session-version> 字段，如果会话描述以任何方式改变以前生成的 answer，该字段必须递增。

如果之前有任何会话描述被提供给 setLocalDescription，那么 answer 将会按照上面的 "have-remote-offer" 状态的步骤生成，还有这些例外情况:

- "s=" 和 "t=" 行必须保持不变。
- 每一行 "m=" 和 "c=" 必须填写 m-section 的默认候选端口和地址，如[RFC8839] 4.2.1.2 节所述。注意，在某些情况下，m-section 协议可能与默认候选协议不匹配，因为 m-section 协议的值必须匹配 offer 中提供的内容，如上所述。
- 每一行 "a=ice-ufrag" 和 "a=iec-pwd" 必须保持不变，除非 m-section 正在重启，在这种情况下，新的 ICE 证书必须创建，如 [RFC8839] 4.4.1.1.1 节所述。如果 m-section 被捆绑到另一个 m-section中，它仍然不能包含任何 ICE 凭证。
- 每个 "a=tls-id" 行必须保持不变，除非 offer 的 "a=tls-id" 行改变了，在这种情况下，必须创建一个新的 tls-id 值，如 [RFC8842] 5.2 节所述。
- 每个 "a=setup" 行必须使用与现有的 DTLS 关联一致的 "active" 或 "passive" 角色。
- 必须使用 RTCP 多路复用，当且仅当 m-section 之前使用了 RTCP 多路复用时，插入 "a=rtcp-mux" 行。
- 如果 m-section 没有捆绑到另一个 m-section 并且 RTCP 复用未启用，"a=rtcp-mux" 属性行必须填写默认 RTCP 候选端口和地址。如果还没有收集 RTCP 候选项，则必须使用默认值，如上面 5.3.1 节所述。
- 如果 m-section 没有捆绑到另一个 m-section，对于在最近(参见章节3.5.1)收集的每个候选，必须添加一个 "a=candidate" 行，如 [RFC8839] 5.1 节中定义。如果该 section 的候选收集已经完成，则必须添加一个 "a=end-of-candidates" 属性，如 [RFC8840] 8.2 节所述。如果 m-section 被捆绑到另一个 m-section 中，那么 "a=candidate" 和 "a=end-of-candidates" 都必须被省略。
- 对于没有停止的 RtpTransceiver，"a=msid" 行必须保持不变，无论 RtpTransceiver 的方向或 track 发生了什么变化。如果当前描述中没有 "a=msid" 行，"a=msid" 行必须按照与初始 answer 相同的规则生成。

#### 5.3.3. Options Handling

createAnswer 方法以一个 RTCAnswerOptions 对象作为参数。RTCAnswerOptions 的参数集不同于 RTCOfferOptions 中支持的参数集；IceRestart 选项是非必要的，因为在所有的 m-section 当提供者选择执行ICE 重启时，ICE 凭证将自动被更改。

RTCAnswerOptions支持以下选项。

##### 5.3.3.1. VoiceActivityDetection

answer 是静音抑制的处理如 5.2.3.2 节所述，但有一个例外：如果支持静音抑制不是在 offer 中声明，VoiceActivityDetection 参数没有影响，answer 必须生成 VoiceActivityDetection 被设置为 "false"。这是在每个编解码器的基础上完成的(例如，如果提供以某种方式提供对 CN 的支持，但为 Opus 设置 "usedtx=0"，设置 VoiceActivityDetection 为 "true" 将导致一个 answer 与 CN 编解码器和 "usedtx=0")。这条规则的影响是应答者不会试图对任何不提供静音抑制的端点使用静音抑制，使得静音抑制保证双向支持，即使是非 JSEP 端点。

### 5.4. Modifying an Offer or Answer

从 createOffer 或 createAnswer 返回的 SDP 在传递给 setLocalDescription 之前不能被更改。如果需要对 SDP 进行精确控制，必须使用前面提到的 createOffer/createAnswer 选项或 RtpTransceiver API。

在调用 setLocalDescription 提供一个 offer 或 answer 后，应用程序可以在发送到远端之前修改 SDP 以减少它的能力，只要它遵循上面定义一个有效的 JSEP offer 或 answer 的规则。类似地，一个应用程序收到了来自对等端的 offer 或 answer，可能会在调用 setRemoteDescription 之前修改收到的 SDP，受同样的约束。

与往常一样，应用程序完全负责它发送给另一方的内容，所有传入的 SDP 将由 JSEP 实现在其能力的范围内处理。假设所有的 SDP 都格式正确是错误的，但是可以假设，该规范的任何实现都能够处理来自该规范的任何其他实现的未修改的 SDP，作为远端 offer 或 answer。

### 5.5. Processing a Local Description

当 SessionDescription 被提供给 setLocalDescription 时，必须执行以下步骤:

- 如果描述类型为 "rollback"，则按照 5.7 节定义的处理，跳过本节其余部分描述的处理。
- 否则，SessionDescription 的类型将根据 PeerConnection 的当前状态进行检查:
  - 当类型为 offer 时，PeerConnection的状态必须是 "stable" 或 "have-local-offer"。
  - 当类型为 pranswer 或 answer 时，PeerConnection 状态必须为 "have-remote-offer" 或 "have-local-pranswer"。
- 如果该类型对当前状态不正确，处理必须停止并返回一个错误。
- 然后检查 SessionDescription 以确保其内容与上次 createOffer/createAnswer 调用中生成的内容相同并且没有被更改，5.4 节所讨论的那样；否则，处理必须停止并返回一个错误。
- 接下来，SessionDescription 被解析成一个数据结构，如下面 5.8 节所述。
- 最后，解析的 SessionDescription 被应用，如下 5.9 节所述。

### 5.6. Processing a Remote Description

当 SessionDescription 被提供给 setRemoteDescription 时，必须执行以下步骤:

- 如果描述类型为 "rollback"，则按照 5.7 节定义的处理，跳过本节其余部分描述的处理。
- 否则，SessionDescription 的类型将根据 PeerConnection 的当前状态进行检查:
  - 如果是 offer, PeerConnection 的状态必须是 "stable" 或者 "have-remote-offer"。
  - 当类型为 pranswer 或 answer 时，PeerConnection 状态必须为 "have-local-offer" 或 "have-remote-pranswer"。
- 如果该类型对当前状态不正确，处理必须停止并返回一个错误。
- 接下来，SessionDescription 被解析成一个数据结构，如下面 5.8 节所述。如果解析由于任何原因失败，处理必须停止并返回一个错误。
- 最后，解析的 SessionDescription 被应用，如下面的 5.10 节所述。

### 5.7. Processing a Rollback

当 PeerConnection 处于除 "stable" 之外的任何状态时，可以执行 "rollback"。这意味着 offer 和 provisional answer 都可以回滚。回滚只能用于取消建议的更改；不支持从 "stable" 状态回滚到以前的 "stable" 状态。如果在 "stable" 状态下尝试回滚，处理必须停止并返回一个错误。请注意，这意味着一旦 answer 执行了 setLocalDescription 就不能回滚。

无论调用 setLocalDescription 还是 setRemoteDescription, rollback 的效果都必须相同。

为了处理回滚，JSEP 实现放弃当前的 offer/answer 事务，将信令状态设置为 "stable"，并将挂起的本地或远程描述设置为 "null"(参见 4.1.14 和 4.1.16 节)。由废弃的局部描述分配的任何资源或候选都将被丢弃;接收到的任何媒体都按照前面的本地和远程描述进行处理。

通过回滚会话描述的应用，回滚将任何与 m-section 相关联的 RtpTransceiver 分离(参见章节5.10和5.9)。这意味着以前关联的一些 RtpTransceiver 将不再与任何 m-secion 相关联；在这种情况下，RtpTransceiver mid 属性的值必须设置为 "null"，并且必须丢弃该 transceiver 和它的 m-section 索引之间的映射关系。如果 RtpTransceiver 是通过应用一个远程提供创建的，被回滚后必须停止并从PeerConnection 移除。但是，如果通过 addTrack 方法将一个 track 附加到 RtpTransceiver 上，那么这个 RtpTransceiver 就不能被移除。这使得应用程序可以调用 addTrack，然后调用setRemoteDescription 提供一个 offer，然后回滚那个 offer，再次调用 createOffer，并在生成的 offer 中为添加的 track 对应的 m-section。

### 5.8. Parsing a Session Description

会话描述对象中包含的 SDP 由一系列的文本行组成，每一行包含一个 key-value 表达式，如 [RFC4566] 5 节所述。逐行读取 SDP，并将其转换为包含反序列化信息的数据结构。但是 SDP 允许许多类型的行，并不是所有的行都与 JSEP 应用程序相关。对于每一行，实现首先要根据其定义的 ABNF 语法确保正确，检查它是否符合 [RFC4566] 和 [RFC3264] 中使用的语义，然后解析、存储或丢弃所提供的值，如下所述。

如果任何一行的格式不对，或者不能按照描述解析，解析器必须错误地停止并拒绝会话描述，即使该值将被丢弃。这确保了实现不会意外地误解存在二义性的 SDP。

#### 5.8.1. Session-Level Parsing

首先，检查和解析会话级别的行。这些行必须以特定的顺序和特定的语法出现，如 [RFC4566] 5 节中定义的那样。请注意，虽然特定的行类型(例如，"v="，"c=")必须以定义的顺序出现，但相同类型的行(通常是 "a=" )可以以任何顺序出现。

下面的非属性行在 JSEP 上下文中没有意义，并且可能在检查后被丢弃。

- "c=" 行必须检查语法，但它的值只用于 ICE 不匹配检测，如 [RFC8445] 5.4 节中定义的那样。注意，JSEP实现永远不会遇到这种情况，因为 WebRTC 需要 ICE。
- "i="，"u="，"e="，"p="，"t="，"r="，"z=" 和 "k=" 行必须检查语法，但它们的值不被使用。

其余的非属性行按如下方式处理:

- "v=" 行必须是 0 的版本，如 [RFC4566] 5.1 节所述。
- "o=" 行必须按照 [RFC4566] 5.2 节中的规定进行解析。
- "b=" 行如果存在，必须按照 [RFC4566] 5.8 节的规定解析，并存储 bwtype 和带宽值。

最后处理属性行。特定处理必须应用于以下会话级属性("a=")行:

- 任何 "a=group" 行被解析为在 [RFC5888] 5 节中指定的，并且组的语义和 mid 被存储。
- 如果存在，一个单独的 "a=ice-lite" 行被解析为 [RFC8839] 5.3 节中指定的，并且一个值表示存在 ice-lite 被存储。
- 如果存在，一个单独的 "a=ice-ufrag" 行被解析为 [RFC8839] 5.4 节中指定的，并且 ufrag 值被存储。
- 如果存在，一个单独的 "a=ice-pwd" 行被解析为[RFC8839] 5.4 节中指定的，并且密码值被存储。
- 如果存在，"a=ice-options" 一行将被解析为[RFC8839] 5.6节中指定的，并且指定的选项集将被存储。
- 任何 "a=fingerprint" 行按照[RFC8122] 5 节中的规定进行解析，并存储指纹和算法值集。
- 如果存在，"a=setup" 一行将按照[RFC4145] 4 节中的规定进行解析，并存储 setup 值。
- 如果存在，"a=tls-id" 一行将按照[RFC8842] 5 节中的规定进行解析，并存储属性值。
- 任何 "a=identity" 行都会被解析并存储标识值以供后续验证，如 [RFC8827] 5 节所述。
- 任何 "a=extmap" 行都按照 [RFC5285] 5 节中指定的方式进行解析，并存储它们的值。

其他与 JSEP 无关的属性也可能出现，实现应该处理它们所识别的任何属性。根据 [RFC4566] 5.13 节的要求，必须忽略未知的属性行。

一旦所有会话级别的行都被解析，处理就会继续进行 m-section 中的行。

#### 5.8.2. Media Section Parsing

和会话级别的行一样，媒体部分的行必须按照特定的顺序和语法出现，这些语法在 [RFC4566] 5 节中定义。

"m=" 行本身必须按照 [RFC4566] 5.14 节中描述的那样被解析，并且存储 \<media>，\<port>，\<proto> 和 \<fmt> 值。

在 "m=" 行之后，必须对以下非属性行应用特定处理:

- 与会话级别的 "c=" 行一样，"c=" 行必须根据 [RFC4566] 5.7 节解析，但它的值没有被使用。
- "b=" 行如果存在，必须按照 [RFC4566] 5.8 节的规定解析，并存储 bwtype 和带宽值。

以下属性行也必须应用特定处理:

- 如果存在一个单独的 "a=ice-ufrag" 行被解析为 [RFC8839] 5.4 节中指定的，并且 ufrag 值被存储。
- 如果存在一个单独的 "a=ice-pwd" 行被解析为[RFC8839] 5.4 节中指定的，并且密码值被存储。
- 如果存在，"a=ice-options" 行将被解析为 [RFC8839] 5.6 节中指定的，并且指定的选项集将被存储。
- 任何 "a=candidate" 属性必须按照 [RFC8839] 5.1 节中的规定进行解析，并存储它们的值。
- 任何 "a=remote-candidate" 的属性必须按照 [RFC8839] 5.2 节中的规定进行解析，但是它们的值会被忽略。
- 如果存在，单个 "a=end-of-candidates" 属性必须按照 [RFC8840] 8.1 节中指定的方式进行解析，并标记和存储它的存在或不存在。
- 任何 "a=fingerprint" 行按照 [RFC8122] 5 节中的规定进行解析，并存储指纹和算法值集。

如果 "m=" \<proto> 值表示使用 RTP，如上文 5.1.2 节所述，必须处理以下属性行:

- "m=" \<fmt> 值必须按照 [RFC4566] 5.14 节中指定的方式解析，并存储各个值。
- 任何 "a=rtpmap" 或 "a=fmtp" 行必须按照 [RFC4566] 6 节中的规定进行解析，并存储它们的值。
- 如果存在 "a=ptime" 行必须按照 [RFC4566] 6 节的描述进行解析，并存储其值。
- 如果存在 "a=maxptime" 行必须按照 [RFC4566] 6 节中的描述进行解析，并存储其值。
- 如果存在一个方向属性行(例如，"a=sendrecv")必须按照 [RFC4566] 6 节中的描述进行解析，并存储其值。
- 任何 "a=ssrc" 属性必须按照 [RFC5576] 4.1 节中的规定进行解析，并存储它们的值。
- 任何 "a=extmap" 属性必须按照 [RFC5285] 5 节中指定的解析，并存储它们的值。
- 任何 "a=rtcp-fb" 属性必须按照 [RFC4585] 4.2 节中的规定进行解析，并存储它们的值。
- 如果存在一个单独的 "a=rtcp-mux" 属性必须按照 [RFC5761] 5.1.3 节中的规定进行解析，并标记和存储它的存在或不存在。
- 如果存在一个 "a=rtcp-mux-only" 属性必须按照 [RFC8858] 3 节中指定的方式进行解析，并标记和存储它的存在或不存在。
- 如果存在一个单独的 "a=rtcp-rsize" 属性必须按照 [RFC5506] 5 节中的规定进行解析，并标记和存储它的存在或不存在。
- 如果存在，"a=rtcp" 属性必须按照 2.1 节 [RFC3605] 中的规定进行解析，但是它的值被忽略，因为在使用 ICE 时，这些信息是多余的。
- 如果存在，"a=msid" 属性必须按照 [RFC8830] 3.2 节中的规定进行解析，并存储它们的值，忽略任何 "appdata" 字段。如果不存在 "a=msid" 属性，则为会话的 "default" MediaStream 生成一个随机的msid-id 值，如果该值还没有出现，则存储该值。
- 任何 "a=imageattr" 属性必须按照 [RFC6236] 3 节中的规定进行解析，并存储它们的值。
- 任何 "a=rid" 行必须按照 [RFC8851] 10 节中的规定解析，并存储它们的值。
- 如果存在一个 "a=simulcast" 行必须按照 [RFC8853] 中的规定进行解析，并存储其值。

否则，如果"m=" \<proto> 值表示使用 SCTP，则必须处理以下属性行:

- "m=" \<fmt> 的值必须按照 [RFC8841] 4.3 节中的规定进行解析，并存储应用协议值。
- 一个 "a=sctp-port" 属性必须存在，它必须按照 [RFC8841] 5.2 节中的规定进行解析，并存储该值。
- 如果存在一个单独的 "a=max-message-size" 属性必须按照 [RFC8841] 6 节中的规定进行解析，并存储该值。否则，使用指定的默认值。

其他与 JSEP 无关的属性也可能出现，实现应该处理它们所识别的任何属性。根据 [RFC4566] 5.13 节的要求，必须忽略未知的属性行。

#### 5.8.3. Semantics Verification

假设解析成功完成，然后对解析的描述进行评估，以确保内部一致性以及对强制特性的适当支持。具体来说，执行以下检查:

- 对于每个 m-section，必须提供 5.1.1 节中列举的每个强制使用特性的有效值。这些值可以在媒体级出现，也可以从会话级继承。
  - ICE ufrag 和 pwd 的值必须符合[RFC8839] 5.4 节中规定的大小限制。
  - tls-id 值必须根据 [RFC8842] 5节设置。如果这是一个更新 offer 或一个更新 offer 的响应，tls-id 值与当前使用的不一样，DTLS 连接不被继续，远程描述必须是 ICE 重启的一部分，以及新的 ufrag 和 pwd 值。
  - DTLS setup 值必须根据 [RFC5763] 5 节中指定的规则设置，并且必须与当前 DTLS 连接所选择的角色保持一致，如果存在并且正在继续。
- DTLS 指纹值，其中至少有一个指纹必须存在。
- 所有在 "a=simulcast" 行中引用的 rid-id 必须作为 "a=rid" 行存在。
- 每个 m-section 也会被检查，以确保不使用禁止的特性。
- 如果 RTP/RTCP 复用策略为 "require"，每个 m-section 必须包含一个 "a=rtcp-mux" 属性。如果一个 m-section 包含一个 "a=rtcp-mux-only" 属性，该 section 也必须包含一个 "a=rtcp-mux" 属性。
- 如果 m-section 出现在前面的 answer 中，RTP/RTCP 复用的状态必须匹配之前协商的。

如果该会话描述类型为 "pranswer" 或 "answer"，则应用以下附加检查:

- 会话描述必须遵循 [RFC3264] 6 节中定义的规则，包括要求 m-section 的数量必须与相关 offer 中 m-section 的数量完全匹配。
- 对于每个 m-section，媒体类型和协议值必须与相关 offer 中相应的 m-section 中的媒体类型和协议值完全匹配。

如果上述任何一个检查失败，处理必须停止并返回一个错误。

### 5.9. Applying a Local Description

以下步骤在媒体引擎级别执行以应用本地描述。如果返回错误，则必须将会话恢复到执行这些步骤之前的状态。

首先，处理 m-section。对于每个 m-section，必须执行以下步骤；如果任何参数超出范围或不能应用，处理必须停止并返回一个错误。

- 如果这个 m-section 是新的，开始收集候选，定义在[RFC8445] 5.1.1 节，除非它是明确被捆绑 ((1) 是一个 offer 和 m-section 被标记 bundle-only 或 (2) 是一个 answer , m-section 捆绑到另一个 m-section)。
- 或者，如果ICE ufrag 和 pwd 的值已经改变，触发 ICE 代理 ICE 重启，如 [RFC8445] 9 节所述，并开始收集 m-section 的新连接候选。如果这个描述是一个 answer，也开始检查那个媒体部分。
- 如果 m-section \<proto> 部分值表示使用 RTP:
  - 如果没有 RtpTransceiver 与这个 m-section 相关联，找到一个并根据以下步骤将其与这个 m-section 相关联。请注意，这种情况只会发生在 offer。
    - 找到对应于这个 m-section 的 RtpTransceiver，使用在创建 offer 时建立的 RtpTransceiver 和 m-section 索引之间的映射。
    - 设置 RtpTransceiver 的 mid 属性值为 m-section 的 mid。
  - 如果 RTCP mux 被指定，准备从 RTP ICE 组件中去除 RTP 和 RTCP，如 [RFC5761] 5.1.3 节所述。
  - 对于每个指定的 RTP 报头扩展，建立扩展 ID 和 URI 之间的映射，如 [RFC5285] 6 节所述。
  - 如果支持 MID 报头扩展，准备基于 MID 报头扩展来 demux RTP 流，如[RFC8843]，15 节所述。
  - 对于每个指定的媒体格式，建立有效负载类型和实际媒体格式之间的映射，如 [RFC3264] 6.1 节所述。另外，根据 m-section 所支持的媒体格式，准备好 demux RTP 流，如 [RFC8843] 9.2 节所述。
  - 对于每个指定的 "rtx" 媒体格式，建立 rtx 有效载荷类型和相关的主有效载荷类型之间的映射，如 [RFC4588] 8.6 和 8.7 节所述。
  - 如果 direction 属性类型为 "sendrecv" 或 "recvonly"，则启用媒体接收和解码。

最后，如果此描述的类型是 "pranswer" 或 "answer"，则遵循下面 5.11 节中定义的处理。

### 5.10. Applying a Remote Description

执行以下步骤应用远程描述。如果返回错误，则必须将会话恢复到执行这些步骤之前的状态。

如果 answer 包含任何 "a=ice-options" 属性，其中 "trickle" 被列为一个属性，更新 PeerConnection canTrickleIceCandidates 属性为 "true"。否则，将此属性设置为 "false"。

必须对会话级别的属性执行以下步骤；如果任何参数超出范围或不能应用，处理必须停止并返回一个错误。

- 对于任何指定的 "CT" 带宽值，将该值设置为所有 m-section 的最大总比特率的限制，如 [RFC4566] 5.8 节所述。在这个总体限制内，实现可以动态地决定如何在 m-section 之间最佳地分配可用带宽，单个 m-section 指定的任何特定限制。
- 对于任何指定的 "RR" 或 "RS" 带宽值，按照 [RFC3556] 2 节中的规定处理。
- 任何 "AS" 带宽值([RFC4566] 5.8 节)必须被忽略，因为这个构造在会话级别上的意义没有被很好地定义。

对于每个 m-section，必须执行以下步骤；如果任何参数超出范围或不能应用，处理必须停止并返回一个错误。

- 如果 ICE ufrag 或 pwd 从之前的远程描述中更改:
  - 如果描述类型为 "offer"，实现必须注意，需要重新启动 ICE，如[RFC8839] 4.4.1.1.1 节所述。
  - 如果描述类型为 "answer" 或 "pranswer"，则检查当前本地描述是否为 ICE 重启，如果不是，则生成一个错误。如果 PeerConnection 的状态是 "have-remote-pranswer"，并且 ICE 的 ufrag 或 pwd 由之前的临时 answer 更改，那么就通知 ICE 代理放弃之前的任何 ICE 检查表状态的 m-section。最后，向ICE代理发出信号，开始检查。
- 如果当前本地描述是 ICE 重启，但ICE ufrag 和 pwd 都没有从之前的远程描述更改(根据 [RFC8445] 9 节的规定)，则会产生错误。
- 配置与此 m-section 相关的 ICE 组件，以使用提供的 ICE 远程 ufrag 和 pwd 进行连接检查。
- 如 [RFC8445] 6.1.2 节所述，将任何提供的 ICE 候选项与任何收集到的本地候选项配对，并使用适当的凭证开始连接检查。
- 如果存在 "a=end-of-candidates" 属性，则按照 [RFC8838] 14 节中的描述处理结束候选的指示。
- 如果"m=" \<proto> 部分值表示使用 RTP:
  - 如果 m-section 被回收(5.2.2 节)，通过设置 RtpTransceiver 的 mid 属性为 "null" 来解除当前关联的 RtpTransceiver，并且丢弃该 RtpTransceiver 和它的 m-section 索引之间的映射。
  - 如果 m-section 没有与任何 RtpTransceiver 关联(可能是因为它在上一步被解除关联)，要么找到一个 RtpTransceiver，要么根据以下步骤创建一个 RtpTransceiver:
    - 如果 m-section 为 sendrecv 或 recvonly，RtpTransceiver 与 m-section 相同类型 && 通过 addTrack 添加到 PeerConnection  && 不与任何 m-section 关联 && 没有停止，符合以上条件的第一个(根据 5.2.1 节) RtpTransceiver。
    - 如果上一步没有发现 RtpTransceiver，需要创建一个 recvonly 方向的 RtpTransceiver。
    - 通过将 RtpTransceiver 的 mid 属性的值设置为 m-section 的 mid，将发现或创建的 RtpTransceiver 与 m-section 关联起来，并在 RtpTransceiver和 m-section 索引间建立一个映射。如果 m-section 不包含 MID(即，远程端点不支持 MID 扩展)，为RtpTransceiver MID 属性生成一个值，遵循 5.2.1 节中提到的 "a=mid" 的指导。
- 对于每个本地实现也支持的指定媒体格式，建立指定负载类型和媒体格式之间的映射，如 [RFC3264] 6.1 节所述。具体地说，这意味着在发送每个指定的媒体格式时，实现需要记录在传出 RTP 包中使用的有效负载类型，以及在它们的顺序中表示的每种格式的相对首选项。如果任何指定的媒体格式不被本地实现支持，则必须忽略它。
- 对于每一种指定的 "rtx" 媒体格式，建立 rtx 有效载荷类型与其关联的主有效载荷类型之间的映射，如 [RFC4588] 4 节所述。如果任何引用的主有效负载类型不存在，则必须返回错误。请注意，RTX 有效载荷类型可能引用本地媒体实现不支持的主有效载荷类型，在这种情况下，RTX 有效载荷类型也必须被忽略。
- 对于本地实现支持的每个指定的 fmtp 参数，在相关的媒体格式上启用它们。
- 对于在 m-section 中发出的每个 SSRC，准备使用该 SSRC 来 demux RTP 流，用于这个 m-section，如 [RFC8843] 9.2 节所述。
- 对于每个指定的 RTP 头部扩展，也被本地实现支持，建立扩展 ID 和 URI 之间的映射，如 [RFC5285] 5 节所述。具体地说，这意味着实现在发送每个指定的报头扩展名时记录在发送 RTP 包中使用的扩展名 ID。如果任何指定的 RTP 报头扩展不被本地实现支持，它必须被忽略。
- 对于本地实现支持的每个指定的 RTCP 反馈机制，在相关的媒体格式上启用它们。
- 对于任何指定的 "TIAS"(Transport Independent Application Specific Maximum)带宽值，将此值设置为发送媒体时使用的最大 RTP 比特率的约束，如 [RFC3890] 中所规定的。如果没有 "TIAS" 值，但指定了一个 "AS" 值，则使用以下公式生成一个 "TIAS" 值:

    ```text
    TIAS = AS * 1000 * 0.95 - (50 * 40 * 8)
    ```

  1000 将单位从 kbps 更改为 bps(根据 TIAS 的要求)，0.95 是将 5% 的带宽分配给 RTCP。然后减去报头开销的估计值，其中 50 基于每秒 50 个包，40 基于典型的报头大小(以字节为单位)，8 将字节转换为比特。请注意，"TIAS" 比 "AS" 更可取，因为它提供了更精确的带宽控制。
- 对于任何 "RR" 或 "RS" 带宽值，按照 [RFC3556] 2 节中的规定处理。
- 任何指定的 "CT" 带宽值都必须被忽略，因为这个构造在媒体层面上的意义没有被很好地定义。
- 如果 m-section 的类型是"audio":
  - 对于每个指定的 "CN" 媒体格式，为所有支持的具有相同时钟速率的媒体格式配置静音抑制，如 [RFC3389] 5 节所述，除了具有自己内部静音抑制机制的格式。这类格式(如 Opus )的静音抑制是通过 fmtp 参数来控制的，如 5.2.3.2 节所述。
  - 对于每个指定的 "telephone-event" 媒体格式，在相同的时钟速率下为所有支持的媒体格式启用双音多频(DTMF)传输，如 [RFC4733] 2.5.1.2 节所述。如果有任何受支持的媒体格式没有相应的电话事件格式，请禁用这些格式的 DTMF 传输。
  - 对于任何指定的 "ptime" 值，将可用的媒体格式配置为在发送时使用指定的包大小。如果媒体格式不支持指定的大小，则使用下一个最接近的值。

最后，如果此描述的类型是 "pranswer" 或 "answer"，则遵循下面第 5.11 节中定义的处理。

### 5.11. Applying an Answer

除了上面提到的处理本地或远程描述的步骤外，在处理类型为 "pranswer" 或 "answer" 的描述时，还将执行以下步骤。

对于每个 m-section，必须执行以下步骤:

- 如果 m-section 已经拒绝了(例如 \<port> 值设置为 0 的 answer)，停止任何接收或传输媒体的这一节中，而且除非 non-rejected 的 m-section 支持 bundle，否则丢弃任何 ICE 组件相关联，如 [RFC8839] 4.4.3.1 节所述。
- 如果远程 DTLS 指纹被更改或 "a=tls-id" 属性的值被更改，需要断开 DTLS 连接。这包括 PeerConnection 状态为 "have-remote-pranswer" 的情况。如果一个 DTLS 连接需要被断开，但是 answer 没有指示 ICE 重启，或者在 "have-remote-pranswer" 的情况下，新的 ICE 凭证，必须生成一个错误。如果在没有改变 tls-id 值或指纹的情况下执行 ICE 重启，那么相同的 DTLS 连接将在新的 ICE 通道上继续。注意，尽管 JSEP 要求当且仅当提供方改变 tls-id 值时，非 JSEP 应答者被允许改变 tls-id 值，只要 offer 包含一个 ICE 重启。因此，在接收 answer 之前处理 DTLS 数据的 JSEP 实现必须准备好接收来自以前的 DTLS 连接的 ClientHello 或数据。
- 如果不存在有效的 DTLS 连接，则在任何底层 ICE 组件处于活动状态时，准备使用指定的角色和指纹启动 DTLS 连接。
- 如果"m=" \<proto> 值表示使用RTP:
  - 如果 m-section 引用了 RTCP 反馈机制，而在 offer 中相应的 m-section 中没有出现，这表明协商存在问题，一定会导致错误。但是，answer 中允许使用新的媒体格式和新的 RTP 报头扩展值，如 [RFC3264] 7 节和 [RFC5285] 6 节所述。
  - 如果 m-section 启用了 RTCP mux，则丢弃 RTCP ICE 组件(如果存在)，并开始或继续在 RTP ICE 组件上 muxing RTCP，如 [RFC5761] 5.1.3 节所述。否则，准备通过 RTCP ICE 组件发送 RTCP；如果没有 RTCP ICE 组件存在，因为之前启用了 RTCP mux 这一定会导致错误。
  - 如果 m-section 启用了 Reduced-Size RTCP，配置这个 m-section 的 RTCP 传输使用 Reduced-Size RTCP，如 [RFC5506] 所述。
  - 如果方向属性的 answer 表明 JSEP 实现应该发送媒体("sendonly" 为本地的 answer，"recvonly" 为远端 answer，或 "sendrecv" 类型的 answer)，选择媒体格式发送最首选的媒体格式从远程描述也是本地支持，如 [RFC3264] 6.1 节和 7 节所述，并在底层传输层建立后开始使用该格式传输 RTP 媒体。如果这个输出 RTP 流还没有选择 SSRC，请选择一个唯一的随机流。如果媒体已经在传输，则应该使用相同的 SSRC，除非新的编解码器的时钟速率不同，在这种情况下必须选择一个新的 SSRC，如 [RFC7160] 4.1 节所述。
  - 远程描述中的有效载荷类型映射用于确定输出 RTP 流的有效载荷类型，包括上面选择的发送媒体格式的有效载荷类型。任何协商的 RTP 报头扩展都应该包含在输出 RTP 流中，使用来自远程描述的扩展映射。如果已经协商好了 MID 报头扩展，将其包含在输出 RTP 流中，如[RFC8843] 15 节所示。如果 RtpStreamId 或 RepairedRtpStreamId 报头扩展已经协商，并且 rid-id 已经建立，在输出 RTP 流中包括这些报头扩展，如 [RFC8851] 4 节所示。
  - 如果 m-section 的类型是音频并且静音抑制，远端和本地都支持的情况下，用静音抑制外向的媒体，按照5.2.3.2 节。如果不满足这些条件，则不能使用静音抑制对外发媒体。
  - 如果协商成功，发送相应数量的源 RTP 流，如 [RFC8853] 5.3.3 节所述。
  - 如果上面选择的发送媒体格式有相应的 "rtx" 媒体格式，或者已经协商了 FEC 机制，则为每个源 RTP 流建立一个具有唯一随机 SSRC 的冗余 RTP 流，并根据需要启动或继续发送 rtx/FEC 包。
  - 如果上面选择的发送媒体格式具有相同的时钟速率对应的 RED 媒体格式，为了弹性目的，允许使用指定格式的冗余编码，如 [RFC8854] 3.2 节所述。请注意，与 RTX 或 FEC 媒体格式不同，RED 格式是在源 RTP 流上传输的，而不是在冗余 RTP 流上。
  - 为所有使用指定媒体格式的源 RTP 流启用媒体部分中引用的 RTCP 反馈机制。具体来说，开始或继续发送请求的反馈类型，并对收到的反馈做出反应，如 [RFC4585] 4.2 节所述。当发送 RTCP 反馈时，请遵循 [RFC8108] 5.4.1 节中的规则和建议来选择使用哪个 SSRC。
  - 如果 answer 中的 direction 属性指示 JSEP 实现不应该发送媒体(本地 answer 为 "recvonly"，远程 answer 为 "sendonly"，或任何一种类型的答案为 "inactive")，停止发送所有 RTP 媒体，但继续发送 RTCP，如 [RFC3264] 5.1 节所述。
- 如果"m=" 部分 \<proto> 值表示使用 SCTP:
  - 如果存在 SCTP 关联，且远端 SCTP 端口已经改变，则丢弃现有的 SCTP 关联。这包括 PeerConnection 状态为 "have-remote-pranswer" 的情况。
  - 如果没有有效的 SCTP 关联存在，准备在关联的 ICE 组件和 DTLS 连接上发起一个 SCTP 关联，使用来自本地描述的本地 SCTP 端口值和来自远端描述的远端 SCTP 端口值，如 [RFC8841] 10.2节所述。

如果 answer 包含了有效的 bundle 组，丢弃所有将被捆绑到每个 bundle 的主 ICE 组件上的 m-section 的ICE组件，并开始相应的混合这些 m-section，如 [RFC8843] 7.4 节所述。

如果描述类型为 "answer"，并且在 ICE 候选池中仍有剩余的候选项，则丢弃它们。

## 6. 处理 RTP/RTCP

捆绑操作时，将传入的 RTP/RTCP 与适当的 m-section 关联(在 [RFC8843] 9.2 节中定义）。当不捆绑时，从接收 RTP/RTCP 的 ICE 组件中清除相应的 m-section。

一旦正确的 m-section 被确认，RTP/RTCP 被交付给与 m-section 相关的 RtpTransceiver(s)， RTP/RTCP 的进一步处理在 RtpTransceiver 级别完成。这包括使用 RID 机制 [RFC8851] 及其相关的 RtpStreamId 和 RepairedRtpStreamId 标识符来区分多个编码流，并确定哪个源 RTP 流可以被指定的冗余的 RTP 流修复。

## 7. 示例

注意，这个示例部分显示了几个 SDP 片段。为了适应 RFC 的行长限制，一些 SDP 行被分成了多行，其中前置空格表示一行是前一行的延续。此外，添加了一些空白行以提高可读性，但在 SDP 中无效。

更多关于 WebRTC 呼叫流的 SDP 例子，包括 IPv6 地址的例子，可以在 [SDP4WebRTC] 中找到。

### 7.1. Simple Example

本节展示了一个非常简单的示例，该示例在两个 JSEP 端点之间建立最小的音频/视频会话，不使用 Trickle ICE。下一节中的示例提供了一个更详细的示例，说明在 JSEP 会话中可能发生的情况。

下面的代码流显示了 Alice 的端点将会话初始化发送到 Bob 的端点。从 Alice 浏览器中的 JavaScript 应用程序到 Bob 浏览器中的 JavaScript 的消息（分别被缩写为 "AliceJS" 和 "BobJS"），被假设为通过 web 服务器的某些信令协议传输。Alice 侧和 Bob 侧的 JavaScript 在发送 offer 或 answer 之前等待所有的连接候选，所以 offer 和 answer 是完整的，不使用 Trickle ICE。Alice 和 Bob 浏览器中的用户代理(JSEP 实现)，分别缩写为 "AliceUA" 和 "BobUA"，都使用默认的 bundle 策略 "balanced" 和默认的 RTCP mux 策略 "require"。

```text
//                  set up local media state
AliceJS->AliceUA:   create new PeerConnection
AliceJS->AliceUA:   addTrack with two tracks: audio and video
AliceJS->AliceUA:   createOffer to get offer
AliceJS->AliceUA:   setLocalDescription with offer
AliceUA->AliceJS:   multiple onicecandidate events with candidates

//                  wait for ICE gathering to complete
AliceUA->AliceJS:   onicecandidate event with null candidate
AliceJS->AliceUA:   get |offer-A1| from pendingLocalDescription

//                  |offer-A1| is sent over signaling protocol to Bob
AliceJS->WebServer: signaling with |offer-A1|
WebServer->BobJS:   signaling with |offer-A1|

//                  |offer-A1| arrives at Bob
BobJS->BobUA:       create a PeerConnection
BobJS->BobUA:       setRemoteDescription with |offer-A1|
BobUA->BobJS:       ontrack events for audio and video tracks

//                  Bob accepts call
BobJS->BobUA:       addTrack with local tracks
BobJS->BobUA:       createAnswer
BobJS->BobUA:       setLocalDescription with answer
BobUA->BobJS:       multiple onicecandidate events with candidates

//                  wait for ICE gathering to complete
BobUA->BobJS:       onicecandidate event with null candidate
BobJS->BobUA:       get |answer-A1| from currentLocalDescription

//                  |answer-A1| is sent over signaling protocol
//                  to Alice
BobJS->WebServer:   signaling with |answer-A1|
WebServer->AliceJS: signaling with |answer-A1|

//                  |answer-A1| arrives at Alice
AliceJS->AliceUA:   setRemoteDescription with |answer-A1|
AliceUA->AliceJS:   ontrack events for audio and video tracks

//                  media flows
BobUA->AliceUA:     media sent from Bob to Alice
AliceUA->BobUA:     media sent from Alice to Bob
```

|offer-A1| SDP 如下:

```SDP
v=0
o=- 4962303333179871722 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 10100 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 203.0.113.100
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:47017fee-b6c1-4162-929c-a25110252400
a=ice-ufrag:ETEn
a=ice-pwd:OtSK0WpNtpUjkY4+86js7ZQl
a=fingerprint:sha-256
              19:E2:1C:3B:4B:9F:81:E6:B8:5C:F4:A5:A8:D8:73:04:
              BB:05:2F:70:9F:04:A9:0E:05:E9:26:33:E8:70:88:A2
a=setup:actpass
a=tls-id:91bbf309c0990a6bec11e38ba2933cee
a=rtcp:10101 IN IP4 203.0.113.100
a=rtcp-mux
a=rtcp-rsize
a=candidate:1 1 udp 2113929471 203.0.113.100 10100 typ host
a=candidate:1 2 udp 2113929470 203.0.113.100 10101 typ host
a=end-of-candidates

m=video 10102 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 203.0.113.100
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:47017fee-b6c1-4162-929c-a25110252400
a=ice-ufrag:BGKk
a=ice-pwd:mqyWsAjvtKwTGnvhPztQ9mIf
a=fingerprint:sha-256
              19:E2:1C:3B:4B:9F:81:E6:B8:5C:F4:A5:A8:D8:73:04:
              BB:05:2F:70:9F:04:A9:0E:05:E9:26:33:E8:70:88:A2
a=setup:actpass
a=tls-id:91bbf309c0990a6bec11e38ba2933cee
a=rtcp:10103 IN IP4 203.0.113.100
a=rtcp-mux
a=rtcp-rsize
a=candidate:1 1 udp 2113929471 203.0.113.100 10102 typ host
a=candidate:1 2 udp 2113929470 203.0.113.100 10103 typ host
a=end-of-candidates
```

|answer-A1| SDP 如下：

```sdp
v=0
o=- 6729291447651054566 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 10200 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 203.0.113.200
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:61317484-2ed4-49d7-9eb7-1414322a7aae
a=ice-ufrag:6sFv
a=ice-pwd:cOTZKZNVlO9RSGsEGM63JXT2
a=fingerprint:sha-256
              6B:8B:F0:65:5F:78:E2:51:3B:AC:6F:F3:3F:46:1B:35:
              DC:B8:5F:64:1A:24:C2:43:F0:A1:58:D0:A1:2C:19:08
a=setup:active
a=tls-id:eec3392ab83e11ceb6a0990c903fbb19
a=rtcp-mux
a=rtcp-rsize
a=candidate:1 1 udp 2113929471 203.0.113.200 10200 typ host
a=end-of-candidates

m=video 10200 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 203.0.113.200
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:61317484-2ed4-49d7-9eb7-1414322a7aae
```

### 7.2. Detailed Example

本节展示了两个 JSEP 端点之间更复杂的会话示例。Trickle ICE 是在 Full Trickle 模式下使用的，捆绑策略是 "max-bundle"，RTCP mux 策略是 "require"，并且只有一个 TURN 服务。最初，Alice 和 Bob 都建立了一个音频通道和一个数据通道。稍后 Bob 添加了两个视频流(一个用于他的视频提要，一个用于屏幕共享)，两者都支持 FEC 并将视频提要配置为 simulcast。Alice 接受这些视频流，但不添加她自己的视频流，因此它们被作为 recvonly 处理。Alice 还指定了最大视频解码器分辨率。

```text
//                  set up local media state
AliceJS->AliceUA:   create new PeerConnection
AliceJS->AliceUA:   addTrack with an audio track
AliceJS->AliceUA:   createDataChannel to get data channel
AliceJS->AliceUA:   createOffer to get |offer-B1|
AliceJS->AliceUA:   setLocalDescription with |offer-B1|

//                  |offer-B1| is sent over signaling protocol to Bob
AliceJS->WebServer: signaling with |offer-B1|
WebServer->BobJS:   signaling with |offer-B1|

//                  |offer-B1| arrives at Bob
BobJS->BobUA:       create a PeerConnection
BobJS->BobUA:       setRemoteDescription with |offer-B1|
BobUA->BobJS:       ontrack event with audio track from Alice

//                  candidates are sent to Bob
AliceUA->AliceJS:   onicecandidate (host) |offer-B1-candidate-1|
AliceJS->WebServer: signaling with |offer-B1-candidate-1|
AliceUA->AliceJS:   onicecandidate (srflx) |offer-B1-candidate-2|
AliceJS->WebServer: signaling with |offer-B1-candidate-2|
AliceUA->AliceJS:   onicecandidate (relay) |offer-B1-candidate-3|
AliceJS->WebServer: signaling with |offer-B1-candidate-3|

WebServer->BobJS:   signaling with |offer-B1-candidate-1|
BobJS->BobUA:       addIceCandidate with |offer-B1-candidate-1|
WebServer->BobJS:   signaling with |offer-B1-candidate-2|
BobJS->BobUA:       addIceCandidate with |offer-B1-candidate-2|
WebServer->BobJS:   signaling with |offer-B1-candidate-3|
BobJS->BobUA:       addIceCandidate with |offer-B1-candidate-3|

//                  Bob accepts call
BobJS->BobUA:       addTrack with local audio
BobJS->BobUA:       createDataChannel to get data channel
BobJS->BobUA:       createAnswer to get |answer-B1|
BobJS->BobUA:       setLocalDescription with |answer-B1|

//                  |answer-B1| is sent to Alice
BobJS->WebServer:   signaling with |answer-B1|
WebServer->AliceJS: signaling with |answer-B1|
AliceJS->AliceUA:   setRemoteDescription with |answer-B1|
AliceUA->AliceJS:   ontrack event with audio track from Bob

//                  candidates are sent to Alice
BobUA->BobJS:       onicecandidate (host) |answer-B1-candidate-1|
BobJS->WebServer:   signaling with |answer-B1-candidate-1|
BobUA->BobJS:       onicecandidate (srflx) |answer-B1-candidate-2|
BobJS->WebServer:   signaling with |answer-B1-candidate-2|
BobUA->BobJS:       onicecandidate (relay) |answer-B1-candidate-3|
BobJS->WebServer:   signaling with |answer-B1-candidate-3|

WebServer->AliceJS: signaling with |answer-B1-candidate-1|
AliceJS->AliceUA:   addIceCandidate with |answer-B1-candidate-1|
WebServer->AliceJS: signaling with |answer-B1-candidate-2|
AliceJS->AliceUA:   addIceCandidate with |answer-B1-candidate-2|
WebServer->AliceJS: signaling with |answer-B1-candidate-3|
AliceJS->AliceUA:   addIceCandidate with |answer-B1-candidate-3|

//                  data channel opens
BobUA->BobJS:       ondatachannel event
AliceUA->AliceJS:   ondatachannel event
BobUA->BobJS:       onopen
AliceUA->AliceJS:   onopen

//                  media is flowing between endpoints
BobUA->AliceUA:     audio+data sent from Bob to Alice
AliceUA->BobUA:     audio+data sent from Alice to Bob

//                  some time later, Bob adds two video streams
//                  note: no candidates exchanged, because of bundle
BobJS->BobUA:       addTrack with first video stream
BobJS->BobUA:       addTrack with second video stream
BobJS->BobUA:       createOffer to get |offer-B2|
BobJS->BobUA:       setLocalDescription with |offer-B2|

//                  |offer-B2| is sent to Alice
BobJS->WebServer:   signaling with |offer-B2|
WebServer->AliceJS: signaling with |offer-B2|
AliceJS->AliceUA:   setRemoteDescription with |offer-B2|
AliceUA->AliceJS:   ontrack event with first video track
AliceUA->AliceJS:   ontrack event with second video track
AliceJS->AliceUA:   createAnswer to get |answer-B2|
AliceJS->AliceUA:   setLocalDescription with |answer-B2|

//                  |answer-B2| is sent over signaling protocol
//                  to Bob
AliceJS->WebServer: signaling with |answer-B2|
WebServer->BobJS:   signaling with |answer-B2|
BobJS->BobUA:       setRemoteDescription with |answer-B2|

//                  media is flowing between endpoints
BobUA->AliceUA:     audio+video+data sent from Bob to Alice
AliceUA->BobUA:     audio+video+data sent from Alice to Bob
```

|offer-B1| SDP 如下：

```sdp
v=0
o=- 4962303333179871723 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 d1

m=audio 9 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 0.0.0.0
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:57017fee-b6c1-4162-929c-a25110252400
a=ice-ufrag:ATEn
a=ice-pwd:AtSK0WpNtpUjkY4+86js7ZQl
a=fingerprint:sha-256
              29:E2:1C:3B:4B:9F:81:E6:B8:5C:F4:A5:A8:D8:73:04:
              BB:05:2F:70:9F:04:A9:0E:05:E9:26:33:E8:70:88:A2
a=setup:actpass
a=tls-id:17f0f4ba8a5f1213faca591b58ba52a7
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize

m=application 0 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=mid:d1
a=sctp-port:5000
a=max-message-size:65536
a=bundle-only
```

|offer-B1-candidate-1| 如下:

```text
ufrag ATEn
index 0
mid   a1
attr  candidate:1 1 udp 2113929471 203.0.113.100 10100 typ host
```

|offer-B1-candidate-2| 如下:

```text
ufrag ATEn
index 0
mid   a1
attr  candidate:1 1 udp 1845494015 198.51.100.100 11100 typ srflx
                raddr 203.0.113.100 rport 10100
```

|offer-B1-candidate-3| 如下:

```text
ufrag ATEn
index 0
mid   a1
attr  candidate:1 1 udp 255 192.0.2.100 12100 typ relay
                raddr 198.51.100.100 rport 11100
```

|answer-B1| SDP 如下

```sdp
v=0
o=- 7729291447651054566 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 d1

m=audio 9 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 0.0.0.0
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:71317484-2ed4-49d7-9eb7-1414322a7aae
a=ice-ufrag:7sFv
a=ice-pwd:dOTZKZNVlO9RSGsEGM63JXT2
a=fingerprint:sha-256
              7B:8B:F0:65:5F:78:E2:51:3B:AC:6F:F3:3F:46:1B:35:
              DC:B8:5F:64:1A:24:C2:43:F0:A1:58:D0:A1:2C:19:08
a=setup:active
a=tls-id:7a25ab85b195acaf3121f5a8ab4f0f71
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize

m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=mid:d1
a=sctp-port:5000
a=max-message-size:65536
```

|answer-B1-candidate-1| 如下：

```text
ufrag 7sFv
index 0
mid   a1
attr  candidate:1 1 udp 2113929471 203.0.113.200 10200 typ host
```

|answer-B1-candidate-2| 如下：

```text
ufrag 7sFv
index 0
mid   a1
attr  candidate:1 1 udp 1845494015 198.51.100.200 11200 typ srflx
                raddr 203.0.113.200 rport 10200
```

|answer-B1-candidate-3| 如下：

```text
ufrag 7sFv
index 0
mid   a1
attr  candidate:1 1 udp 255 192.0.2.200 12200 typ relay
                raddr 198.51.100.200 rport 11200
```

|offer-B2| 的 SDP 如下所示。除了新的 m-section 的视频，两者都提供 FEC 并且其中一个为 simulcast，注意在 "o=" 行版本号的增量；更改 "c=" 行，表示选中的本地候选；以及将集合的候选包含为 a=candidate 行。

```sdp
v=0
o=- 7729291447651054566 2 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 d1 v1 v2
a=group:LS a1 v1

m=audio 12200 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 192.0.2.200
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:71317484-2ed4-49d7-9eb7-1414322a7aae
a=ice-ufrag:7sFv
a=ice-pwd:dOTZKZNVlO9RSGsEGM63JXT2
a=fingerprint:sha-256
              7B:8B:F0:65:5F:78:E2:51:3B:AC:6F:F3:3F:46:1B:35:
              DC:B8:5F:64:1A:24:C2:43:F0:A1:58:D0:A1:2C:19:08
a=setup:actpass
a=tls-id:7a25ab85b195acaf3121f5a8ab4f0f71
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=candidate:1 1 udp 2113929471 203.0.113.200 10200 typ host
a=candidate:1 1 udp 1845494015 198.51.100.200 11200 typ srflx
            raddr 203.0.113.200 rport 10200
a=candidate:1 1 udp 255 192.0.2.200 12200 typ relay
            raddr 198.51.100.200 rport 11200
a=end-of-candidates

m=application 12200 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 192.0.2.200
a=mid:d1
a=sctp-port:5000
a=max-message-size:65536

m=video 12200 UDP/TLS/RTP/SAVPF 100 101 102 103 104
c=IN IP4 192.0.2.200
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=rtpmap:104 flexfec/90000
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:71317484-2ed4-49d7-9eb7-1414322a7aae
a=rid:1 send
a=rid:2 send
a=rid:3 send
a=simulcast:send 1;2;3

m=video 12200 UDP/TLS/RTP/SAVPF 100 101 102 103 104
c=IN IP4 192.0.2.200
a=mid:v2
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=rtpmap:104 flexfec/90000
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:81317484-2ed4-49d7-9eb7-1414322a7aae
```

|answer-B2|的 SDP 如下所示。接受视频 m-section 使用 a=recvonly 来表示单向视频，使用 a=imageattr 来限制接收的分辨率，注意使用 setup:passive 来维护现有的 DTLS 角色。

```sdp
v=0
o=- 4962303333179871723 2 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 d1 v1 v2
a=group:LS a1 v1

m=audio 12100 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 192.0.2.100
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:57017fee-b6c1-4162-929c-a25110252400
a=ice-ufrag:ATEn
a=ice-pwd:AtSK0WpNtpUjkY4+86js7ZQl
a=fingerprint:sha-256
              29:E2:1C:3B:4B:9F:81:E6:B8:5C:F4:A5:A8:D8:73:04:
              BB:05:2F:70:9F:04:A9:0E:05:E9:26:33:E8:70:88:A2
a=setup:passive
a=tls-id:17f0f4ba8a5f1213faca591b58ba52a7
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=candidate:1 1 udp 2113929471 203.0.113.100 10100 typ host
a=candidate:1 1 udp 1845494015 198.51.100.100 11100 typ srflx
            raddr 203.0.113.100 rport 10100
a=candidate:1 1 udp 255 192.0.2.100 12100 typ relay
            raddr 198.51.100.100 rport 11100
a=end-of-candidates

m=application 12100 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 192.0.2.100
a=mid:d1
a=sctp-port:5000
a=max-message-size:65536

m=video 12100 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 192.0.2.100
a=mid:v1
a=recvonly
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=imageattr:100 recv [x=[48:1920],y=[48:1080],q=1.0]
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli

m=video 12100 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 192.0.2.100
a=mid:v2
a=recvonly
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=imageattr:100 recv [x=[48:1920],y=[48:1080],q=1.0]
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
```

### 7.3. Early Transport Warmup Example

这个例子演示了 4.1.10.1 节中描述的传输预热技术。在这里，Alice 的端点向 Bob 的端点发送一个 offer 以启动一个音频/视频通话。Bob 立即响应一个 answer，该 answer 接受音频/视频 m-section，但将它们标记为 send-only (从 Bob 的角度来看)，这意味着 Alice 还不能发送媒体。这允许 JSEP 实现立即开始协商 ICE 和 DTLS。Bob 的端点然后提示他接听电话，当他接听时他的端点发送第二个 offer，这使音频和视频 m-section 被激活从而实现双向媒体传输。这种流程的优点是，一旦收到第一个 answer，实现就可以继续进行 ICE 和 DTLS 协商，并建立会话传输。如果传输设置在第二个 offer 被发送之前完成，那么媒体可以被被呼叫者在应答电话时立即传输，最大限度地减少拨号后的延迟。第二个 offer/answer 交换还可以更改首选的编解码器或其他会话参数。

这个例子也使用了在 3.5.3 节中描述的 relay ICE 候选策略来最小化所需的 ICE 收集和检查。

```text
//                  set up local media state
AliceJS->AliceUA:   create new PeerConnection with "relay" ICE policy
AliceJS->AliceUA:   addTrack with two tracks: audio and video
AliceJS->AliceUA:   createOffer to get |offer-C1|
AliceJS->AliceUA:   setLocalDescription with |offer-C1|

//                  |offer-C1| is sent over signaling protocol to Bob
AliceJS->WebServer: signaling with |offer-C1|
WebServer->BobJS:   signaling with |offer-C1|

//                  |offer-C1| arrives at Bob
BobJS->BobUA:       create new PeerConnection with "relay" ICE policy
BobJS->BobUA:       setRemoteDescription with |offer-C1|
BobUA->BobJS:       ontrack events for audio and video

//                  a relay candidate is sent to Bob
AliceUA->AliceJS:   onicecandidate (relay) |offer-C1-candidate-1|
AliceJS->WebServer: signaling with |offer-C1-candidate-1|

WebServer->BobJS:   signaling with |offer-C1-candidate-1|
BobJS->BobUA:       addIceCandidate with |offer-C1-candidate-1|

//                  Bob prepares an early answer to warm up the
//                  transport
BobJS->BobUA:       addTransceiver with null audio and video tracks
BobJS->BobUA:       transceiver.setDirection(sendonly) for both
BobJS->BobUA:       createAnswer
BobJS->BobUA:       setLocalDescription with answer

//                  |answer-C1| is sent over signaling protocol
//                  to Alice
BobJS->WebServer:   signaling with |answer-C1|
WebServer->AliceJS: signaling with |answer-C1|

//                  |answer-C1| (sendonly) arrives at Alice
AliceJS->AliceUA:   setRemoteDescription with |answer-C1|
AliceUA->AliceJS:   ontrack events for audio and video

//                  a relay candidate is sent to Alice
BobUA->BobJS:       onicecandidate (relay) |answer-B1-candidate-1|
BobJS->WebServer:   signaling with |answer-B1-candidate-1|

WebServer->AliceJS: signaling with |answer-B1-candidate-1|
AliceJS->AliceUA:   addIceCandidate with |answer-B1-candidate-1|

//                  ICE and DTLS establish while call is ringing

//                  Bob accepts call, starts media, and sends
//                  new offer
BobJS->BobUA:       transceiver.setTrack with audio and video tracks
BobUA->AliceUA:     media sent from Bob to Alice
BobJS->BobUA:       transceiver.setDirection(sendrecv) for both
                    transceivers
BobJS->BobUA:       createOffer
BobJS->BobUA:       setLocalDescription with offer

//                  |offer-C2| is sent over signaling protocol
//                  to Alice
BobJS->WebServer:   signaling with |offer-C2|
WebServer->AliceJS: signaling with |offer-C2|

//                  |offer-C2| (sendrecv) arrives at Alice
AliceJS->AliceUA:   setRemoteDescription with |offer-C2|
AliceJS->AliceUA:   createAnswer
AliceJS->AliceUA:   setLocalDescription with |answer-C2|
AliceUA->BobUA:     media sent from Alice to Bob

//                  |answer-C2| is sent over signaling protocol
//                  to Bob
AliceJS->WebServer: signaling with |answer-C2|
WebServer->BobJS:   signaling with |answer-C2|
BobJS->BobUA:       setRemoteDescription with |answer-C2|
```

|offer-C1| SDP 如下：

```sdp
v=0
o=- 1070771854436052752 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 9 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 0.0.0.0
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:bbce3ba6-abfc-ac63-d00a-e15b286f8fce
a=ice-ufrag:4ZcD
a=ice-pwd:ZaaG6OG7tCn4J/lehAGz+HHD
a=fingerprint:sha-256
              C4:68:F8:77:6A:44:F1:98:6D:7C:9F:47:EB:E3:34:A4:
              0A:AA:2D:49:08:28:70:2E:1F:AE:18:7D:4E:3E:66:BF
a=setup:actpass
a=tls-id:9e5b948ade9c3d41de6617b68f769e55
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize

m=video 0 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 0.0.0.0
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:bbce3ba6-abfc-ac63-d00a-e15b286f8fce
a=bundle-only
```

|offer-C1-candidate-1| 如下：

```text
ufrag 4ZcD
index 0
mid   a1
attr  candidate:1 1 udp 255 192.0.2.100 12100 typ relay
                raddr 0.0.0.0 rport 0
```

|answer-C1| SDP 如下：

```sdp
v=0
o=- 6386516489780559513 1 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 9 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 0.0.0.0
a=mid:a1
a=sendonly
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:751f239e-4ae0-c549-aa3d-890de772998b
a=ice-ufrag:TpaA
a=ice-pwd:t2Ouhc67y8JcCaYZxUUTgKw/
a=fingerprint:sha-256
              A2:F3:A5:6D:4C:8C:1E:B2:62:10:4A:F6:70:61:C4:FC:
              3C:E0:01:D6:F3:24:80:74:DA:7C:3E:50:18:7B:CE:4D
a=setup:active
a=tls-id:55e967f86b7166ed14d3c9eda849b5e9
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize

m=video 9 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 0.0.0.0
a=mid:v1
a=sendonly
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:751f239e-4ae0-c549-aa3d-890de772998b
```

|answer-C1-candidate-1| 如下：

```text
ufrag TpaA
index 0
mid   a1
attr  candidate:1 1 udp 255 192.0.2.200 12200 typ relay
                raddr 0.0.0.0 rport 0
```

|offer-C2| SDP 如下：

```sdp
v=0
o=- 6386516489780559513 2 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 12200 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 192.0.2.200
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:751f239e-4ae0-c549-aa3d-890de772998b
a=ice-ufrag:TpaA
a=ice-pwd:t2Ouhc67y8JcCaYZxUUTgKw/
a=fingerprint:sha-256
              A2:F3:A5:6D:4C:8C:1E:B2:62:10:4A:F6:70:61:C4:FC:
              3C:E0:01:D6:F3:24:80:74:DA:7C:3E:50:18:7B:CE:4D
a=setup:actpass
a=tls-id:55e967f86b7166ed14d3c9eda849b5e9
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=candidate:1 1 udp 255 192.0.2.200 12200 typ relay
            raddr 0.0.0.0 rport 0
a=end-of-candidates

m=video 12200 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 192.0.2.200
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:751f239e-4ae0-c549-aa3d-890de772998b
```

|answer-C2| SDP 如下：

```sdp
v=0
o=- 1070771854436052752 2 IN IP4 0.0.0.0
s=-
t=0 0
a=ice-options:trickle ice2
a=group:BUNDLE a1 v1
a=group:LS a1 v1

m=audio 12100 UDP/TLS/RTP/SAVPF 96 0 8 97 98
c=IN IP4 192.0.2.100
a=mid:a1
a=sendrecv
a=rtpmap:96 opus/48000/2
a=rtpmap:0 PCMU/8000
a=rtpmap:8 PCMA/8000
a=rtpmap:97 telephone-event/8000
a=rtpmap:98 telephone-event/48000
a=fmtp:97 0-15
a=fmtp:98 0-15
a=maxptime:120
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:2 urn:ietf:params:rtp-hdrext:ssrc-audio-level
a=msid:bbce3ba6-abfc-ac63-d00a-e15b286f8fce
a=ice-ufrag:4ZcD
a=ice-pwd:ZaaG6OG7tCn4J/lehAGz+HHD
a=fingerprint:sha-256
              C4:68:F8:77:6A:44:F1:98:6D:7C:9F:47:EB:E3:34:A4:
              0A:AA:2D:49:08:28:70:2E:1F:AE:18:7D:4E:3E:66:BF
a=setup:passive
a=tls-id:9e5b948ade9c3d41de6617b68f769e55
a=rtcp-mux
a=rtcp-mux-only
a=rtcp-rsize
a=candidate:1 1 udp 255 192.0.2.100 12100 typ relay
            raddr 0.0.0.0 rport 0
a=end-of-candidates

m=video 12100 UDP/TLS/RTP/SAVPF 100 101 102 103
c=IN IP4 192.0.2.100
a=mid:v1
a=sendrecv
a=rtpmap:100 VP8/90000
a=rtpmap:101 H264/90000
a=fmtp:101 packetization-mode=1;profile-level-id=42e01f
a=rtpmap:102 rtx/90000
a=fmtp:102 apt=100
a=rtpmap:103 rtx/90000
a=fmtp:103 apt=101
a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
a=extmap:3 urn:ietf:params:rtp-hdrext:sdes:rtp-stream-id
a=rtcp-fb:100 ccm fir
a=rtcp-fb:100 nack
a=rtcp-fb:100 nack pli
a=msid:bbce3ba6-abfc-ac63-d00a-e15b286f8fce
```

## 8. Security Considerations

IETF 已经发布了单独的文档 [RFC8827] [RFC8826]，将 WebRTC 的安全架构作为一个整体来描述。本节的其余部分将介绍本文档的安全考虑事项。

虽然 JSEP 接口在形式上是一个 API，但最好将其视为一个 Internet 协议，从 JSEP 实现的角度来看，应用程序 JavaScript 是不可信任的。因此，采用 [RFC3552] 的威胁模型。特别是 JavaScript 可以以任何顺序和任何输入(包括恶意输入)调用 API。当我们考虑传递给 setLocalDescription 的 SDP 时，这尤其重要。虽然正确的 API 使用要求应用程序通过从 createOffer 或 createAnswer 派生的 SDP，但不能保证应用程序这样做。JSEP 实现必须为 JavaScript 传入虚假数据做好准备。

相反，应用程序开发者需要知道 JavaScript 不能完全控制端点行为。特别值得一提的一种情况是，将 ICE 候选从 SDP 中提取出来或写入 Trickle 候选并不具有预期的行为：实现仍然会对这些候选执行检查，即使他们没有被发送到另一边。因此无法通过删除 server-reflexive 候选来阻止远端对等体探测自己的公共 IP 地址。希望隐藏其公网 IP 地址的应用程序必须将 ICE 代理配置为只使用 relay 候选地址。

## 9. IANA Considerations

This document has no IANA actions.
