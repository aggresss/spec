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

### 3.1. Resources

### 3.2. Representations

### 3.3. Connections, Clients, and Servers

### 3.4. Messages

### 3.5. User Agents

### 3.6. Origin Server

### 3.7. Intermediaries

### 3.8. Caches

### 3.9. Example Message Exchange

## 4. Identifiers in HTTP

### 4.1. URI References

### 4.2. HTTP-Related URI Schemes

#### 4.2.1. http URI Scheme

#### 4.2.2. https URI Scheme

#### 4.2.3. http(s) Normalization and Comparison

#### 4.2.4. Deprecation of userinfo in http(s) URIs

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


