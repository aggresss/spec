# HTTP Semantics

> 原文 [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)

## 摘要

超文本传输协议 (HTTP) 是一种用于分布式、协作的超文本信息系统的无状态应用层协议。本文档描述了 HTTP 的整体架构，建立了通用术语，并从协议中所有版本共享的层面进行定义。在这个定义中包括核心协议元素、可扩展性机制以及 "http" 和 "https" 统一资源标识符 (URI) 方案。

本文档更新了RFC 3864，废止了 RFC 2818、7231、7232、7233、7235、7538、7615、7694 和 7230 的部分内容。

## 1. Introduction

### 1.1. Purpose

Hypertext Transfer Protocol (HTTP) 是一系列无状态、应用层、基于请求/响应的协议，它们共享通用接口、可拓展语义和自描述信息，以实现与基于网络的超文本信息系统的灵活交互。

HTTP 通过向客户端提供独立于所提供资源类型的统一接口，隐藏了服务如何实现的细节。同样，服务端不需要知道每个客户端的目的:可以孤立地考虑请求，而不是与特定类型的客户端或预定的应用程序步骤序列相关联。这允许在许多不同的上下文中有效地使用通用实现，降低交互复杂性，并支持随着时间的推移而独立地发展。

HTTP 还被设计为用作中介协议，其中代理和网关可以将非 HTTP 信息系统转换为更通用的接口。

这种灵活性的一个后果是协议不能根据接口背后发生的事情来定义。相反，我们被限制在定义通信的语法、接收到的通信的意图和接收方的预期行为。如果孤立地考虑通信，那么成功的操作应该反映在服务器提供的可观察接口的相应更改中。然而，由于多个客户端可能并行操作，并且可能出于不同的目的，我们不能要求这些更改在单个响应的范围之外是可观察到的。

### 1.2. History and Evolution

HTTP 自 1990 年引入以来一直是万维网的主要信息传输协议。它最初是一种用于低延迟请求的简单机制，使用一个 GET 方法请求传输由给定路径名标识的假定超文本文档。随着 Web 的发展，HTTP 得到了扩展，可以在消息中包含请求和响应，使用类似 MIME 的媒体类型传输任意数据格式，并通过中介路由请求。这些协议最终定义为 HTTP/0.9 和 HTTP/1.0 (参见 [HTTP/1.0])。

HTTP/1.1 旨在改进协议的功能，同时保持与现有的基于文本的消息语法的兼容性，提高其在互联网上的互操作性、可伸缩性和健壮性。这包括：

- 用于固定和动态(分块)内容的基于长度的数据分隔符
- 用于内容协商的一致框架
- 用于条件请求的不透明验证器
- 用于更好的缓存一致性的缓存控件
- 用于部分更新的范围请求
- 默认持久连接。

HTTP/1.1 于 1995 年引入，1997 年在 Standards Track 上发布 [RFC2068]， 1999 年修订 [RFC2616]，并于 2014 年再次修订 ([RFC7230] 至 [RFC7235])。

HTTP/2 ([HTTP/2]) 在现有 TLS 和 TCP 协议的基础上引入了一个多路复用会话层，用于通过高效的字段压缩和服务器推送来交换并发 HTTP 消息。

HTTP/3 ([HTTP/3]) 通过使用 基于 UDP 的 QUIC 作为安全多路传输协议，而不是 TCP，为并发消息提供了更大的独立性。

HTTP 的三个主要版本都依赖于本文档定义的语义。它们并没有互相淘汰，因为每一种都有特定的好处和限制，这取决于使用的环境。具体实现应该为其特定的上下文选择最合适的传输和消息传递语法。

当前的 HTTP 修订文档从 HTTP/1.1 消息传递语法 ([HTTP/1.1]) 中分离了语义(本文档)和缓存 ([CACHING]) 的定义，以允许每个主要协议版本在引用相同的核心语义的同时独立发展。

### 1.3. Core Semantics

HTTP 通过发送操作或传输表示的消息 (第3.2节)，提供了与资源交互的统一接口 (第3.1节) —— 无论其类型、性质或实现如何。

每条消息要么是请求，要么是响应。客户端构造用于传递其意图的请求消息，并将这些消息路由到已识别的源服务器。服务器侦听请求，解析接收到的每个消息，解释与已识别的目标资源相关的消息语义，并用一个或多个响应消息响应该请求。客户端检查收到的响应，以确定其意图是否被执行，并根据收到的状态代码和内容确定下一步要做什么。

HTTP 语义包括每个请求方法定义的意图 (第9节)、可能在请求头字段中描述的语义扩展、描述响应的状态代码 (第15节) 以及可能在响应字段中给出的其他控制数据和资源元数据。

HTTP 语义还包括描述接收者打算如何解释内容的表示元数据、可能影响内容选择的请求报头字段以及统称为“内容协商”的各种选择算法 (第12节)。

### 1.4. Specifications Obsoleted by This Document

