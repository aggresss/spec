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

具体的问题涉及到 m-line 被指定为 "bundle-only"，将在本 RFC 4.1.1 节讨论。目前，JSEP 和 BUNDLE 之间存在分歧，这些规范和现有的浏览器实现之间也存在分歧:

- JSEP 规定，m-line 应该使用端口 0，并在初始 offer 中添加 "a=bundle-only" 属性，而不是在 answer 或后续 offer 中；
- BUNDLE 规定，m-line 应该像前面所描述的那样标记，但是是在所有的 offer 和 answer 中。
- 当前大多数浏览器不标记任何端口为 0 的 m-line，而是为所有捆绑的 m-line 使用相同的端口；其他则遵循 JSEP 定义。

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

此外，不同的 RFC 对 offer 和 answer 的格式提出了不同的条件。例如，offer 可以提出任意数量的 m-line(即，媒体描述如 [RFC4566]，5.14 节所述)，但 answer 必须包含与要约完全相同的数字。

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

### 3.3. 会话描述格式

JSEP 的会话描述使用会话描述协议(session Description Protocol, SDP)语法进行内部表示。虽然这种格式对于 JavaScript 操作不是最佳的，但它被广泛接受并经常更新新特性；任何会话描述的替代表示都必须与 SDP 的变化保持同步，至少在这种新的表示取代 SDP 流行之前是如此。

然而，为了提供未来的灵活性，SDP 语法被封装在 SessionDescription 对象中，该对象可以从 SDP 构造并序列化到 SDP。如果未来的规范对会话描述采用 JSON 格式达成一致，我们可以很容易地让这个对象生成并使用 JSON。

如下所述，大多数应用程序应该能够将这些各种 API 调用产生和使用的 sessiondescription 视为不透明的 blob；也就是说，应用程序不需要读取或更改它们。

### 3.4. 会话描述控制

为了让应用程序控制各种公共会话参数，JSEP 提供了控制面，告诉 JSEP 实现如何生成会话描述。在大多数情况下，这避免了 JavaScript 修改会话描述的需要。

对这些对象的更改将导致对后续 createOffer/createAnswer 调用生成的会话描述的更改。

#### 3.4.1. RtpTransceivers

RtpTransceivers 允许应用程序控制与一个 m-line 相关联的 RTP media。每个 RtpTransceiver 有一个 RtpSender 和 一个RtpReceiver，应用程序可以使用它们来控制 RTP media 的发送和接收。应用程序也可以直接修改 RtpTransceiver，例如停止操作。

RtpTransceivers 通常与 m-line 是 1:1 的映射，尽管可能会有比 m-line 更多的 RtpTransceivers ，例如：当RtpTransceivers 被创建但还没有关联到 m-line，或者如果 RtpTransceivers 已经停止并从 m-line 中分离。如果 RtpTransceiver 的 mid (media identification) 属性是非空的，则 RtpTransceiver 被认为与 m-line 相关联；否则它被认为是游离的。关联的 m-line 是在创建一个 offer 或应用一个远程 offer 时使用收发器和 m-line 索引之间的映射确定的。

一个 RtpTransceiver 从来不会与一个以上的 m-line 相关联，并且一旦会话描述被应用，一个 m-line 总是与一个 RtpTransceiver 相关联。然而，在某些情况下，当一个 m-line 被拒绝时，正如下面 5.2.2 节中讨论的那样，m-line 将被“回收”，RtpTransceiver 将与一个新的 MID 值相关联。

RtpTransceivers 可以由应用程序显式创建，也可以通过调用 setRemoteDescription 来隐式创建，并提供一个新的 m-line。

#### 3.4.2. RtpSenders

RtpSenders 允许应用程序控制 RTP 媒体的发送方式。RtpSender 在概念上负责由 m-line 描述输出的 RTP 流。这包括编码附加的 MediaStreamTrack，发送 RTP 媒体包，以及关联的 RTCP。

#### 3.4.3. RtpReceivers

RtpReceivers 允许应用程序检查如何接收 RTP 媒体。RtpReceiver 在概念上负责传入的 RTP 流，由 m-line 描述。这包括处理接收到的 RTP 媒体包，解码传入的流以产生远程 MediaStreamTrack，并为传入的 RTP 流生成和处理 RTCP。

### 3.5. ICE

