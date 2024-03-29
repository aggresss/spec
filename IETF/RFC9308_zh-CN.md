# Applicability of the QUIC Transport Protocol

> 原文 [https://datatracker.ietf.org/doc/html/rfc9308](https://datatracker.ietf.org/doc/html/rfc9308)

## 摘要

本文档讨论 QUIC 传输协议的适用性，重点关注影响 QUIC 上的应用程序协议开发和部署的注意事项。其目标受众是 QUIC 的应用程序协议映射的设计者以及这些应用程序协议的实现者。

## 1. Introduction

QUIC [QUIC] 是一种新的传输协议，提供许多高级功能。虽然最初是为 HTTP 用例设计的，但它提供了可用于更广泛应用程序的功能。QUIC 封装在 UDP 中。QUIC 版本 1 集成了 TLS 1.3 [TLS13] 来加密所有有效负载数据和大多数控制信息。使用 QUIC 的 HTTP 版本称为 HTTP/3 [QUIC-HTTP]。

本文档为想要使用 QUIC 协议而不自行实现该协议的应用程序开发人员提供了指导。这包括通过 HTTP/3 或直接通过 QUIC 运行的应用程序的一般指南。

在以下部分中，我们将讨论 QUIC 适用性的具体注意事项以及应用程序开发人员在使用 QUIC 作为其应用程序的传输时必须考虑的问题。

## 2. The Necessity of Fallback

QUIC 使用 UDP 作为底层。这使得用户空间实现成为可能，并允许穿透网络中间件（包括 NAT），而无需更新现有网络基础设施。

测量研究表明，3% [Trammell16] 到 5% [Swett16] 的网络会阻止所有 UDP 流量，但几乎没有证据表明与 TCP [Edeline16] 相比，UDP 流量存在其他形式的系统劣势。这种阻塞意味着在 QUIC 之上运行的所有应用程序必须准备好接受此类网络上的连接故障，或者进行设计以回退到某些其他传输协议。对于 HTTP，此后备是基于 TCP 的 TLS。

IETF 传输服务 (TAPS) 规范 [TAPS-ARCH] 描述了具有适用于多种协议的通用 API 的系统。这对于 QUIC 尤其重要，因为它解决了多个协议之间回退的影响。

具体来说，需要避免回退到不安全协议或较弱版本的安全协议。一般来说，实现回退的应用程序需要考虑安全后果。TCP 和 TLS 的回退会将控制信息暴露给网络中的修改和操作。此外，降级到 QUIC 版本 1 中使用的 1.3 之前的 TLS 版本可能会导致加密保护明显减弱。例如，协议协商 [RFC7301] 的结果只有在使用 TLS 1.3 时才具有机密性保护。

在缺乏 QUIC 提供的后备协议中不存在的功能的情况下，这些应用程序必须运行（可能功能受损）。对于回退到 TCP 上的 TLS，最明显的区别是 TCP 不提供流复用，因此如果需要，需要在应用层实现流复用。此外，TCP 实现和网络路径通常不支持 TCP 快速打开 (TFO) 选项 [RFC7413]，该选项允许将有效负载数据与新连接的第一个控制数据包一起发送，QUIC 中的 0-RTT 会话恢复也提供了这种功能。请注意，有一些证据表明，即使 TFO 已成功协商，中间件也会阻止 SYN 数据（请参阅 [PaaschNanog]）。即使 Fast Open 成功地进行端到端操作，它也仅限于 TLS 握手和应用程序数据的单个数据包，这与 QUIC 0-RTT 不同。

此外，虽然加密（在本例中为 TLS）与 QUIC 密不可分，但通过 TCP 进行的 TLS 协商可能会被阻止。如果无法支持 TCP 上的 TLS，则应中止连接，然后应用程序应向用户显示安全通信不可用的适当提示。

总之，任何后备机制都可能会降低性能并降低安全性；但是，回退不得默默地违反应用程序对其有效负载数据的机密性或完整性的期望。

## 3. 0-RTT

QUIC 提供 0-RTT 连接建立。尽管 TLS 1.3 和 TCP 中存在相同的功能，但 0-RTT 为使用 QUIC 的应用程序带来了机遇和挑战。

从使用它的应用程序的角度来看，提供 0-RTT 连接建立的传输协议与不提供 0-RTT 的传输协议有本质上的不同。关闭和重新打开连接的成本与尝试保持连接打开的成本之间的相对权衡是不同的；参见 第 3.2 节。

应用程序需要刻意选择使用 0-RTT，因为 0-RTT 存在重放攻击的风险。使用 0-RTT 的应用程序协议需要一个配置文件来描述可以安全发送的信息类型。对于 HTTP，此配置文件在 [HTTP-REPLAY] 中描述。

### 3.1. Replay Attacks

重传或恶意重放 0-RTT 数据包中包含的数据可能会导致服务器端收到相同数据的多个副本。

客户端以 0-RTT 数据包发送的应用程序数据如果重放，可能会被多次处理。应用程序需要了解在 0-RTT 中发送的内容是安全的。寻求启用 0-RTT 的应用协议需要仔细分析并描述可以在 0-RTT 中发送的内容；请参阅 [QUIC-TLS] 的第 5.6 节。

在某些情况下，将 0-RTT 中发送的应用程序数据限制为不会对服务器产生持久影响的操作可能就足够了。启动数据检索或建立配置是安全操作的示例。幂等操作（重复操作与单个操作具有相同净效果的操作）可能是安全的。然而，也可以将单独的幂等操作组合成非幂等操作序列。

一旦服务器接受 0-RTT 数据，就无法选择性地丢弃接收到的数据。但是，协议可以定义拒绝单个操作的方法，这些操作如果重播可能会不安全。

某些 TLS 实现和部署可能能够提供部分甚至完整的重放保护，这可用于管理重放风险。

### 3.2. Session Resumption versus Keep-Alive

由于 QUIC 封装在 UDP 中，因此使用 QUIC 的应用程序必须处理较短的网络空闲超时。部署的有状态网络中间件通常会在发送的第一个数据包上建立 UDP 流的状态，并在比 TCP 短得多的空闲时间内保持状态。[RFC5382] 建议 TCP 空闲期至少为 124 分钟，尽管文献中没有证据表明该准则得到广泛实施。然而，UDP 的短网络超时是有据可查的。根据 2010 年的一项研究 ([Hatonen10])，UDP 应用程序可以假设任何 NAT 绑定或其他状态条目都可以在不活动 30 秒后过期。 [RFC8085] 的第 3.5 节进一步讨论了 UDP 的保活间隔：它要求最小值为 15 秒，但建议更大的值，或者完全省略保活。

通过使用连接 ID，QUIC 被设计为对超时后 NAT 重新绑定具有鲁棒性。但是，只有当一个端点在其对等方使用的地址上保持可用性并且对等方是超时发生后要发送的端点时，这才会有帮助。

某些 QUIC 连接对于 NAT 重新绑定可能不稳健，因为路由基础设施（特别是负载均衡器）使用地址/端口 4 元组来引导流量。此外，具有地址转换以外功能的中间件仍然可能影响路径。特别地，一些防火墙不接纳防火墙不具有从客户端发送的相应数据包的最近状态的服务器流量。

QUIC 应用程序可以调整空闲时间来管理超时风险。空闲周期和网络空闲超时与连接空闲超时不同，连接空闲超时定义为任一端点空闲超时参数的最小值；请参见 [QUIC] 的第 10.1 节。有以下三个选项：

- 如果应用层协议仅包含没有空闲期或空闲期很短的交互，或者协议对 NAT 重新绑定的抵抗能力足够，则可以忽略此问题。
- 确保没有长时间闲置。
- 在长时间空闲后恢复会话，在适当的时候使用 0-RTT 恢复。

第一种策略是最简单的，但它仅适用于某些应用程序。

QUIC 应用程序中的服务器或客户端都可以发送 PING 帧作为保持活动状态，以防止连接和任何路径状态超时。使用保持活动的建议是特定于应用程序的，主要取决于应用程序的延迟要求和消息频率。在这种情况下，应用程序映射必须指定客户端或服务器是否负责保持应用程序处于活动状态。虽然 [Hatonen10] 建议当 NAT 处于路径上时，30 秒可能是公共互联网的合适值，但如果部署能够始终在 NAT 重新绑定中幸存下来或已知处于受控环境中（例如，数据中心），则更大的值是更好的选择）以降低网络和计算负载。

在长时间空闲期间发送 PING 帧的频率超过每 30 秒可能会导致在某些情况下产生过多的非生产性流量，并且对于功率受限的（移动）设备而言，可能会导致不可接受的功耗。此外，短于 30 秒的超时可能会使处理瞬时网络中断变得更加困难，例如虚拟机 (VM) 迁移或移动期间的覆盖范围丢失。参见 [RFC8085]，特别是第 3.5 节。

或者，客户端（但不是服务器）可以使用会话恢复而不是发送保持活动流量。在这种情况下，想要通过闲置时间超过服务器空闲超时（可从 idle_timeout 传输参数获取）的连接向服务器发送数据的客户端可以简单地重新连接。如果可能，此重新连接可以使用 0-RTT 会话恢复，从而减少重新启动连接所涉及的延迟。当然，这种方法仅在可以安全使用 0-RTT 且客户端是重新启动的对等方的情况下才有效。

恢复和保持活动之间的权衡需要根据每个应用程序进行评估。一般来说，应用程序应该仅在很可能持续通信的情况下使用保活；例如，[QUIC-HTTP] 建议仅在请求未完成时才使用 keep-alives。

## 4. Use of Streams

QUIC 的流复用功能允许应用程序通过单个连接运行多个流，而不会在流之间出现队头阻塞。流数据在帧内承载，线路上的一个 QUIC 数据包可以承载一个或多个流帧。

流可以是单向的或双向的，并且流可以由客户端或服务器发起。只有单向流的发起者才能在其上发送数据。

由于流偏移的编码限制和连接流控制限制，流和连接在每个方向上最多可以承载 2^62 -1 字节。在目前不太可能发生的情况下，应用程序达到此限制，则需要建立新连接。

流可以独立地打开和关闭，优雅地或突然地。应用程序可以通过指示 QUIC 在 STREAM 帧中发送 FIN 位来正常关闭流的出口方向。如果没有对等方生成的 FIN，它无法正常关闭入口方向，就像 TCP 中一样。但是，端点可以突然关闭出口方向或请求其对等方突然关闭入口方向；这些操作是完全相互独立的。

QUIC 不提供用于任何流的异常处理的接口。如果对应用程序至关重要的流被关闭，则应用程序可以在应用程序层上生成错误消息以通知另一端和/或更高层，从而最终终止 QUIC 连接。

应用程序数据到流的映射是特定于应用程序的，并在 [QUIC-HTTP] 中针对 HTTP/3 进行了描述。在设计应用程序对流的使用时，需要应用一些一般原则：

- 单个流提供排序。如果应用程序需要按顺序接收某些数据，则应在同一流上发送该数据。不保证跨流的传输、接收或传递顺序。
- 多个流提供并发性。可以独立处理的数据，因此如果强制按顺序接收，将会遭受队头阻塞，应该通过单独的流传输。
- 流可以提供消息定向并允许取消消息。如果一条消息映射到单个流，则重置流以使未确认的消息过期可用于模拟该消息的部分可靠性。

如果 QUIC 接收方已打开允许的最大并发流，并且发送方指示需要更多流，则不会自动导致接收方增加最大流数量。因此，应用程序在确定如何将数据映射到流时应考虑允许的、当前打开的和当前使用的流的最大数量。

QUIC 为每个流分配一个数字标识符，称为流 ID。虽然这些标识符和流类型之间的关系在 QUIC 版本 1 中明确定义，但未来版本可能会因各种原因改变这种关系。QUIC 实现应该公开每个流的属性（哪个端点发起流、流是单向还是双向、用于流的流 ID）；应用程序应该查询这些属性，而不是尝试从流 ID 推断它们。

为应用程序打开的流分配流标识符的方法可能因传输实现而异。因此，应用程序不应假设特定的流 ID 将分配给尚未分配的流。例如，HTTP/3 使用流 ID 来引用已打开的流，但不对未来的流 ID 或它们的分配方式做出任何假设（请参阅 [QUIC-HTTP] 的第 6 节）。

### 4.1. Stream versus Flow Multiplexing

流仅对应用程序有意义；由于流信息是在 QUIC 的加密边界内携带的，因此给定的数据包不会公开有关数据包内携带哪些流的信息。因此，流复用并不旨在用于在网络处理方面区分流。因此，需要不同网络处理的应用程序流量应通过不同的 5 元组（即多个 QUIC 连接）承载。鉴于 QUIC 能够在连接的第一个 RTT 中发送应用程序数据（如果已成功建立与同一主机的先前连接以提供必要的凭据），建立另一个连接的成本非常低。

### 4.2. Prioritization

流优先级不会暴露给网络或接收器。优先级由发送方管理，QUIC 传输应为应用程序提供一个接口来对流进行优先级排序 [QUIC]。应用程序可以在 QUIC 之上实现自己的优先级方案：在 QUIC 之上运行的应用程序协议可以为信令优先级定义显式消息，例如 [RFC9218] 中为 HTTP 定义的消息。应用程序协议可以定义允许端点根据上下文确定优先级的规则，或者可以提供更高级别的接口并将决定权留给顶部的应用程序。

重传的优先级处理可以由发送方在传输层实现。[QUIC] 建议在新数据之前重新传输丢失的数据，除非应用程序另有指示。当 QUIC 端点使用完全可靠的流进行传输时，重传的优先级在大多数情况下都是有益的，可以填补间隙并释放流量控制窗口。对于部分可靠或不可靠的流，可能不希望对重传的优先级调度高于较高优先级流的数据。对于此类流，QUIC 可以提供显式接口来控制优先级，或者从流的可靠性级别得出优先级决策。

### 4.3. Ordered and Reliable Delivery

QUIC 流可实现有序且可靠的交付。尽管实现可以提供使用流实现部分可靠性或无序交付的选项，但大多数实现都会假设数据是按顺序可靠交付的。

在此假设下，接收流数据的端点可能不会向前推进，直到与流的开头相邻的数据可用。特别是，接收方可能会扣留流控制信用，直到连续数据传送到应用程序为止；请参阅[ QUIC ]的第 2.2 节。为了支持此接收逻辑，端点将发送流数据直至其被确认，从而确保首先发送并确认流开头的数据。

使用不同发送行为并且不与其对等方协商更改的端点可能会遇到性能问题或死锁。

### 4.4. Flow Control Deadlocks

QUIC 流控制（[ QUIC ]第 4 节）提供了一种管理端点对传入数据所具有的有限缓冲区的访问的方法。此机制限制了端点缓冲区中或网络上传输的数据量。但是，限制可能会通过多种方式产生导致连接性能不佳或陷入死锁的条件。

对于任何使用 QUIC 的协议来说，流量控制中的死锁都可能发生，尽管它们是否成为问题取决于实现如何使用数据并提供流量控制信用。了解导致死锁的原因可能有助于实现避免死锁。

流量控制信用更新的大小和速率会影响性能。使用 QUIC 的应用程序通常具有从传输缓冲区读取数据的数据使用者。某些实现可能在传输层和应用层具有独立的接收缓冲区。使用数据并不总是意味着它会被立即处理。然而，常见的实现技术是通过在数据消耗时发出 MAX_DATA 和/或 MAX_STREAM_DATA 帧来向发送方扩展流量控制信用。这些帧的传送受到从接收器到数据发送器的反向通道的延迟的影响。如果没有及时提供信用，发送应用程序可能会被阻止，从而有效地限制发送者。

如果接收者不从传输中增量读取数据，大型应用程序消息可能会产生死锁。如果消息大于可用的流控制信用，并且接收者在接收并传递整个消息之前不释放额外的流控制信用，则可能会发生死锁。即使未达到流流量控制限制，这也是可能的，因为连接流量控制限制可能被其他流消耗。

长度前缀的消息格式使数据消费者更容易将未读的数据保留在传输缓冲区中，从而扣留流量控制信用。如果流量控制限制阻止发送消息的其余部分，则会导致死锁。长度前缀还可以检测此类死锁。当应用程序协议具有可以作为单个单元处理的消息时，自动为整个消息保留流控制信用可以降低这种类型的死锁的可能性。

数据消费者可以在所有数据可用时立即读取所有数据，以使接收者延长流量控制信用并减少死锁的机会。然而，这样的数据消费者可能需要其他方法来让对等方对其为部分处理的消息保留的附加状态负责。

如果不同流上的数据相互依赖，也可能会发生死锁。假设一个流上的数据先于它所依赖的第二个流上的数据到达。如果第一个流未被读取，则可能会发生死锁，从而阻止接收方扩展第二个流的流控制信用。为了减少相互依赖的数据发生死锁的可能性，发送方应确保在流级和连接级流控制信用中都考虑了依赖数据之前，不会发送依赖的数据。

某些死锁情况可以通过使用 STOP_SENDING 或 RESET_STREAM 取消受影响的流来解决。在某些协议中，取消某些流会导致连接终止。

### 4.5. Stream Limit Commitments

QUIC 端点负责传达它们允许其对等点打开的流的累积限制。初始限制使用 initial_max_streams_bidi 和 initial_max_streams_uni 传输参数进行通告。随着流的打开和关闭，它们会被消耗，并且累积总数会增加。可以使用 MAX_STREAMS 框架增加限制，但没有减少限制的机制。一旦达到流限制，就无法再打开更多流，这会阻止使用 QUIC 的应用程序取得进一步进展。在此阶段，可以通过空闲超时或显式关闭来终止连接；参见第 10 节。

使用 QUIC 并传达累积流限制的应用程序可能需要在达到限制之前关闭连接，例如停止服务器以执行计划维护。立即连接关闭会导致主动使用的流突然关闭。根据应用程序使用 QUIC 流的方式，这可能是不受欢迎的或有损行为或性能。

一种更优雅的关闭技术是停止发送对流限制的增加，并允许连接在剩余流被消耗后自然终止。然而，这样做所需的时间取决于对等点，并且不可预测的关闭时间可能不适合应用程序或操作需求。使用 QUIC 的应用程序可以通过开放流限制保持保守，以减少承诺和不确定性。但是，对流限制过于保守会影响流并发性。平衡这些方面可以特定于应用程序及其部署。

可以使用应用程序层的优雅关闭机制来传达在未来某个时刻显式关闭连接的意图，而不是依赖流限制来避免突然关闭。HTTP/3 使用 GOAWAY 帧提供了这样的机制。在 HTTP/3 中，当客户端收到 GOAWAY 帧时，即使累积流限制允许，它也会停止打开新流。相反，客户端将创建一个新连接来打开更多流。一旦旧连接上的所有流都关闭，它就可以通过连接关闭或空闲超时到期后安全地终止（请参阅第 10 节）。

## 5. Packetization and Latency

QUIC 公开了一个接口，为应用程序提供多个流；然而，应用程序通常无法控制如何将通过这些流传输的数据映射到帧中或如何将这些帧捆绑到数据包中。

默认情况下，许多实现会尝试将来自一个或多个流的 STREAM 帧打包到每个 QUIC 数据包中，以最大限度地减少带宽消耗和计算成本（请参阅[ QUIC ]的第 13 节）。如果没有足够的数据来填充数据包，实现可能会等待一小段时间来优化带宽效率而不是延迟。该延迟可以预先配置，也可以根据观察到的应用程序发送模式动态调整。

如果应用程序需要低延迟，并且只需要发送小块数据，则向 QUIC 指示所有数据应立即发送出去可能很有价值。或者，如果应用程序希望使用特定的发送模式，它还可以向 QUIC 提供建议的延迟，指示将帧捆绑到数据包之前需要等待多长时间。

同样，应用程序通常无法控制线路上 QUIC 数据包的长度。QUIC 提供了添加 PADDING 帧来任意增加数据包大小的能力。QUIC 使用填充来确保路径能够在握手期间传输至少一定大小的数据报（请参阅 [QUIC]的第 8.1 和 14.1 节）以及连接迁移后的路径验证（请参阅 [QUIC] 的第 8.2 节））以及数据报分组层 PMTU 发现（DPLPMTUD）（参见[ QUIC ]的第 14.3 节）。

应用程序还可以使用填充来减少有关所发送数据的信息泄漏。QUIC 实现可以公开一个接口，允许应用程序层指定如何应用填充。

## 6. Error Handling

QUIC 建议端点向对等方发出任何检测到的错误信号。错误可能发生在传输层和应用层。传输错误（例如违反协议）会影响整个连接。使用 QUIC 的应用程序可以定义自己的错误检测和信令（例如，参见[ QUIC-HTTP ]的第 8 节）。应用程序错误可能会影响整个连接或单个流。

QUIC 定义了一个错误代码空间，用于传输层的错误处理。QUIC 鼓励端点使用最具体的代码，但允许使用任何适用的代码，包括通用代码。

使用 QUIC 的应用程序定义独立于 QUIC 或其他应用程序的错误代码空间（例如，参见 [QUIC-HTTP] 的第 8.1 节）。应用程序错误代码空间中的值可以在连接级和流级错误之间重用。

连接错误导致连接终止。它们使用 CONNECTION_CLOSE 帧发出信号，该帧包含错误代码和长度可以为零的原因字段。不同类型的 CONNECTION_CLOSE 帧用于指示传输和应用程序错误。

流错误导致流终止。这些是使用 STOP_SENDING 或 RESET_STREAM 帧发出的，这些帧仅包含错误代码。

## 7. Acknowledgment Efficiency

没有扩展的 QUIC 版本 1 使用从 TCP 采用的确认策略（参见 [QUIC] 的第 13.2 节）。也就是说，它建议确认所有其他数据包。然而，生成和处理 QUIC 确认会消耗发送方和接收方的资源。确认还会产生转发成本并有助于链路利用率，这可能会影响某些类型网络的性能。应用程序也许能够通过使用降低确认率的替代策略来提高整体性能。 [QUIC-ACK-FREQUENCY] 描述了一种扩展，用于表示所需的确认延迟，并讨论了用例以及对拥塞控制和恢复的影响。

## 8. Port Selection and Application Endpoint Discovery

一般来说，端口号有两个目的："首先，它们提供多路分解标识符来区分同一对端点之间的传输会话，其次，它们还可以识别进程连接的应用程序协议和关联服务"（第 3 节） [RFC6335]）。正如 [RFC6335] 中所述，由于动态端口分配的封装和机制，可以根据端口号在网络中识别应用程序的假设现在已不太正确。

由于 QUIC 是一种通用传输协议，因此服务器不要求为 QUIC 使用特定的 UDP 端口。对于回退到 TCP 且尚未具有到 UDP 的备用映射的应用程序，通常适合注册（如果需要）并使用与已为该应用程序注册的 TCP 端口相对应的 UDP 端口号。例如，HTTP/3 [ QUIC-HTTP ]的默认端口是 UDP 端口 443，类似于基于 TCP 的 TLS 的 HTTP/1.1 或 HTTP/2。

鉴于网络管理实践中普遍存在端口号明确映射到应用程序的假设，使用无法轻松映射到注册服务名称的端口可能会导致网络元素（例如，使用端口号进行应用程序识别的防火墙）。

应用程序可以定义备用端点发现机制，以允许使用默认端口以外的端口。例如，HTTP/3（[QUIC-HTTP] 的第 3.2 和 3.3 节）指定 HTTP 源使用 HTTP 替代服务 [RFC7838] ，通过使用 "h3" 作为应用层协议协商 (ALPN) [RFC7301] 令牌。

ALPN 允许客户端和服务器协商在给定连接上使用多种协议中的哪一种。因此，基于所提供的 ALPN 令牌，单个 UDP 端口上可能支持多个应用程序。使用 QUIC 的应用程序需要注册 ALPN 令牌以在 TLS 握手中使用。

由于 QUIC 版本 1 推迟定义完整的版本协商机制，HTTP/3 需要 QUIC 版本 1 并定义 ALPN 令牌（"h3"）仅适用于该版本。到目前为止，无论是在 HTTP/3 还是一般情况下，都没有选择单一方法来管理不同 QUIC 版本的使用。使用 QUIC 的应用程序协议需要考虑协议如何管理不同的 QUIC 版本。这些协议的决策可能会受到其他协议（例如 HTTP/3）所做的选择的影响。

### 8.1. Source Port Selection

某些 UDP 协议容易受到反射攻击，攻击者能够将流量引导至第三方以造成拒绝服务。例如，这些源端口与已知容易受到反射攻击的应用程序相关联，这通常是由于服务器配置错误造成的：

- 端口 53 - DNS [RFC1034]
- 端口 123 - NTP [RFC5905]
- 端口 1900 - SSDP [SSDP]
- 端口 5353 - mDNS [RFC6762]
- 端口 11211 - 内存缓存

服务可能会阻止与已知易受反射攻击的协议关联的源端口，以避免处理大量数据包的开销。然而，这种做法对客户端有负面影响——它不仅需要建立新连接，而且在某些情况下可能会导致客户端在一段时间内避免对该服务使用 QUIC 并降级到非 UDP 协议（参见第 2 节）。

因此，鼓励客户端实现避免使用与已知易受反射攻击的协议关联的源端口。请注意，遵循 [RFC6335] 中给出的客户端实现的一般指南 ，使用 49152-65535 范围内的临时端口，可以避免这些端口。请注意，其他源端口也可能是反射向量。

## 9. Connection Migration

QUIC 支持客户端的连接迁移。如果客户端的 IP 地址发生变化，QUIC 端点仍然可以使用 QUIC 标头中的目标连接 ID 字段（请参阅第 11 节）将数据包与现有传输连接相关联。这支持地址信息更改的情况，例如 NAT 重新绑定、本地接口的有意更改、临时 IPv6 地址 [RFC8981] 过期或来自服务器的首选地址指示（[QUIC] 的第 9.6 节））。

如果任何客户端位于或可能位于 NAT 之后，强烈建议服务器使用非零长度的连接 ID。当支持主动迁移时，还强烈建议使用非零长度的连接 ID。如果有意将连接迁移到新路径，则会使用新的连接 ID 来最大程度地减少网络观察者的可链接性。如果提供了非零长度的连接 ID，则另一个 QUIC 端点使用连接 ID 将不同的地址链接到同一连接和实体。

QUIC 版本 1 的基本规范仅支持一次使用单个网络路径，从而支持故障转移用例。需要进行路径验证，以便端点在使用前验证路径，以避免地址欺骗攻击。路径验证至少需要1个RTT，路径迁移后拥塞控制也会重置。因此，迁移通常会对性能产生影响。

QUIC 探测数据包可以同时在多个路径上发送，用于执行地址验证以及测量路径特征。探测数据包不能携带应用程序数据，但可能包含填充帧。端点可以使用有关其接收的信息作为该路径的拥塞控制的输入。应用程序可以使用从探测中学到的信息来决定切换路径。

在 QUIC 版本 1 中，只有客户端可以主动迁移。但是，服务器可以在握手期间指示它们更愿意在握手后将连接转移到不同的地址。例如，这可用于从多个服务器共享的地址移动到服务器实例唯一的地址。服务器可以在 TLS 握手期间在传输参数中提供 IPv4 和 IPv6 地址，并且客户端可以在两者之间进行选择（如果两者都提供）。请参阅 [QUIC] 的第 9.6 节。

## 10. Connection Termination

QUIC 连接以三种方式之一终止：隐式空闲超时、显式立即关闭或显式无状态重置。

QUIC 不提供任何优雅连接终止的机制；使用 QUIC 的应用程序可以定义自己的优雅终止过程（例如，参见 [QUIC-HTTP] 的第 5.2 节）。

QUIC 空闲超时通过传输参数启用。客户端和服务器宣布一个超时时间，连接的有效值为两个值中的最小值。超时时间过后，连接将被静默关闭。因此，应用程序应该能够配置自己的最大值，并且能够访问该连接的计算出的最小值。应用程序可以根据打开或预期连接的数量调整新连接的最大空闲超时，因为较短的超时值可以更快地释放资源。

在流或数据报中交换的应用程序数据会延迟 QUIC 空闲超时。因此，提供自己的保持活动机制的应用程序将保持 QUIC 连接处于活动状态。不提供自己的保持活动的应用程序可以使用传输层机制（请参阅 [QUIC]的第 10.1.2 节和第 3.2 节）。然而，用于控制此类传输行为的 QUIC 实现接口可能会有所不同，从而影响此类方法的稳健性。

立即关闭由 CONNECTION_CLOSE 帧发出信号（参见 第 6 节）。立即关闭会导致所有流立即关闭，这可能会影响应用程序；参见第 4.5 节。

无状态重置是无法访问连接状态的端点的最后手段。接收无状态重置表示不可恢复的错误，与连接错误不同，因为没有提供应用程序层信息。

## 11. Information Exposure and the Connection ID

QUIC 在建立加密上下文之前或因为这些信息打算由网络使用而在标头的未加密部分中向网络公开一些信息。有关 QUIC 可管理性的更多信息，请参阅 [QUIC-MANAGEABILITY]。QUIC 有一个长标头，公开一些附加信息（版本和源连接 ID），而短标头仅公开目标连接 ID。在QUIC版本1中，长标头用于连接建立期间，而短标头用于已建立的连接中的数据传输。

连接 ID 的长度可以为零。可以在每个端点上单独选择零长度连接 ID，也可以在除连接建立期间客户端发送的第一个数据包之外的任何数据包上选择零长度连接 ID。

选择零长度连接 ID 的端点将接收具有零长度目标连接 ID 的数据包。端点需要使用其他信息，例如源和目标 IP 地址以及端口号来识别引用了哪个连接。这可能意味着，如果这些值发生更改，端点将无法成功将数据报与连接匹配，从而导致连接实际上无法承受 NAT 重新绑定或迁移到新路径。

### 11.1. Server-Generated Connection ID

QUIC 支持服务器生成的连接 ID，该连接 ID 在连接建立期间传输到客户端（请参阅 [QUIC] 的第 7.2 节）。负载均衡器后面的服务器可能需要在握手期间更改连接 ID，对服务器的身份或其负载均衡池的信息进行编码，以支持无状态负载均衡。

具有负载均衡器和其他路由基础设施的服务器部署需要确保该基础设施始终将数据包路由到具有连接状态的服务器实例，即使地址、端口或连接 ID 发生变化也是如此。这可能需要服务器和基础设施之间的协调。实现此目的的一种方法是将路由信息编码到连接 ID 中。有关此技术的示例，请参阅 [QUIC-LB]。

### 11.2. Mitigating Timing Linkability with Connection ID Migration

如果 QUIC 端点不发出新的连接 ID，则客户端无法通过使用它们来降低地址迁移的可链接性。选择无法链接到外部观察者的值可确保不同路径上的活动无法使用连接 ID 进行简单关联。

虽然足够强大的连接 ID 生成方案将缓解可链接性问题，但它们并不能提供全面的保护。无论如何，对 6 元组（源地址和目标地址以及迁移的连接 ID）的生命周期的分析可能会暴露这些链接。

在服务器池中连接迁移很少的情况下，观察者关联两个连接 ID 是很简单的。相反，在每个服务器同时处理多个迁移的情况下，即使公开的服务器映射也可能是不够的信息。

针对这些攻击最有效的缓解措施是通过网络设计和/或操作实践，通过使用负载平衡架构将更多流量加载到单个服务器端地址，通过协调迁移时间以尝试增加流量数量。在给定时间同时迁移，或使用其他方式。

### 11.3. Using Server Retry for Redirection

QUIC 提供了一个重试数据包，可以由服务器发送以响应客户端初始数据包。服务器可以在该数据包中选择一个新的连接 ID，客户端将通过发送另一个带有服务器选择的连接 ID 的客户端初始数据包来重试。该机制可用于将连接重定向到不同的服务器，例如，由于性能原因或当服务器池中的服务器逐渐升级并因此可以支持不同版本的QUIC时。

在这种情况下，假设属于某个池的所有服务器都与基于连接 ID 转发流量的负载均衡器合作提供服务。服务器可以选择重试数据包中的连接 ID，以便负载均衡器将下一个初始数据包重定向到该池中的其他服务器。或者，负载均衡器可以直接提供重试卸载，如 [QUIC-RETRY] 中进一步描述。

[RFC5077] 第 4 节中描述的用于构建 TLS 恢复票证的方法提供了一个也可以应用于验证令牌的示例。但是，强烈建议使用更现代的加密算法。

## 12. Quality of Service (QoS) and Diffserv Code Point (DSCP)

QUIC，如 [QUIC] 中所定义，有一个拥塞控制器和恢复处理程序。此设计假设 QUIC 连接的所有数据包，或者至少具有相同的 5 元组 {dest addr，source addr，protocol，dest port，source port}，具有相同的 Diffserv 代码点 (DSCP) [RFC2475] 将接收类似的网络处理，因为有关每个数据包丢失或延迟的反馈被用作拥塞控制器的输入。因此，属于同一连接的数据包应使用单个 DSCP。[RFC7657] 的第 5.1 节讨论了 Diffserv 与数据报传输协议[ RFC7657 ]的交互（在这方面，与 QUIC 的交互类似于流控制传输协议 (SCTP) 的交互）。

当通过单个 QUIC 连接复用多个流时，所选的 DSCP 值应该是与所有复用流请求的最高优先级相关联的值。

如果需要差异化网络处理，例如通过使用不同的 DSCP，则可以使用到同一服务器的多个 QUIC 连接。一般来说，建议尽量减少同一服务器的 QUIC 连接数量，以避免增加开销，更重要的是，避免竞争拥塞控制。

与 Diffserv 的其他用途一样，当数据包进入不支持 DSCP 值的网段时，可能会导致连接未收到其期望的网络处理。当数据包沿着网络路径传输时，也可以对该数据包中的 DSCP 值进行注释，从而更改请求的处理。

## 13. Use of Versions and Cryptographic Handshake

QUIC 中的版本控制可能会完全改变协议的行为，除了一些已声明为不变的标头字段的含义 [QUIC-INVARIANTS]。版本号较高的 QUIC 版本不一定会提供更好的服务，而可能只是提供不同的功能集。因此，应用程序需要能够选择要使用的 QUIC 版本。

新版本可以使用 TLS 1.3 或更高版本以外的加密方案。 [QUIC] 指定当前由 TLS 1.3 实现的加密握手要求，并在单独的规范 [ QUIC-TLS ]中进行描述。执行此拆分是为了启用具有不同加密握手的轻量级版本控制。

[QUIC] 中建立的 "QUIC Versions" 注册表允许临时注册进行实验。注册（也包括实验版本）对于避免冲突非常重要。实验版本不应长期使用或注册为永久版本，以最大限度地降低基于版本号的指纹识别风险。

## 14. Enabling Deployment of New Versions

QUIC 版本 1 未在基本规范中指定版本协商机制，但 [QUIC-VERSION-NEGOTIATION] 提出了提供兼容版本协商的扩展。

此方法使用三阶段部署机制，可以在大型服务器部署中逐步部署和试验多个版本。在此方法中，部署中的所有服务器必须在任何服务器公布新版本（第 2 阶段）之前接受使用新版本的连接（第 1 阶段），并且新版本的身份验证（第 3 阶段）仅在该版本的公布完全部署后才进行。

有关详细信息，请参阅 [QUIC-VERSION-NEGOTIATION] 的第 5 节。

## 15. Unreliable Datagram Service over QUIC

[RFC9221] 指定了 QUIC 扩展，以允许通过 QUIC 发送和接收不可靠的数据报。与直接通过 UDP 操作不同，根据 [RFC8085] ，使用 QUIC 数据报服务的应用程序不需要实现自己的拥塞控制，因为 QUIC 数据报是拥塞控制的。

QUIC 数据报不受流量控制，因此如果接收器过载，数据块可能会被丢弃。虽然 QUIC 的可靠传输服务提供了基于流的接口来通过多个 QUIC 流按顺序发送和接收数据，但数据报服务具有基于无序消息的接口。如果需要，可以在顶部使用应用层成帧，以允许在一个 QUIC 连接上复用单独的不可靠数据报流。