Title | Reference | See
--|--|--
HTTP Over TLS | [RFC2818] | B.1
HTTP/1.1 Message Syntax and Routing [*] | [RFC7230] | B.2
HTTP/1.1 Semantics and Content | [RFC7231] | B.3
HTTP/1.1 Conditional Requests | [RFC7232] | B.4
HTTP/1.1 Range Requests | [RFC7233] | B.5
HTTP/1.1 Authentication | [RFC7235] | B.6
HTTP Status Code 308 (Permanent Redirect) | [RFC7538] | B.7
HTTP Authentication-Info and Proxy-Authentication-Info Response Header Fields | [RFC7615] | B.8
HTTP Client-Initiated Content-Encoding | [RFC7694] | B.9

[*] 本文档仅废弃了 RFC7230 中独立于 HTTP/1.1 消息传递语法和连接管理的部分; RFC7230 的剩余部分由 "HTTP/1.1"[HTTP/1.1] 废止。

## 2. Conformance

### 2.1. Syntax Notation

本规范使用[RFC5234]的 Augmented Backus-Naur Form (ABNF)，扩展了[RFC7405]中定义的字符串区分大小写的表示法。

它还使用了在第 5.6.1 节中定义的列表扩展，允许使用 "#" 操作符 (类似于 "*" 操作符表示重复)紧凑地定义逗号分隔的列表。附录 A 显示了收集到的语法，其中所有列表操作符扩展为标准 ABNF 表示法。

按照惯例，ABNF 规则名前缀为 "obs-" 表示由于历史原因而出现的过时语法规则。

参考 [RFC5234] 附录 B.1 定义的核心规则包括: ALPHA(字母)、CR(回车)、CRLF(CR LF)、CTL(控制符)、DIGIT(十进制0-9)、DQUOTE(双引号)、HEXDIG(十六进制0-9/A-F/a-f)、HTAB(水平制表符)、LF(换行)、OCTET(任何8位数据序列)、SP(空格)、VCHAR(任何可见的 US-ASCII 字符)。

第 5.6 节定义了字段值的一些通用语法组件。

本规范使用了 [RFC6365] 中定义的术语 “字符”、“字符编码方案”、“字符集”和“协议元素”。

### 2.2. Requirements Notation

本文档中的关键词“必须”、“不得”、“必需”、“应”、“不应”、“建议”、“不建议”、“可”和“可选”在所有大写字母出现时（如图所示）应按照 BCP 14 [RFC2119] [RFC8174] 所述进行解释。

该规范根据 HTTP 通信中参与者的角色确定一致性标准。因此，需求被放在发送者、接收者、客户端、服务器、用户代理、中介体、原始服务器、代理、网关或缓存上，这取决于需求所约束的行为。当实现、资源所有者和协议元素注册超出单个通信的范围时，会对它们提出额外的要求。

当需求只应用于创建协议元素的实现，而不是向下游转发接收到的元素的实现时，使用动词 "generate" 而不是 "send"。

如果一个实现符合与它在 HTTP 中所扮演的角色相关的所有需求，那么它就被认为是符合的。

发送方绝对不能生成与相应 ABNF 规则定义的语法不匹配的协议元素。在给定的消息中，发送方绝对不能生成仅允许由其他角色(即发送方没有该消息的角色)的参与者生成的协议元素或语法替代方案。

对 HTTP 的一致性既包括对所使用协议版本的特定消息传递语法的一致性，也包括对所发送的协议元素的语义的一致性。例如，一个客户端声称符合 HTTP/1.1，但未能识别 HTTP/1.1 接收方所需的功能，将无法与根据这些声明调整响应的服务器进行互操作。反映用户选择的特性，如内容协商和用户选择的扩展，可以影响超出协议流的应用程序行为;发送不准确反映用户选择的协议元素将使用户感到困惑并抑制选择。

当实现在语义一致性上失败时，该实现的消息的接收者最终将开发变通方法来相应地调整其行为。如果解决办法仅限于发生故障的实现，接收方可以在保持符合本协议的情况下采用此类解决办法。例如，服务器经常扫描 User-Agent 字段值的一部分，而用户代理经常扫描 Server 字段值，以根据已知的错误或选择不当的默认值调整自己的行为。

### 2.3. Length Requirements

接收方应该防御性地解析接收到的协议元素，只期望该元素符合其 ABNF 语法并适合合理的缓冲区大小。

HTTP对它的许多协议元素没有特定的长度限制，因为合适的长度可能会有很大的不同，这取决于部署上下文和实现的目的。因此，发送方和接收方之间的互操作性取决于对每个协议元素的合理长度的共同期望。此外，在过去 30 年的 HTTP 使用过程中，通常理解的一些协议元素的合理长度已经发生了变化，并且预计在未来还会继续发生变化。

至少，接收方必须能够解析和处理协议元素长度，这些长度至少与它在其他消息中为这些相同的协议元素生成的值相同。例如，将很长的 URI 引用发布到其自身资源的源服务器需要在接收到目标 URI 时能够解析和处理这些相同的引用。

许多接收到的协议元素只在识别和向下游转发该元素所需的范围内进行解析。例如，中介可能将接收到的字段解析为其字段名和字段值组件，但随后转发该字段，而无需在字段值内部进一步解析。

### 2.4. Error Handing