#### 3.5.1. ICE Gathering 概述

JSEP 根据应用程序的需要收集 ICE candidates。ICE candidates 的收集被称为收集阶段，这是由添加一个新的或回收的 m-line 到本地会话描述或描述中新的 ICE 凭证触发的，表明 ICE 重启。新 ICE 凭据的使用可以由应用程序显式触发，也可以由 JSEP 实现隐式触发，以响应 ICE 配置中的更改。

当 ICE 配置发生变化，需要一个新的收集阶段时，就会设置一个 “needs-ice-restart” 标识。设置了这个标识后，调用 createOffer API 将生成新的 ICE 凭证。这个标识通过调用setLocalDescription API 来清除，该 API 带有来自 offer 或 answer 的新 ICE 凭证，即本地或远程发起的ICE重启。

当一个新的收集阶段开始时，ICE 代理将通过状态更改事件通知应用程序正在进行收集。然后，当每个新的 ICE candidates 可用时，ICE 代理将通过一个 oniccandidate 事件通知应用程序；这些候选对象也将自动添加到当前或者挂起的本地会话描述中。最后，当收集完所有的 ICE candidates 后，将发送一个最后的 oniccandidate 事件，以表明收集过程已经完成。

注意，收集阶段只收集 new/recycled/restaring 状态的 m-line 所需的候选；其他 m-line 继续使用它们现有的候选项。同样，如果一个 m-line 被绑定(通过一个成功的bundle 协商或者被标记为 bundle-only)，那么当且仅当它的 MID 项是一个 bundle 标签时，候选的 m-line 将被收集并交换，如 [RFC8843] 所述。

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

可以通过一个 m-line 索引或者 MID 这两种方式中的一种确认 iceCandidate 对象与 m-line 是相关联的，m-line 索引是一个从零开始的索引,索引 N 指的第 N + 1 m-line 的会话描述引用的这个 IceCandidate。 MID 是一个 “media stream identification” 的值，在[RFC5888] 第4章中定义，它提供了一种更健壮的方法来标识会话描述中的 m-line，使用关联的 RtpTransceiver 对象的MID(它可能是由应答者在与不支持 MID 属性的非 JSEP 端点交互时本地生成的，如下面 5.10 节所讨论的)。如果在接收到的 iccandidate 中有 MID 字段，它必须被用于识别;否则，将使用 m-line 索引。如上所述，实现时必须考虑到接收对象缺少某些字段的情况。

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

接收器将首先结合任何已知的本地限制(例如，硬件解码器能力或本地策略)，以确定它可以接收的绝对最小和最大尺寸。如果没有已知的局部限制，"a=imageattr" 属性应该被省略。如果这些局部限制排除了接收任何视频，例如，不允许分辨率的简并情况下，"a=imageattr" 属性必须被省略，如果合适的话，m-line 必须被标记为 sendonly/inactive。

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

对于由 RtpSender 产生的每个编码，远程描述中相应的 m-line 中的 "a=imageattr recv" 属性集将被处理，以确定应该传输什么。仅考虑引用为编码选择的媒体格式的属性；每个这样的属性都是单独计算的，从 "q=" 值最高的属性开始。如果多个属性具有相同的 "q=" 值，则按照它们在包含 "m=" 部分中出现的顺序计算它们。请注意，虽然 JSEP 端点每一种媒体格式最多包含一个 "a=imageattr recv"属性，但 JSEP 端点可以从非 JSEP 端点接收包含多个此类属性的 m-line 的会话描述。

对于每个 "a=imageattr recv" 属性，应用以下规则。如果此处理成功，则相应地传输编码，并且不再考虑该编码的其他属性。否则，将按照前面提到的顺序计算下一个属性。如果所提供的所有属性都不能被成功处理，那么就不能传输编码，并且应该向应用程序抛出一个错误。