接收方必须根据本规范为其定义的语义(包括本规范的扩展)解释接收到的协议元素，除非接收方(通过经验或配置)确定发送方错误地实现了这些语义所暗示的内容。例如，如果对 User-Agent 报头字段的检查表明某个特定的实现版本在接收到某些内容编码时失败，那么源服务器可能会忽略接收到的 Accept-Encoding 报头字段的内容。

除非另有说明，接收方可以尝试从无效的构造中恢复可用的协议元素。HTTP 不定义特定的错误处理机制，除非这些机制对安全性有直接影响，因为协议的不同应用程序需要不同的错误处理策略。例如，Web 浏览器可能希望从 Location 报头字段没有根据 ABNF 解析的响应中透明地恢复，而系统控制客户端可能认为任何形式的错误恢复都是危险的。

在底层连接失败的情况下，一些请求可以被客户端自动重试，如 9.2.2 节所述。

### 2.5. Protocol Version

HTTP的版本号由 "." 分隔的两个十进制数字组成。(句号或小数点)。第一个数字(主要版本)表示消息语法，而第二个数字(次要版本)表示发送方符合(能够理解以便将来通信)的主要版本中最高的次要版本。

虽然 HTTP 的核心语义在协议版本之间不会改变，但它们的表达式可能会改变，因此当对有线格式进行不兼容的更改时，HTTP版本号也会改变。此外，HTTP允许对协议进行增量的、向后兼容的更改，而无需通过使用定义的扩展点更改其版本 (第16节)。

协议版本作为一个整体表明发送方是否符合该版本对应规范中规定的要求集。例如，"HTTP/1.1" 版本是由本文档的 "HTTP缓存"[Caching] 和 "HTTP/1.1"[HTTP/1.1] 的组合规范定义的。

当引入不兼容的消息语法时，HTTP 的主版本号会增加。当对协议所做的更改增加消息语义或暗示发送方的额外功能时，次要编号将增加。

即使发送方只使用向后兼容的协议子集，次要版本也会公布发送方的通信能力，从而让接收方知道可以在响应(服务器)或未来的请求(客户端)中使用更高级的功能。

当 HTTP 的主版本没有定义任何次要版本时，暗示次要版本 "0"。"0" 用于在需要次要版本标识符的元素中引用该协议。

## 3. Terminology and Core Concepts

HTTP 是为万维网 (WWW) 架构而创建的，并随着时间的推移不断发展，以支持全球超文本系统的可伸缩性需求。该体系结构的大部分内容反映在用于定义 HTTP 的术语中。

### 3.1. Resources

HTTP 请求的目标称为“资源”。HTTP 不限制资源的性质；它仅仅定义了一个可能用于与资源交互的接口。大多数资源由统一资源标识符 (Uniform Resource Identifier, URI) 标识，如第4节所述。

HTTP 的一个设计目标是将资源标识与请求语义分离，这可以通过在请求方法 (第9节) 和一些请求修改头字段中嵌入请求语义来实现。资源不能以与请求方法的语义不一致的方式处理请求。例如，尽管资源的 URI 可能意味着不安全的语义，但客户端可以期望资源在使用安全方法处理请求时避免不安全的操作 (参见章节9.2.1)。

HTTP 依赖统一资源标识符(Uniform Resource Identifier, URI) 标准[URI] 来指示目标资源(章节7.1)和资源之间的关系。

### 3.2. Representations

“表示” 是旨在反映给定资源的过去、当前或期望状态的信息，其格式可以通过协议随时进行通信。一个表示由一组表示元数据和一个潜在的无界表示数据流组成(第8节)。

HTTP 允许“信息隐藏”在其统一的接口后面，通过定义与资源状态的可传输表示相关的通信，而不是传输资源本身。这允许URI标识的资源是任何东西，包括时间函数，如“拉古纳海滩的当前天气”，同时潜在地提供在消息生成时表示该资源的信息[REST]。

统一接口类似于一个窗口，通过这个窗口，人们只能通过将消息传递给另一边的独立行动者来观察事物并对其采取行动。在我们的通信中，需要一个共享的抽象来表示(“取代”)该事物的当前或期望状态。当一个表示是超文本时，它既可以提供资源状态的表示，也可以提供帮助指导接收者未来交互的处理指令。

目标资源可能提供或能够生成多个表示，每个表示旨在反映资源的当前状态。通常基于内容协商(第12节)的算法将用于从这些表示中选择最适用于给定请求的一种。这个“选定的表示”为评估条件请求(第13节)提供了数据和元数据，并为GET的200 (OK)、206(部分内容)和304(未修改)响应构造内容(第9.3.1节)。

### 3.3. Connections, Clients, and Servers

HTTP 是一个在可靠的传输层或会话层“连接”上运行的客户端/服务器协议。

HTTP “客户端”是为了发送一个或多个 HTTP 请求而与服务器建立连接的程序。HTTP “服务器”是一个接受连接的程序，通过发送HTTP响应来服务HTTP请求。

术语客户端和服务器仅指这些程序为特定连接执行的角色。同一个程序可能在某些连接上充当客户端，在其他连接上充当服务器。

HTTP被定义为无状态协议，这意味着可以孤立地理解每个请求消息的语义，并且连接和其中的消息之间的关系对这些消息的解释没有影响。例如，一个CONNECT请求(章节9.3.6)或一个带有Upgrade报头字段的请求(章节7.8)可以在任何时候发生，而不仅仅是在连接的第一条消息中。许多实现依赖于HTTP的无状态设计，以便在多个服务器之间重用代理连接或动态负载平衡请求。

因此，服务器绝对不能假定同一连接上的两个请求来自同一个用户代理，除非连接是安全的并且特定于该代理。一些非标准的HTTP扩展(例如，[RFC4559])已经被发现违反了这一要求，导致了安全性和互操作性问题。

### 3.4. Messages

HTTP 是一种无状态的请求/响应协议，用于在连接之间交换“消息”。术语“发送方”和“接收方”分别指发送或接收给定消息的任何实现。

客户端以带有方法(第9节)和请求目标(第7.1节)的“请求”消息的形式向服务器发送请求。请求还可能包含用于请求修饰符、客户端信息和表示元数据的报头字段(章节6.3)、用于按照方法处理的内容(章节6.4)以及用于在发送内容时传递收集到的信息的尾字段(章节6.5)。

服务器通过发送一个或多个“响应”消息来响应客户机的请求，每个“响应”消息包括一个状态码(第15节)。响应还可能包含用于服务器信息的报头字段、资源元数据和表示元数据、根据状态代码解释的内容，以及用于在发送内容时传递收集到的信息的尾部字段。

### 3.5. User Agents

术语“用户代理”指的是发起请求的各种客户机程序。

最常见的用户代理形式是通用 Web 浏览器，但这只占实现的一小部分。其他常见的用户代理包括 spider (穿越网络的机器人)、命令行工具、广告牌屏幕、家用电器、秤、灯泡、固件更新脚本、移动应用程序和各种形状和大小的通信设备。

作为用户代理并不意味着在请求时有一个人类用户直接与软件代理交互。在许多情况下，用户代理被安装或配置为在后台运行，并保存其结果以供以后检查(或仅保存那些可能有趣或错误的结果的子集)。例如，通常会给 spider 一个起始 URI，并将其配置为在以超文本图的形式在 Web 上爬行时遵循某些行为。

许多用户代理不能(或选择不)向用户提供交互式建议，或就安全或隐私问题提供足够的警告。在少数情况下，该规范要求向用户报告错误，这种报告只能在错误控制台或日志文件中可见，这是可以接受的。同样地，在继续之前由用户确认自动操作的需求可以通过预先配置选择、运行时选项或简单地避免不安全操作来满足;如果用户已经做出了选择，确认并不意味着任何特定的用户界面或正常处理的中断。

### 3.6. Origin Server

术语“源服务器”是指可以为给定的目标资源发起权威响应的程序。

最常见的原始服务器形式是大型公共网站。然而，就像用户代理被等同于浏览器一样，很容易被误导，认为所有的原始服务器都是一样的。常见的原始服务器还包括家庭自动化单元、可配置的网络组件、办公机器、自主机器人、新闻源、交通摄像头、实时广告选择器和视频点播平台。

大多数 HTTP 通信由一个检索请求 (GET) 组成，该请求用于 URI 标识的某些资源的表示。在最简单的情况下，这可以通过用户代理 (UA) 和源服务器 (O) 之间的单个双向连接 (===) 来完成。

```
         request   >
    UA ======================================= O
                                <   response
```

### 3.7. Intermediaries

HTTP 允许使用中介体通过连接链来满足请求。HTTP “中介”有三种常见形式: 代理、网关和隧道。在某些情况下，单个中介可能充当原始服务器、代理、网关或隧道，根据每个请求的性质切换行为。

```
         >             >             >             >
    UA =========== A =========== B =========== C =========== O
               <             <             <             <
```

上图显示了用户代理和原始服务器之间的三个中介(A、B和C)。在整个链中传递的请求或响应消息将通过四个独立的连接。有些 HTTP 通信选项可能只应用于与最近的非隧道邻居的连接，只应用于链的端点，或者应用于链上的所有连接。尽管图是线性的，但是每个参与者可能同时进行多个通信。例如，在处理 A 的请求的同时，B 可能正在接收来自除 A 以外的许多客户机的请求，并且/或将请求转发到除 C 以外的服务器。同样，以后的请求可能通过不同的连接路径发送，通常基于负载平衡的动态配置。

术语“上游”和“下游”用于描述与消息流相关的定向需求:所有消息都从上游流向下游。术语“入站”和“出站”用于描述与请求路由相关的定向需求:入站意味着“指向源服务器”，而出站意味着“指向用户代理”。

“代理”是一种消息转发代理，通常由客户端通过本地配置规则选择，用于接收某些类型的绝对 URI 请求，并试图通过 HTTP 接口的转换来满足这些请求。有些转换是最小的，例如 “http” uri的代理请求，而其他请求可能需要转换到完全不同的应用程序级协议。代理通常用于通过公共中介对组织的 HTTP 请求进行分组，以实现安全服务、注释服务或共享缓存。有些代理被设计为在转发消息或内容时对选定的消息或内容应用转换，如第7.7节所述。