- 将该属性的限制与编码器分辨率进行比较。只考虑以下提到的具体限制；任何其他值，比如图片的宽高比，都必须被忽略。当考虑生成旋转视频的 MediaStreamTrack 时，必须使用未旋转的分辨率进行检查。无论接收机是否支持执行接收侧旋转(例如，通过协调视频定向(CVO) [TS26.114])，这都是必需的，因为这大大简化了匹配逻辑。
- 如果属性包含一个“sar=”(样本宽高比)值设置为“1.0”以外的值，表明接收者想要接收非正方形像素，这不能满足，属性绝对不能被使用。
- 如果编码器的分辨率超过属性允许的最大大小，并且允许编码器调整其分辨率，编码器应该应用降尺度以满足限制。一定不要改变图片的宽高比的编码，忽略任何微小的差异，由于舍入。例如，如果编码器的分辨率是1280x720，而属性的最大指定值是640x480，那么预期的输出分辨率将是640x360。如果不能应用降尺度，则绝对不能使用该属性。
- 如果编码器分辨率小于该属性允许的最小大小，则一定不要使用该属性;编码器一定不能应用升级。JSEP实现应该通过允许接收任意小的分辨率(可能通过回退到软件解码器)来避免这种情况。
- 如果编码器分辨率在最大和最小尺寸范围内，则不需要任何操作。

#### 3.7. Simulcast

JSEP 支持 MediaStreamTrack 的 Simulcast 传输，其中媒体源的多个编码可以在一个 m-line 的上下文中传输。当前的 JSEP API 设计用于允许应用程序发送 Simulcast 媒体，但只接收单一编码。这允许在多用户场景中，每个发送客户端向服务器发送多个编码，然后服务器为每个接收客户端选择要转发的适当编码。

应用程序通过在 RtpSender上配置多个编码来请求支持 Simulcast。在生成报价或答案时，这些编码通过相应的 m-line 上的 SDP 标记表示，如下所述。理解并愿意接收 Simulcast 的接收器也将包括 SDP 标记来表示他们的支持，而 JSEP 端点将使用这些标记来确定是否允许给定的 RtpSender 进行 Simulcast。如果没有协商同步传输支持，RtpSender 将只使用第一个配置的编码。

请注意，Simulcast 的确切参数取决于发送程序。虽然前面提到的 SDP 标记是为了确保远端可以接收和分解多个 Simulcast 的编码，但在 JSEP 中，用于每个编码的具体分辨率和比特率仅由发送端决定。

JSEP 目前还没有提供一种机制来配置 Simulcast 的接收。这意味着，如果远端提供了 simulcast，则 JSEP端点生成的 answer 将不指示是否支持接收simulcast，因此远端将只发送每个 m-line 的单个编码。

此外，JSEP 没有提供处理来自 JSEP 端点 simulcast offer 请求的机制。这意味着，在 JSEP 端点收到初始 offer 的情况下，设置 simulcast 需要带外信令或 SDP 检查。然而，如果 JSEP 端点在其初始 offer 中设置了 simulcast，则任何已建立的 simulcast 流将在收到传入的重新 offer 后继续工作。该规范的未来版本可能会添加额外的 API 来处理传入的初始 offer 场景。

当使用 JSEP 从 RtpSender 发送多个编码时，使用来自 [RFC8853] 和 [RFC8851] 的技术。具体来说，当一个 RtpSender 被配置了多个编码时，RtpSender 的 m-line 将包含一个 "a=simulcast" 属性，正如在 [RFC8853] 5.1 节中定义的那样，带有一个 "send" simulcast 描述并列出了每个期望的编码，而没有 "recv" simulcast 描述。m-line 还将包含每个编码的 "a=rid" 属性，如 [RFC8851] 4 节中指定的；使用限制标识符(rid，也称为 rid-ids 或 RtpStreamIds )可以消除各个编码的歧义，即使它们都属于同一个 m-line。

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

应用程序可以指定使用 bundle 的首选策略，bundle 的多路复用机制定义在 [RFC8843] 中。无论策略如何，应用程序将总是尝试协商捆绑到一个单一的传输，并将提供一个单一 bundle 组的所有 m-line；这个单一传输的使用取决于接受 bundle 的应答器。但是，通过从下面的列表中指定策略，应用程序可以精确地控制它将媒体流捆绑在一起的积极程度，这将影响它与 "non-bundle-aware" 端点的互操作方式。当与 "non-bundle-aware" 终端协商时，只有未标记为 "bundle-only" 的流将被建立。

可用的策略集合如下:

- balanced:
每个类型(音频、视频或应用程序)的第一个 m-line 将包含传输参数，这将允许应答者拆分该部分。每个类型的第二个和后续的 m-line 将被标记为 "bundle-only"。结果是，如果有N种不同的媒体类型，那么将为 N 个媒体流收集候选媒体。这一政策平衡了实现多元化的期望和确保基本音频和视频仍然可以在遗留方案中协商的需要。当作为应答者时，如果 offer 中没有bundle 组，实现将拒绝所有类型的 m-line 。
- max-compat:
所有 m-line 将包含传输参数但不标记为 "bundle-only"。这个策略将允许所有的流都被 "non-bundle-aware" 的端点接收，但是需要为每个媒体流收集单独的候选。
- max-bundle:
只有第一个 m-line 将包含传输参数，除第一个流之外的所有流将被标记为 "bundle-only"。该策略旨在最小化候选集合和最大化复用，以降低与历史端点的兼容性为代价。当作为应答者时，实现将拒绝除第一个 m-line 区段之外的任何 m-line，除非它们与 m-line 在同一个 bundle 组中。

由于提供了性能和与遗留端点兼容性之间的最佳权衡，默认 bundle 策略必须设置为 "balanced"。

应用程序可以指定使用 RTP/RTCP 传输复用 [RFC5761] 的首选策略，使用以下策略之一:

- negotiate:
JSEP实现将收集 RTP 和 RTCP 候选，但也将提供 "a=rtcp-mux"，从而允许兼容多路或非多路复用终端。
- require:
JSEP 实现将只收集 RTP 候选，并将在它生成的 offer 中任何新的 m-line 中插入 "a=rtcp-mux-only" 。这样一来，候选收集者需要收集的候选数量就减少了一半。使用不包含"a=rtcp-mux" 属性的 m-line 描述将导致返回错误。

默认的复用策略必须设置为 "require"。实现可以选择拒绝应用程序设置多路复用策略为 "negotiate" 的尝试。

#### 4.1.2. addTrack

addTrack 方法为 PeerConnection 添加一个 MediaStreamTrack，使用 MediaStream 参数将该 track 与同一 MediaStream 中的其他 track 关联起来，这样当创建一个 offer 或 answer 时，它们可以被添加到相同的 "LS"(Lip Synchronization)组。将 track 添加到相同的 "LS" 组表明，这些 track 的播放应该同步以进行正确的lip sync，如[RFC5888]，第 7 节所述。addTrack 试图最小化收发器的数量，如下所示：如果PeerConnection处于 "have-remote-offer" 状态，该 track 将被附加到第一个兼容的 transceiver 上，该 transceiver 是由最近的 setRemoteDescription 调用创建的，并且没有本地 track。否则，将创建一个新的 transceiver，如 4.1.4 节所述。

#### 4.1.3. removeTrack

removeTrack 方法从 PeerConnection 中删除 MediaStreamTrack，使用 RtpSender 参数来指示哪个 sender 的 track 应该被删除。清除 sender 的 track 后，sender 停止发送。调用 createOffer 后，如果是 transceiver，方向由 sendrecv 变为 recvonly，或者由 sendonly 变为 inactive 。

#### 4.1.4. addTransceiver

addTransceiver 方法增加一个新的 RtpTransceiver 到 PeerConnection。如果提供了 MediaStreamTrack 参数，则 transceiver 将被配置为该媒体类型，并且 track 将被附加到 transceiver 上。否则，应用程序必须显式地指定类型；这种模式对于创建 recvonly transceiver 非常有用，对于创建可以在以后附加 track 的 transceiver 也非常有用。

应用程序可以在创建的时候指定 transceiver 方向属性、关联的 MediaStreams (允许 "LS" 组分配)以及一组媒体编码(用于在 3.7 节中描述的 simulcast)。

#### 4.1.5. onaddtrack 事件

当 setRemoteDescription 调用后生成了一个新的 remote track 时，onaddtrack 事件被通知给应用程序。新 track 在事件中被作为 MediaStreamTrack 对象，与该 track 所在的 MediaStream 一起包含在事件参数中。

#### 4.1.6. createDataChannel

createDatachannel 方法创建一个新的数据通道并将其附加到 PeerConnection。如果当前 PeerConnection 没有数据通道，则需要一次新的 offer/answer 交换。在一个给定的PeerConnection 上的所有数据通道共享相同的 SCTP/DTLS 关联("SCTP" 代表 "Stream Control Transmission Protocol")，因此相同的 m-line，后续的数据通道的创建不会对 JSEP 状态产生任何影响。

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