一个“门户”(又名“门户”)。“反向代理”)是一种中介，充当出站连接的源服务器，但转换接收到的请求并将它们转发到另一个或多个服务器。网关通常用于封装遗留或不受信任的信息服务，通过“加速器”缓存来提高服务器性能，并支持跨多台机器对HTTP服务进行分区或负载平衡。

适用于源服务器的所有HTTP要求也适用于网关的出站通信。网关使用它想要的任何协议与入站服务器通信，包括超出本规范范围的HTTP私有扩展。但是，希望与第三方HTTP服务器进行互操作的 HTTP-to-HTTP 网关需要符合网关入站连接上的用户代理要求。

“隧道”在不改变消息的情况下充当两个连接之间的盲中继。一旦激活，隧道就不被视为 HTTP 通信的一方，尽管隧道可能是由 HTTP 请求发起的。当中继连接的两端关闭时，隧道即停止存在。隧道用于通过中介扩展虚拟连接，例如当传输层安全(TLS， [TLS13])用于通过共享防火墙代理建立机密通信时。

上述中介类别仅考虑充当 HTTP 通信参与者的中介。还有一些中介体可以作用于网络协议栈的较低层，在消息发送方不知情或不允许的情况下过滤或重定向 HTTP 流量。网络中介体(在协议级别上)与路径上的攻击者难以区分，由于错误地违反 HTTP 语义，通常会引入安全缺陷或互操作性问题。

例如，“拦截代理”[RFC3040](通常也称为“透明代理”[RFC1919])与HTTP代理不同，因为它不是由客户端选择的。相反，拦截代理过滤或重定向传出的 TCP 端口 80 包(偶尔还有其他常见端口流量)。拦截代理通常出现在公共网络接入点上，作为在允许使用非本地Internet服务之前强制帐户订阅的一种手段，并在公司防火墙内强制执行网络使用策略。

### 3.8. Caches

“缓存”是以前响应消息的本地存储，以及控制其消息存储、检索和删除的子系统。缓存存储可缓存的响应，以减少未来等效请求的响应时间和网络带宽消耗。任何客户端或服务器都可以使用缓存，但缓存不能在充当隧道时使用。

缓存的效果是，如果链上的一个参与者有一个适用于该请求的缓存响应，那么请求/响应链就会缩短。如果 B 有一个未被 UA 或 A 缓存的请求的 O(通过 C ) 早期响应的缓存副本，则如下所示的结果链。

```
            >             >
       UA =========== A =========== B - - - - - - C - - - - - - O
                  <             <
```

如果允许缓存存储响应消息的副本以用于响应后续请求，则响应是“可缓存的”。即使响应是可缓存的，客户机或源服务器也可能对何时可以将缓存的响应用于特定请求设置额外的约束。缓存行为和可缓存响应的HTTP要求在[CACHING]中定义。

在万维网上和大型组织内部部署了各种各样的缓存体系结构和配置。其中包括全国层次结构的代理缓存，以节省带宽和降低延迟，使用网关缓存优化流行站点的区域和全球分布的内容交付网络，广播或组播缓存条目的协作系统，在脱机或高延迟环境中使用的预取缓存条目的存档，等等。

### 3.9. Example Message Exchange

下面的例子说明了一个典型的 HTTP/1.1 消息交换，用于 URI "http://www.example.com/hello.txt" 上的 GET 请求(章节9.3.1):

Client request:

```
GET /hello.txt HTTP/1.1
User-Agent: curl/7.64.1
Host: www.example.com
Accept-Language: en, mi
```

Server response:

```
HTTP/1.1 200 OK
Date: Mon, 27 Jul 2009 12:28:53 GMT
Server: Apache
Last-Modified: Wed, 22 Jul 2009 19:15:56 GMT
ETag: "34aa387-d-1568eb00"
Accept-Ranges: bytes
Content-Length: 51
Vary: Accept-Encoding
Content-Type: text/plain

Hello World! My content includes a trailing CRLF.
```

## 4. Identifiers in HTTP

统一资源标识符 (URI) [URI]在整个 HTTP 中用作标识资源的手段(章节3.1)。

### 4.1. URI References

URI 引用用于定位请求、指示重定向和定义关系。