大多数应用程序不需要使用 "pranswer" 类型创建 answers。虽然 这样可以对 offer 立即响应，为了尽快建立会话运输，防止媒体裁剪发生，JSEP 优先处理应用程序创建和发送一个 "sendonly" 最终答案空 MediaStreamTrack 后立即收到 answer，这将阻止媒体由呼叫者发送，并允许媒体在被呼叫者回答后立即发送。稍后，当被调用方实际接受该调用时，应用程序可以插入真正的 MediaStreamTrack 并创建一个新的 "sendrecv" offer 来更新之前的 offer/answer对，并启动双向媒体流。虽然这也可以通过 "sendonly" pranswer 后接 "sendrecv" answer 来完成，但最初的 pranswer 保留 offer/answer 交换开放，这意味着呼叫者不能在这段时间内发送更新的 offer。

例如，考虑一个典型的 JSEP 应用程序，它希望尽可能快地设置音频和视频。当接收到一个 offer 包含了音频和视频 MediaStreamTracks，它将发送一个直接的 answer 并将这些 track 设置为 sendonly (这意味着接受者将不会发送任何媒体，因为尚未添加自己的 MediaStreamTracks，发起者也不会发送任何媒体)。然后，它将等待用户接受呼叫并获取所需的本地 track。在用户接受后，应用程序将载入它获得的 track，这时 ICE 握手和 DTLS 握手可能已经完成，可以立即开始传输。应用程序还将向远端发送一个新的 offer，指示呼叫接受，并将音频和视频设置为 "sendrecv"。7.3 节给出了一个详细的用例流程。

当然，一些应用程序可能无法执行这种双重 offer/answer 交换，特别是那些使用历史信令协议的应用程序。在这些情况下，pranswer 仍然可以为应用程序提供一种机制来进行传输预连接。

##### 4.1.10.2. Rollback

在某些情况下，可能需要 "rollback" 对 setLocalDescription 或 setRemoteDescription 调用后进行更改。考虑这样一种情况，一个呼叫正在进行，一方希望更改一些会话参数，该端生成一个更新的 offer，然后调用 setLocalDescription。但是，远端(在 setRemoteDescription 之前或之后)决定它不想接受新参数，并向提供程序发送拒绝消息。现在，提供者(可能还有回答者)需要返回到 "stable" 状态和先前的 offer/answer 描述。为了支持这一点，我们引入了 "rollback" 的概念，它丢弃对会话的任何建议更改，将状态机返回到 "stable" 状态。回滚是通过向 setLocalDescription 或 setRemoteDescription 提供带有空内容的类型为 "rollback" 的会话描述来执行的。

#### 4.1.11. setLocalDescription

setLocalDescription 方法指示 PeerConnection 应用提供的会话描述作为它的本地配置。type 字段指示该描述应被处理为 offer 、provisional answer、final answer 还是 rollback；offer 和 answer 是不同的检查，与已存在的每个SDP line 规则兼容。

这个 API 改变了本地媒体的状态；除此之外它还创建了接收和解码媒体的本地资源。为了使应用程序可以支持改变媒体格式的场景，PeerConnection 必须能够同时支持使用当前和等待本地描述(例如,支持所有已存在的编解码器格式)。这种双重处理开始于 PeerConnection 进入 "have-local-offer" 状态时，并一直持续到 setRemoteDescription 传入一个 final answer 或者 rollback。

这个API间接地控制候选的收集过程。当提供了本地描述，并且当前使用的传输数量与本地描述所需的传输数量不匹配时，PeerConnection 将根据需要创建传输，并开始为每个传输收集候选，如果可用，使用候选池中的候选。

如果(1)setRemoteDescription 被提前调用并带有一个 offer，(2)setLocalDescription被调用带有一个答案(临时或最终)，(3)媒体方向是兼容的，(4)媒体可用来发送，这将导致媒体传输的开始。

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


#### 4.1.20. onicecandidate 事件

### 4.2. RtpTransceiver

#### 4.2.1. stop

#### 4.2.2. stopped

#### 4.2.3. setDirection

#### 4.2.4. direction

#### 4.2.5. currentDirection

#### 4.2.6. setCodecPreferences