“URI-reference”、“absolute-URI”、“relative-part”、“authority”、“port”、“host”、“path-abempty”、“segment”、“query” 等定义均采用 URI 泛型语法。为可以包含非空路径组件的协议元素定义了“绝对路径”规则。(此规则与 RFC3986 中的 path-abempty 规则略有不同，后者允许使用空路径，而path-absolute规则不允许以 “//” 开头的路径。) “partial-URI” 规则是为可以包含相对 URI 但不包含片段组件的协议元素定义的。

```
  URI-reference = <URI-reference, see [URI], Section 4.1>
  absolute-URI  = <absolute-URI, see [URI], Section 4.3>
  relative-part = <relative-part, see [URI], Section 4.2>
  authority     = <authority, see [URI], Section 3.2>
  uri-host      = <host, see [URI], Section 3.2.2>
  port          = <port, see [URI], Section 3.2.3>
  path-abempty  = <path-abempty, see [URI], Section 3.3>
  segment       = <segment, see [URI], Section 3.3>
  query         = <query, see [URI], Section 3.4>

  absolute-path = 1*( "/" segment )
  partial-URI   = relative-part [ "?" query ]
```

HTTP 中允许 URI 引用的每个协议元素都将在其 ABNF 生成中指示该元素是否允许任何形式的引用 (URI-reference)、是否只允许绝对形式的 URI (absolute-URI)、是否只允许路径和可选查询组件(partial-URI)，或以上几种类型的某种组合。除非另有说明，否则URI引用将相对于目标URI进行解析(章节7.1)。

建议所有发送方和接收方至少支持协议元素中长度为8000字节的uri。注意，这意味着某些结构和在线表示(例如，HTTP/1.1中的请求行)在某些情况下必然会更大。

### 4.2. HTTP-Related URI Schemes

IANA在 <https://www.iana.org/assignments/uri-schemes/> 维护 URI 方案 [BCP35] 的注册表。尽管请求可能针对任何URI方案，但以下方案是HTTP服务器固有的:

URI Scheme | Description | Section
--|--|--
http | Hypertext | Transfer Protocol | 4.2.1
https | Hypertext Transfer Protocol Secure | 4.2.2

请注意，“http” 或 “https” URI的存在并不意味着始终有一个 http 服务器在标识的原点侦听连接。任何人都可以创建 URI，无论服务器是否存在，以及服务器当前是否将该标识符映射到资源。注册名称和 IP 地址的委托性质将创建一个联邦名称空间，无论是否存在 HTTP 服务器。

#### 4.2.1. http URI Scheme

这里定义了 “http” URI方案，用于在分层命名空间内创建标识符，该命名空间由侦听给定端口上 TCP ([TCP]) 连接的潜在 http 源服务器所治理。

```
http-URI = "http" "://" authority path-abempty [ "?" query ]
```

“http”URI的源服务器由授权组件标识，它包括一个主机标识符([URI]，章节3.2.2)和可选端口号([URI]，章节3.2.3)。如果端口子组件为空或没有给出，TCP端口80(为WWW服务预留的端口)是默认的。原点决定了谁有权权威地响应以已标识资源为目标的请求，如第4.3.2节所定义。

发送方绝对不能生成带有空主机标识符的“http”URI。处理此类URI引用的接收方必须将其视为无效而拒绝。

分层路径组件和可选查询组件在源服务器的名称空间中标识目标资源。

#### 4.2.2. https URI Scheme

这里定义了 “https” URI 方案，用于在分层命名空间内创建标识符，该命名空间由潜在的源服务器管理，侦听给定端口上的 TCP 连接，并能够建立已为 HTTP 通信提供安全保护的TLS ([TLS13])连接。在这种情况下，“安全的”具体指的是服务器已经被认证为代表所识别的授权机构，并且与该服务器的所有HTTP通信都具有客户机和服务器都可接受的机密性和完整性保护。

```
 https-URI = "https" "://" authority path-abempty [ "?" query ]
```

“https” URI 的源服务器由授权组件标识，它包括一个主机标识符([URI]，章节3.2.2)和可选端口号([URI]，章节3.2.3)。如果端口子组件为空或未给出，则 TCP 端口 443 (HTTP over TLS的预留端口) 是默认值。原点决定了谁有权权威地响应以标识资源为目标的请求，如第 4.3.3 节中定义的那样。

发送方绝对不能生成带有空主机标识符的 “https” URI。处理此类 URI 引用的接收方必须将其视为无效而拒绝。

分层路径组件和可选查询组件在源服务器的名称空间中标识目标资源。

客户端必须确保其对 “https” 资源的 HTTP 请求在通信之前是安全的，并且它只接受对这些请求的安全响应。请注意，客户端和服务器可以接受的加密机制的定义通常是协商的，并且可以随着时间的推移而改变。

通过 “https” 方案提供的资源与 “http” 方案没有共享标识。它们是具有独立名称空间的不同起源。然而，对 HTTP 的扩展被定义为应用于具有相同主机的所有源，例如 Cookie 协议[Cookie]，允许一个服务设置的信息影响与匹配主机域中的其他服务的通信。这种扩展的设计应该非常谨慎，以防止从安全连接获得的信息在不安全的上下文中被无意地交换。

#### 4.2.3. http(s) Normalization and Comparison

带有 “http” 或 “https” 方案的 URI 根据[URI]第 6 节中定义的方法进行规范化和比较，并使用上述每种方案的默认值。

HTTP 不需要使用特定的方法来确定等价性。例如，缓存键可以作为一个简单的字符串进行比较，在基于语法的归一化之后，或者在基于方案的归一化之后。

“http” 和 “https” URI 的基于方案的规范化([URI]第6.2.3节)涉及以下附加规则:

- 如果端口等于某个方案的默认端口，一般形式是省略端口子组件。
- 当不被用作 OPTIONS 请求的目标时，空路径组件相当于“/”的绝对路径，因此正常形式是提供 “/” 的路径。
- scheme 和 host 不区分大小写，通常以小写形式提供；所有其他组件都以区分大小写的方式进行比较。
- 除“保留”集中的字符外，其他字符等效于它们的百分比编码的八字节:正常形式是不编码它们(参见[URI]的2.1和2.2节)。

例如，以下三个uri是等价的:

```
   http://example.com:80/~smith/home.html
   http://EXAMPLE.com/%7Esmith/home.html
   http://EXAMPLE.com:/%7esmith/home.html
```

可以假设规范化(使用任何方法)后等价的两个 HTTP uri 标识相同的资源，并且任何 HTTP 组件都可以执行规范化。因此，不同的资源不应该被标准化后等效的 HTTP URI 标识(使用[URI]章节6.2中定义的任何方法)。

#### 4.2.4. Deprecation of userinfo in http(s) URIs

授权的 URI 通用语法还包括 userinfo 子组件([URI]， Section 3.2.1)，用于在 URI 中包含用户身份验证信息。在该子组件中，不建议使用 “user:password” 格式。

有些实现将 userinfo 组件用于身份验证信息的内部配置，例如在命令调用选项、配置文件或书签列表中，尽管这种使用可能会暴露用户标识符或密码。

当在消息中生成 “http” 或 “https” URI 引用作为目标 URI 或字段值时，发送方绝对不能生成 userinfo 子组件(及其“@”分隔符)。

在使用来自不可信源的 “http” 或 “https” URI 引用之前，接收方应该解析 userinfo 并将其作为错误处理;这很可能是为了网络钓鱼攻击而被用来掩盖授权。

#### 4.2.5. http(s) References with Fragment Identifiers

### 4.3 Authoritative Access

#### 4.3.1. URI Origin

#### 4.3.2. http Origins

#### 4.3.3. https Origins

#### 4.3.4. https Certificate Verification

#### 4.3.5. IP-ID Reference Identity

## 5. Fields

### 5.1. Field Names

### 5.2. Field Lines and Combined Field Value

### 5.3. Field Order

### 5.4. Field Limits

### 5.5. Field Values

### 5.6. Common Rules for Defining Field Values

#### 5.6.1. Lists (#rule ABNF Extension)

##### 5.6.1.1. Sender Requirements

##### 5.6.1.2. Recipient Requirements

#### 5.6.2. Tokens

#### 5.6.3. Whitespace

#### 5.6.4. Quoted Strings

#### 5.6.5. Comments

#### 5.6.6. Parameters

#### 5.6.7. Date/Time Formats

## 6. Message Abstraction

### 6.1. Framing and Completeness

### 6.2. Control Data

### 6.3. Header Fields

### 6.4. Content

#### 6.4.1. Content Semantics

#### 6.4.2. Identifying Content

### 6.5. Trailer Fields

#### 6.5.1. Limitations on Use of Trailers

#### 6.5.2. Processing Trailer Fields

### 6.6. Message Metadata

#### 6.6.1. Date

#### 6.6.2. Trailer

## 7. Routing HTTP Messages

### 7.1. Determining the Target Resource

### 7.2. Host and :authority

### 7.3. Routing Inbound Requests

#### 7.3.1. To a Cache

#### 7.3.2. To a Proxy

#### 7.3.3. To the Origin

### 7.4. Rejecting Misdirected Requests

### 7.5. Response Correlation

### 7.6. Message Forwarding

#### 7.6.1. Connection

#### 7.6.2. Max-Forwards

#### 7.6.3. Via

### 7.7. Message Transformations

### 7.8. Upgrade

## 8. Representation Data and Metadata

### 8.1. Representation Data

### 8.2. Representation Metadata

### 8.3. Content-Type

#### 8.3.1. Media Type

#### 8.3.2. Charset

#### 8.3.3. Multipart Types

### 8.4. Content-Encoding

#### 8.4.1. Content Codings

##### 8.4.1.1. Compress Coding

##### 8.4.1.2. Deflate Coding

##### 8.4.1.3. Gzip Coding

### 8.5. Content-Language

#### 8.5.1. Language Tags

### 8.6. Content-Length

### 8.7. Content-Location

### 8.8. Validator Fields

#### 8.8.1. Weak versus Strong

#### 8.8.2. Last-Modified

##### 8.8.2.1. Generation

##### 8.8.2.2. Comparison

#### 8.8.3. ETag

##### 8.8.3.1. Generation

##### 8.8.3.2. Comparison

##### 8.8.3.3. Example: Entity Tags Varying on Content-Negotiated Resources

## 9. Methods

### 9.1. Overview

### 9.2. Common Method Properties

#### 9.2.1. Safe Methods

#### 9.2.2. Idempotent Methods

#### 9.2.3. Methods and Caching

### 9.3. Method Definitions

#### 9.3.1. GET

#### 9.3.2. HEAD

#### 9.3.3. POST

#### 9.3.4. PUT

#### 9.3.5. DELETE

#### 9.3.6. CONNECT

#### 9.3.7. OPTIONS

#### 9.3.8. TRACE

## 10. Message Context

### 10.1. Request Context Fields

#### 10.1.1. Expect

#### 10.1.2. From

#### 10.1.3. Referer

#### 10.1.4. TE

#### 10.1.5. User-Agent

### 10.2. Response Context Fields

#### 10.2.1. Allow

#### 10.2.2. Location

#### 10.2.3. Retry-After

#### 10.2.4. Server

### 11. HTTP Authentication

#### 11.1. Authentication Scheme

#### 11.2. Authentication Parameters

#### 11.3. Challenge and Response

#### 11.4. Credentials

#### 11.5. Establishing a Protection Space (Realm)

#### 11.6. Authenticating Users to Origin Servers

##### 11.6.1. WWW-Authenticate

##### 11.6.2. Authorzation

##### 11.6.3. Authentication-Info

#### 11.7. Authencating Clients to Proxies

##### 11.7.1. Proxy-Authenticate

##### 11.7.2. Proxy-Authorization

##### 11.7.3. Proxy-Authentication-Info

## 12. Content Negotiation

### 12.1. Proactive Negotiation

### 12.2. Reactive Negotiation

### 12.3. Request Content Negotiation

### 12.4. Content Negotitation Field Features

#### 12.4.1. Absence

#### 12.4.2. Quality Values

#### 12.4.3. Wildcard Values

### 12.5. Content Negotiation Fields

#### 12.5.1. Accept

#### 12.5.2. Accept-Charset

#### 12.5.3. Accept-Encoding

#### 12.5.4. Accept-Language

#### 12.5.5. Vary

## 13. Conditional Requests

### 13.1. Preconditions

#### 13.1.1. If-Match

#### 13.1.2. If-None-Match

#### 13.1.3. If-Modified-Since

#### 13.1.4. If-Unmodified-Since

#### 13.1.5. If-Range

## 14. Range Requests

### 14.1. Range Units

#### 14.1.1. Range Specifiers

#### 14.1.2. Byte Ranges

### 14.2. Range

### 14.3. Accept-Ranges

### 14.4. Content-Range

### 14.5. Partial PUT

### 14.6. Media Type multipart/byteranges

## 15. Status Codes

### 15.1. Overview of Status Codes

### 15.2. Infomational 1xx

#### 15.2.1. 100 Continue

#### 15.2.2. 101 Switching Protocols

### 15.3. Successful 2xx

#### 15.3.1. 200 OK

#### 15.3.2. 201 Created

#### 15.3.3. 202 Accepted

#### 15.3.4. 203 Non-Authoritative Information

#### 15.3.5. 204 No Content

#### 15.3.6. 205 Reset Content

#### 15.3.7. 206 Partial Content

##### 15.3.7.1. Single Part

##### 15.3.7.2. Multiple Parts

##### 15.3.7.3. Combining Parts

### 15.4. Redirection 3xx

#### 15.4.1. 300 Multiple Choices

#### 15.4.2. 301 Moved Permanently

#### 15.4.3. 302 Found

#### 15.4.4. 303 See Other

#### 15.4.5. 304 Not Modified

#### 15.4.6. 305 Use Proxy

#### 15.4.7. 306 (Unused)

#### 15.4.8. 307 Temporary Redirect

#### 15.4.9. 308 Permanent Redirect

### 15.5. Client Error 4xx

#### 15.5.1 400 Bad Request

#### 15.5.2 401 Unauthorized

#### 15.5.3 402 Payment Required

#### 15.5.4 403 Forbidden

#### 15.5.5 404 Not Found

#### 15.5.6 405 Method Not Allowed

#### 15.5.7 406 Not Acceptable

#### 15.5.8 407 Proxy Authentication Required

#### 15.5.9 408 Request Timeout

#### 15.5.10 409 Conflict

#### 15.5.11 410 Gone

#### 15.5.12 411 Length Required

#### 15.5.13 412 Precondition Failed

#### 15.5.14 413 Content Too Large

#### 15.5.15 414 URI Too Long

#### 15.5.16 415 Unsupported Media Type

#### 15.5.17 416 Range Not Satisfiable

#### 15.5.18 417 Expectation Failed

#### 15.5.19 418 (Unused)

#### 15.5.20 421 Misdirected Request

#### 15.5.21 422 Unprocessalbe Content

#### 15.5.22 426 Upgrade Required

### 15.6. Server Error 5xx

#### 15.6.1. 500 Internal Server Error

#### 15.6.2. 501 Not Implemented

#### 16.6.3. 502 Bad Gateway

#### 16.6.4. 503 Service Unavailable

#### 16.6.5. 504 Gateway Timeout

#### 16.6.6. 505 HTTP Version Not Supported

## 16. Extending HTTP

### 16.1. Method Extensibility

#### 16.1.1. Method Registry

#### 16.1.2. Considerations for New Methods

### 16.2. Status Code Extensibility

#### 16.2.1. Status Code Registry

#### 16.2.2. Considerations for New Status codes

### 16.3. Field Extensibility

#### 16.3.1. Field Name Registry

#### 16.3.2. Considerations for New Field Names

##### 16.3.2.1. Considerations for New Field Names

##### 16.3.2.2. Considerations for New Field Values

### 16.4. Authentication Scheme Extensibility

#### 16.4.1. Authentication Scheme Registry

#### 16.4.2. Consierations for New Authentication Schemes

### 16.5. Range Unit Extensibility

#### 16.5.1. Range Unit Registry

#### 16.5.2. Considerations for New Range Units

### 16.6. Content Coding Extensibility

#### 16.6.1. Content Coding Registry

#### 16.6.2. Considerations for New Content Codings

### 16.7. Upgrade Token Registry


