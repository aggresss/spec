# HTTP/1.1

> 原文 [https://www.rfc-editor.org/rfc/rfc9112](https://www.rfc-editor.org/rfc/rfc9112)

## 摘要

超文本传输协议(HTTP)是一种无状态的应用层协议，适用于分布式、协作的超文本信息系统。本文档介绍了 HTTP/1.1 协议的消息语法、消息解析、连接管理以及相关的安全问题。

该文件淘汰了 [RFC7230] 的部分内容。

## 1. Introduction

超文本传输协议(HTTP)是一种无状态的应用层请求/响应协议，它使用可扩展的语义和自描述消息与基于网络的超文本信息系统进行灵活的交互。HTTP/1.1 定义如下:

- This document
- "HTTP Semantics" [HTTP]
- "HTTP Caching" [CACHING]

本文档指定了如何使用 HTTP/1.1 消息语法、分帧和连接管理机制传达 HTTP 语义。它的目标是为 HTTP/1.1 消息解析器和消息转发中介定义一套完整的需求。

本文档淘汰了 [RFC7230] 中与 HTTP/1.1 消息传递和连接管理相关的部分，这些更改将在附录 C.3 中总结。[RFC7230] 的其他部分被 "HTTP Semantics" [HTTP] 淘汰了。

### 1.1. Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

Conformance criteria and considerations regarding error handling are defined in Section 2 of [HTTP].

### 1.2.

本规范使用了 [RFC5234] 的 ABNF 表示法，扩展了 [RFC7405] 中定义的区分大小写的字符串表示法。

它还使用了 [HTTP] 5.6.1节中定义的列表扩展，该扩展允许使用“#”操作符(类似于“*”操作符表示重复)紧凑地定义逗号分隔的列表。附录A展示了收集到的语法，其中所有的列表操作符扩展为标准的 ABNF 表示法。

作为一种惯例，以“obs-”为前缀的ABNF规则名称表示由于历史原因而出现的过时语法规则。

引用包含以下核心规则，如[RFC5234]，附录B.1中定义的:ALPHA(字母)，CR(回车)，CRLF (CR LF)， CTL(控制)，DIGIT(十进制0-9)，DQUOTE(双引号)，HEXDIG(十六进制0-9/A-F/ A-F)， HTAB(水平制表符)，LF(换行)，OCTET(任意8位数据序列)，SP(空格)，VCHAR(任意可见的[USASCII]字符)。

以下规则定义在[HTTP]中:

```
  BWS           = <BWS, see [HTTP], Section 5.6.3>
  OWS           = <OWS, see [HTTP], Section 5.6.3>
  RWS           = <RWS, see [HTTP], Section 5.6.3>
  absolute-path = <absolute-path, see [HTTP], Section 4.1>
  field-name    = <field-name, see [HTTP], Section 5.1>
  field-value   = <field-value, see [HTTP], Section 5.5>
  obs-text      = <obs-text, see [HTTP], Section 5.6.4>
  quoted-string = <quoted-string, see [HTTP], Section 5.6.4>
  token         = <token, see [HTTP], Section 5.6.2>
  transfer-coding =
                  <transfer-coding, see [HTTP], Section 10.1.4>
```

The rules below are defined in [URI]:

```
  absolute-URI  = <absolute-URI, see [URI], Section 4.3>
  authority     = <authority, see [URI], Section 3.2>
  uri-host      = <host, see [URI], Section 3.2.2>
  port          = <port, see [URI], Section 3.2.3>
  query         = <query, see [URI], Section 3.4>
```


## 2. Message

HTTP/1.1 客户端和服务器通过发送消息进行通信。有关 HTTP 的一般术语和核心概念，请参阅 [HTTP] 第 3 节。

### 2.1. Message Format

HTTP/1.1 消息由起始行、CRLF和字节序列组成，其格式类似于互联网消息格式 [RFC5322]:零个或多个报头字段行(统称为“报头”或“报头部分”)，表示报头部分结束的空行，以及可选的消息体。

```
  HTTP-message   = start-line CRLF
                   *( field-line CRLF )
                   CRLF
                   [ message-body ]
```

消息可以是客户端到服务器的请求，也可以是服务器到客户端的响应。从语法上讲，这两种消息只在起始行(请求行(requests)或状态行(status line)，以及确定消息体长度的算法(第6节)上有所不同。

```
  start-line     = request-line / status-line
```

理论上，客户端可以接收请求，服务器可以接收响应，并通过不同的起始行格式区分它们。在实践中，服务器被实现为只期望一个请求(响应被解释为一个未知或无效的请求方法)，而客户端被实现为只期望一个响应。

HTTP 利用了一些类似于多用途互联网邮件扩展(MIME)的协议元素 [RFC2045]。HTTP 和 MIME 消息之间的区别参见附录 B。

### 2.2. Message Parsing

解析 HTTP 消息的正常过程是将起始行读取到一个结构中，将每个头字段行按字段名读取到一个散列表中，直到空行，然后使用解析后的数据来确定是否需要消息主体。如果指定了消息体，则将其读取为流，直到读取的字节数等于消息体长度或关闭连接。

接收方必须将 HTTP 消息解析为编码为 US-ASCII [USASCII] 超集的字节序列。将 HTTP 消息解析为 Unicode 字符流，而不考虑具体的编码，会产生安全漏洞，这是因为字符串处理库处理包含八位字节 LF (%x0A)的无效多字节字符序列的方式不同。基于字符串的解析器只能在从消息中提取元素之后在协议元素中安全使用，例如在消息解析后描述了各个字段行的头字段行值中。

尽管起始行和字段的行终止符是序列 CRLF，但接收方可以将单个 LF 识别为行终止符并忽略前面的任何 CR。

发送方不能在除内容之外的任何协议元素中生成裸 CR (CR 字符后不紧跟 LF)。这种裸 CR 的接收方必须认为该元素无效，或者在处理元素或转发消息之前用 SP 替换每个裸 CR 。

旧的 HTTP/1.0 用户代理实现可能会在 POST 请求之后发送额外的 CRLF，以解决一些早期服务器应用程序无法读取未以行结束的消息体内容的问题。HTTP/1.1 的用户代理不能在请求之前或之后使用额外的 CRLF。如果希望以行结尾结束请求消息体，那么用户代理必须将终止 CRLF 字节作为消息体长度的一部分。

为了保证健壮性，服务器在接收并解析请求行时应该忽略在请求行之前收到的至少一个空行(CRLF)。

发送方不能在起始行和第一个头字段之间发送空白。

如果接收到起始行和第一个标题字段之间的空白，则必须将消息视为无效而拒绝接收，或者将之前的每一行空白处理掉(即忽略整行，以及所有之前有空白的后续行，直到接收到格式正确的标题字段或标题部分终止)。拒绝或删除无效的空格行是必要的，以防止下游接收方对它们的错误理解，从而容易受到请求走私(章节11.2)或响应分割(章节11.1)攻击。

当服务器只监听 HTTP 请求消息，或处理从起始行开始看起来是 HTTP 请求消息的内容时，接收到的字节序列(除了上面列出的健壮性异常)与 HTTP 消息语法不匹配时，服务器应该返回 400(Bad Request) 响应并关闭连接。

### 2.3. HTTP Version

HTTP uses a "<major>.<minor>" numbering scheme to indicate versions of the protocol. This specification defines version "1.1". Section 2.5 of [HTTP] specifies the semantics of HTTP version numbers.

The version of an HTTP/1.x message is indicated by an HTTP-version field in the start-line. HTTP-version is **case-sensitive**.

```
  HTTP-version  = HTTP-name "/" DIGIT "." DIGIT
  HTTP-name     = %s"HTTP"
```

当一个 HTTP/1.1 消息被发送给一个 HTTP/1.0 接收方 [HTTP/1.0] 或一个版本未知的接收方时，HTTP/1.1 消息被构造成这样，如果忽略所有的更新特性，它可以被解释为一个有效的 HTTP/1.0 消息。这个规范对一些新功能提出了接受者版本要求，这样符合规范的发送方将只使用兼容的功能，直到通过配置或接收消息确定接收方支持 HTTP/1.1。

处理 HTTP 消息的中介体(即除了充当隧道的中介体之外的所有中介体)必须在转发的消息中发送自己的 HTTP 版本，除非有目的地将其降级为上游问题的解决 scheme 。换句话说，在没有确保消息的协议版本与中介体在接收和发送消息时都符合的协议版本相匹配的情况下，中介体不能盲目地转发起始行。当下游接收者使用消息发送者的版本来确定在以后与该发送者通信时使用哪些功能是安全的时，不重写HTTP版本转发HTTP消息可能会导致通信错误。

如果服务器知道或怀疑客户端没有正确实现 HTTP 规范，并且无法正确处理后续版本的响应，比如客户端未能正确解析版本号，或者知道中介盲目转发 HTTP 版本，即使它不符合给定的协议小版本，服务器也会向 HTTP/1.1 请求发送 HTTP/1.0 响应。除非由特定的客户端属性触发，否则不应该执行这种协议降级，例如当一个或多个请求首部字段(例如 User-Agent )与已知出错的客户端发送的值唯一匹配时。

## 3. Request Line

请求行以方法令牌开始，后面跟着一个空格(SP)、请求目标和另一个空格(SP)，最后以协议版本结束。

```
  request-line   = method SP request-target SP HTTP-version
```

尽管请求行语法规则要求每个组件元素由单个 SP 字节分隔，但接收方可以解析以空格分隔的单词边界，除 CRLF 终止符外，将任何形式的空格视为 SP 分隔符，同时忽略前面或后面的空格;这些空白包括以下一个或多个字节:SP、HTAB、VT (%x0B)、FF (%x0C)或裸CR。然而，如果消息有多个接收者，并且每个接收者都有自己对健壮性的独特解释，那么宽松的解析可能会导致请求偷袭安全漏洞(参见 11.2 节)。

如 [HTTP] 的 2.3 节所述，HTTP 对请求行长度没有预定义的限制。如果服务器接收到的方法比它实现的任何方法都长，则应该以 501(Not Implemented) 状态码响应。如果服务器收到的请求目标比它想要解析的URI长，则必须以 414(URI Too Long) 状态码响应(参见 [HTTP] 的 15.5.15 节)。

在实践中可以发现对请求行长度的各种特殊限制。建议所有 HTTP 发送方和接收方至少支持 8000 个字节的请求行长度。

### 3.1. Method

方法令牌表示要在目标资源上执行的请求方法。请求方法区分大小写。

```
  method         = token
```

本规范定义的请求方法可以在 [HTTP] 的第 9 节中找到，还有关于 HTTP 方法注册表的信息和定义新方法时的注意事项。

### 3.2. Request Target

request-target标识应用请求的目标资源。客户端从所需的目标URI派生出请求目标。请求目标有四种不同的格式，这取决于所请求的方法以及请求是否指向代理。

```
  request-target = origin-form
                 / absolute-form
                 / authority-form
                 / asterisk-form
```

在 request-target 中不允许有空格。不幸的是，一些用户代理无法正确编码或排除超文本引用中的空白字符，导致这些被禁止的字符在格式不正确的请求行中作为请求目标被发送。

接收到无效的请求行应该返回 400((Bad Request) 错误或者 301(Moved Permanently) 重定向，并且正确编码了请求目标。接收者不应该尝试自动更正，然后在没有重定向的情况下处理请求，因为无效的请求行可能是故意设计的，以绕过请求链上的安全过滤器。

客户端必须在所有 HTTP/1.1 请求消息中发送 Host 首部字段([HTTP]章节7.2)。如果目标 URI 包含一个授权组件，那么客户端必须发送与该授权组件相同的主机字段值，不包括任何 userinfo 子组件及其“@”分隔符([HTTP]第4.2节)。如果目标URI的权威组件缺失或未定义，则客户端必须发送带有空字段值的 Host 头字段。

对于任何没有 Host 首部字段的 HTTP/1.1 请求，或者包含多个 Host 首部字段的请求，或者Host首部字段的值无效的请求，服务器都必须返回 400(Bad Request) 状态码。

#### 3.2.1. origin-form

request-target 最常见的形式是 “origin-form”。

```
  origin-form    = absolute-path [ "?" query ]
```

当直接向源服务器发出请求时，与CONNECT或服务器范围选项请求(如下所述)不同，客户端必须仅将目标URI的绝对路径和查询组件作为请求目标发送。如果目标URI的路径组件为空，客户端必须发送“/”作为request-target的原始形式中的路径。根据[HTTP] 7.2节的定义，Host首部字段也会被发送。

例如，客户端希望检索标识为的资源的表示

```
  http://www.example.org/where?q=now
```

直接从源服务器打开(或重用)到主机 “www.example.org” 的 80 端口的 TCP 连接，并发送以下内容:

```
GET /where?q=now HTTP/1.1
Host: www.example.org
```

后面是请求消息的其余部分。

#### 3.2.2. absolute-form

在向代理发出请求时，除了 CONNECT 或服务器范围选项请求(如下所述)之外，客户端必须以“绝对形式”发送目标URI作为请求目标。

```
  absolute-form  = absolute-URI
```

如果可能的话，请求代理从有效缓存中服务该请求，或者代表客户端向下一个入站代理服务器或直接向请求目标指定的源服务器发出相同的请求。对这种消息“转发”的要求定义在[HTTP]的章节7.6中。

绝对形式的 request-line 示例如下:

```
GET http://www.example.org/pub/WWW/TheProject.html HTTP/1.1
```

客户端必须在 HTTP/1.1 请求中发送 Host 首部字段，即使请求目标是绝对形式的，因为这允许主机信息通过可能没有实现 Host 的古老 HTTP/1.0 代理转发。

当代理接收到具有绝对形式的 request-target 的请求时，代理必须忽略接收到的 Host 头字段(如果有的话)，并将其替换为 request-target 的主机信息。转发这样一个请求的代理必须根据接收到的 request-target 生成一个新的 Host 字段值，而不是转发接收到的 Host 字段值。

当源服务器接收到具有绝对形式的 request-target 的请求时，源服务器必须忽略接收到的 Host 头字段(如果有的话)，而使用 request-target 的 Host 信息。注意，如果请求目标没有权限组件，在这种情况下将发送一个空的 Host 头字段。

尽管大多数 HTTP/1.1 客户端只会将绝对格式发送给代理，但服务器必须接受请求中的绝对格式。

#### 3.2.3. authority-form

request-target的“授权形式”仅用于连接请求([HTTP]章节9.3.6)。它仅由uri-host和隧道目的端口号组成，用冒号(“:”)分隔。

```
  authority-form = uri-host ":" port
```

当通过一个或多个代理发起连接请求建立隧道时，客户端必须只发送隧道目的端的主机和端口作为请求目标。客户端从目标 URI 的授权组件获得主机和端口，但如果目标URI省略了端口，则发送 scheme 的默认端口。例如，一个到 “http://www.example.com” 的连接请求看起来像下面这样:

```
CONNECT www.example.com:80 HTTP/1.1
Host: www.example.com
```

#### 3.2.4. asterisk-from

request-target的“星号形式”仅用于服务器范围的选项请求([HTTP]章节9.3.7)。

* -form = "*"
当客户端希望请求整个服务器的选项，而不是该服务器的特定命名资源时，客户端必须只发送“*”(%x2A)作为请求目标。例如,

选项* http /1.1
如果代理接收到具有绝对形式的request-target的选项请求，其中URI具有空路径和没有查询组件，那么请求链上的最后一个代理在将请求转发到指定的源服务器时必须发送“*”的request-target。

例如，请求

```
OPTIONS http://www.example.org:8001 HTTP/1.1
```

将由最终代理转发为:

```
OPTIONS * HTTP/1.1
Host: www.example.org:8001
```

连接到“www.example.org”主机的8001端口后。

### 3.3. Reconstructing the Target URI

当请求目标是绝对形式时，目标 URI 就是请求目标。在这种情况下，服务器将将 URI 解析为其通用组件以进行进一步计算。

否则，服务器将从连接上下文和请求消息的各个部分重建目标URI，以识别目标资源([HTTP] 章节 7.1):

- 如果服务器的配置提供了一个固定的 URI scheme ，或者一个可信出站网关提供了一个 scheme ，那么该 scheme 将用于目标 URI。这在大规模部署中很常见，因为网关服务器将接收客户端的连接上下文，并将其替换为它们自己到入站服务器的连接。否则，如果请求是通过安全连接接收的，则目标 URI 的 scheme 是 "https" ;否则，scheme 是 "http" 。
- 如果请求目标采用权限形式，则目标 URI 的权限组件就是请求目标。否则，目标 URI 的权限组件是 Host 头字段的字段值。如果没有 Host 头字段，或者其字段值为空或无效，则目标 URI 的权限组件为空。
- 如果请求目标采用权限形式或星号形式，则目标 URI 的组合路径和查询组件为空。否则，目标 URI 的组合路径和查询组件就是请求目标。
- 一旦如上所述确定了重建的目标 URI 的组件，就可以通过连接 scheme、"://" 、authority 以及合并后的路径和查询组件，重新组合成绝对 URI 形式。

示例1: 通过安全连接接收到以下消息

```
GET /pub/WWW/TheProject.html HTTP/1.1
Host: www.example.org
```

目标 URI 是

```
https://www.example.org/pub/WWW/TheProject.html
```

例2:通过不安全连接接收到的消息

```
OPTIONS * HTTP/1.1
Host: www.example.org:8080
```

目标 URI 是

http://www.example.org:8080

如果目标 URI 的 authority 组件是空的，并且其 URI scheme 需要一个非空的权限(如 “http” 和 “https” 的情况)，则服务器可以拒绝请求或确定是否应用配置的默认值，该默认值与传入连接的上下文一致。上下文可能包括连接细节，如地址和端口，应用了什么安全策略，以及特定于服务器配置的本地定义信息。在进一步处理请求之前，将空权限替换为配置的默认权限。

如果用户代理的预期权限可能与默认权限不同，那么在安全连接的上下文中为权限提供默认名称本质上是不安全的。能够从请求上下文唯一标识授权机构的服务器可以使用该标识作为默认值，而不会有此风险。或者，将请求重定向到一个安全的资源，说明如何获取新客户端，这样可能更好。

请注意，重构客户端的目标URI只是识别目标资源过程的一半。另一半用于确定目标 URI 是否标识了服务器愿意并能够为其发送响应的资源，如 [HTTP] 的 7.4 节所定义。

## 4. Status Line

响应消息的第一行是状态行，由协议版本、一个空格(SP)、状态码和另一个空格组成，以描述状态码的可选文本短语结尾。

```
  status-line = HTTP-version SP status-code SP [ reason-phrase ]
```

尽管 status-line 语法规则要求每个组件元素由单个SP字节分隔，但接收方可以解析以空格分隔的单词边界，除行终止符外，还可以将任何形式的空格视为SP分隔符，同时忽略前面或后面的空格;这些空白包括以下一个或多个字节:SP、HTAB、VT (%x0B)、FF (%x0C)或裸CR。然而，如果消息有多个接收方，并且每个接收方对健壮性都有自己的唯一解释(参见11.1节)，那么宽松解析可能会导致响应分割安全漏洞。

status-code 元素是一个 3 位数的整数代码，描述了服务器试图理解并满足客户端相应请求的结果。接收方根据为该状态码定义的语义(如果该接收方识别了该状态码)解析和解释响应消息的其余部分，或者在未识别特定代码时，根据该状态码的类解析和解释响应消息。

```
status-code = 3DIGIT
```

HTTP 的核心状态码定义在[HTTP]的第15节中，还包括状态码的类别、定义新状态码时的考虑因素，以及用于收集此类定义的 IANA 注册表。

reason-phrase 元素存在的唯一目的是提供与数字状态码相关的文本描述，这主要是出于对早期互联网应用协议的尊重，后者更常用于交互式文本客户端。

```
reason-phrase = 1*(HTAB / SP / VCHAR / obs-text)
```

客户端应该忽略 reason-phrase 内容，因为它不是可靠的信息通道(它可能会根据给定的语言环境进行翻译，被中间设备覆盖，或者在通过其他版本的 HTTP 转发消息时被丢弃)。服务器必须发送分隔状态代码和原因短语的空格，即使原因短语不存在(即状态行以空格结束)。

## 5. Field Syntax

每个字段行由不区分大小写的字段名和冒号(“:”)、可选的前导空格、字段行值和可选的结尾空格组成。

```
field-line = field-name ":" OWS field-value OWS
```

解析字段值的规则在 [HTTP] 的 5.5 节中定义。本节介绍HTTP/1.1消息中包含和提取首部字段的通用语法。

### 5.1. Field Line Parsing

消息使用通用算法进行解析，独立于单个字段名称。给定字段行值内的内容直到消息解释的后一阶段(通常是在消息的整个字段部分处理完毕之后)才被解析。

字段名和冒号之间不允许有空格。在过去，处理这些空白符的不同导致了请求路由和响应处理中的安全漏洞。任何接收到的请求消息，如果首部字段名称和冒号之间有空格，服务器必须拒绝，响应状态码为 400(Bad Request)。在向下游转发消息之前，代理必须从响应消息中删除任何此类空白。

字段行值之前和/或之后可以有可选的空格(OWS);为了保持可读性，最好在字段行值之前使用一个 SP。字段行值不包括开头或结尾的空白:字段行值的第一个非空白八位之前或字段行值的最后一个非空白八位之后发生的 OWS，在解析器从字段行提取字段行值时被排除。

### 5.2. Obsolete Line Folding

从历史上看, HTTP/1.x 字段的值可以扩展到多行，在每个额外的行之前至少使用一个空格或水平制表符(obs-fold)。本规范不支持行折叠，除非是在 "message/http" 媒体类型中(章节10.1)。

```
  obs-fold     = OWS CRLF RWS
               ; obsolete line folding
```

发送方不能生成包含行折叠的消息(即，任何字段行值都与 obs-fold 规则匹配)，除非消息是要在 "message/http" 媒体类型中打包。

服务器在接收到不在 "message/http" 容器中的 obs-fold 请求时，要么发送一个 400(Bad Request) 拒绝该请求，最好是用一个表示来解释过时的行折叠是不可接受的，要么在解释字段值或转发消息之前，将每个接收到的 obs-fold 替换为一个或多个 SP 字节。

如果代理或网关在响应消息中接收到不在 "message/http" 容器中的 obs-fold，则必须丢弃该消息并将其替换为 502(Bad Gateway) 响应，最好是使用解释接收到不可接受的行折叠的表示，或者在解释字段值或转发消息之前，将每个接收到的 obs-fold 替换为一个或多个 SP 字节。

如果用户代理在响应消息中接收到一个不在 "message/http" 容器中的 obs-fold，则必须在解释字段值之前将每个接收到的 obs-fold 替换为一个或多个SP字节。

## 6. Message Body

HTTP/1.1 消息的消息体(如果有的话)用于携带请求或响应的内容([HTTP]的章节 6.4)。除非应用了传输编码(如 6.1 节所述)，否则消息体与内容是相同的。

```
  message-body = *OCTET
```

确定 HTTP/1.1 消息体何时出现的规则因请求和响应而异。

请求中消息体的存在由 Content-Length 或 Transfer-Encoding 头字段表示。请求消息分帧独立于方法语义。

响应中消息体的存在(详见6.3节)取决于响应的请求方法和响应状态码。这对应于 HTTP 语义允许响应内容的时间( [HTTP] 第 6.4.1 节)。

### 6.1. Transfer-Encoding

transfer - encoding头字段列出了与已经(或将要)应用于内容以形成消息体的传输编码序列相对应的传输编码名称。转移编码在第7节中定义。

```
  Transfer-Encoding = #transfer-coding
                       ; defined in [HTTP], Section 10.1.4
```

Transfer-Encoding 类似于 MIME 的 Content-Transfer-Encoding 字段，它被设计用来在 7 位传输服务上安全传输二进制数据([RFC2045]，第6节)。然而，安全传输对于纯 8 位的传输协议有不同的关注点。在 HTTP 中，传输编码的主要目的是准确地划分动态生成的内容。它还用于区分只在传输过程中应用的编码和所选表示的特征编码。

接收者必须能够解析 chunked 传输编码(章节7.1)，因为当内容大小事先未知时， chunked 传输编码在分帧消息中起着至关重要的作用。发送方不得对消息体多次使用 chunked 传输编码(即不允许对已经 chunked 的消息进行 chunked )。如果对请求的内容应用了 chunked 以外的任何传输编码，发送方必须将 chunked 应用为最终的传输编码，以确保消息被正确地分帧。如果对响应的内容应用了 chunked 以外的任何传输编码，发送方必须要么将 chunked 应用为最终传输编码，要么通过关闭连接来终止消息。

例如,

```
Transfer-Encoding: gzip, chunked
```

表示内容已经使用 gzip 编码进行压缩，然后在形成消息体时使用 chunked 编码进行 chunked 。

与内容编码 ( [HTTP] 第 8.4.1 节)不同，传输编码是消息的属性，而不是表示。请求/响应链上的任何接收方都可以对接收到的传输编码进行解码，或者对消息体应用额外的传输编码，假设对传输编码字段值进行了相应的更改。关于编码参数的附加信息可以由本规范未定义的其他头字段提供。

传输编码可以在对 HEAD 请求的响应中发送，也可以在对 GET 请求的 304(Not Modified) 响应中发送( [HTTP] 的 15.4.5 节)，这两种响应都不包含消息体，以表明如果请求是无条件的GET，源服务器将对消息体应用传输编码。不过，这个指示不是必需的，因为响应链上的任何接收方(包括源服务器)都可以在不需要传输编码时删除它们。

服务器不能在任何响应中发送状态码为 1xx(Informational) 或 204(No Content) 的传输编码报头字段。服务器不能在对连接请求的任何 2xx(Successful) 响应中发送 Transfer-Encoding 首部字段( [HTTP] 的 9.3.6 节)。

如果服务器接收到带有它不理解的传输编码的请求消息，则应该使用 501(Not implemented) 进行响应。

传输编码是在 HTTP/1.1 中添加的。通常假定只发布 HTTP/1.0 支持的实现不会理解如何处理传输编码的内容，并且使用传输编码接收的 HTTP/1.0 消息很可能在转发时没有适当处理传输中的 chunked 传输编码。

客户端不能发送包含传输编码的请求，除非它知道服务器将处理 HTTP/1.1 请求(或后续的小版本);这些知识可能是特定用户配置的形式，或者是通过记住先前接收到的响应的版本。服务器不能发送包含传输编码的响应，除非对应的请求表明 HTTP/1.1 (或更高的小版本)。

传输编码的早期实现偶尔会发送 chunked 传输编码用于消息分帧，并发送一个估计的 Content-Length 首部字段供进度条使用。这就是为什么 Transfer-Encoding 被定义为覆盖 Content-Length，而不是相互不兼容。不幸的是，如果任何下游接收方不能根据本规范解析消息，特别是当下游接收方只实现 HTTP/1.0 时，转发这样的消息可能会导致有关请求夹带(章节11.2)或响应分割(章节11.1)攻击的漏洞。

服务器可以拒绝同时包含内容长度和传输编码的请求，或者仅根据传输编码处理这样的请求。无论如何，服务器必须在响应这样的请求后关闭连接，以避免潜在的攻击。

服务器或客户端接收到包含 Transfer-Encoding 首部字段的 HTTP/1.0 消息时，必须将其视为分帧错误，即使 Content-Length 存在，并在处理完消息后关闭连接。消息发送者可能在buffer中保留了消息的一部分，这可能会被连接的进一步使用所误解。

### 6.2. Content-Length

当消息没有传输编码报头字段时，Content-Length 报头字段([HTTP] 8.6节)可以为潜在内容提供预期的大小，以字节的十进制数表示。对于包含内容的消息，Content-Length 字段值提供了确定数据(和消息)在何处结束所必需的分帧信息。对于不包含内容的消息，Content-Length 表示所选表示的大小( [HTTP] 章节 8.6 )。

发送方不能在任何包含传输编码头字段的消息中发送 Content-Length 头字段。

> 注意: HTTP 对 Content-Length 的消息分帧使用与 MIME 中的相同字段有很大不同，在MIME中它是一个可选字段，只在 "message/external-body" 媒体类型中使用。

### 6.3. Message Body Length

消息体的长度由以下因素之一决定(按优先级排序):

1. 对 HEAD 请求的任何响应以及带有 1xx(Informational)、204(No Content) 或 304(Not Modified) 状态代码的任何响应总是由报头字段后的第一行空行终止，而不管消息中出现的报头字段如何，因此不能包含消息正文或尾部部分。
2. 对 CONNECT 请求的任何 2xx(Successful) 响应都意味着连接将在结束报头字段的空行之后立即成为隧道。客户端必须忽略在这样的消息中收到的任何 Content-Length 或 Transfer-Encoding 报头字段。
3. 如果接收到的消息同时带有 Transfer-Encoding 和 Content-Length 报头字段，则 Transfer-Encoding 将覆盖 Content-Length。这样的消息可能表明试图执行请求夹带(第11.2节)或响应分裂(第11.1节)，应该作为错误处理。选择转发消息的中介必须首先删除接收到的 Content-Length 字段并处理 Transfer-Encoding (如下所述)，然后才向下游转发消息。
4. 如果存在 Transfer-Encoding 报头字段，并且 chunked 传输编码(章节7.1)是最终编码，则通过读取和解码 chunked 数据来确定消息体长度，直到传输编码表示数据完成。
    - 如果响应中存在 Transfer-Encoding 报头字段，而 chunked 传输编码不是最终编码，则通过读取连接来确定消息体长度，直到服务器关闭该连接。
    - 如果请求中存在 Transfer-Encoding 报头字段，而 chunked 传输编码不是最终编码，则不能可靠地确定消息体长度;服务器必须响应400(坏请求)状态码，然后关闭连接。
5. 如果收到消息没有传输编码内容长度和无效的头字段,然后消息分帧无效,接收者必须把它当作一个不可恢复的错误,除非字段值可以成功解析为一个以逗号分隔(部分5.6.1 (HTTP)),列表中的所有值都是有效的,列表中的所有值都是相同的(在这种情况下,消息处理的单值用作内容长度字段值)。如果在请求消息中出现不可恢复的错误，服务器必须响应 400(Bad Request) 状态码，然后关闭连接。如果它在代理接收到的响应消息中，代理必须关闭与服务器的连接，丢弃接收到的响应，并向客户端发送 502(Bad Gateway) 响应。如果它在用户代理接收到的响应消息中，用户代理必须关闭与服务器的连接并丢弃接收到的响应。
6. 如果存在有效的 Content-Length 报头字段而没有 Transfer-Encoding，则其十进制值以字节为单位定义预期的消息体长度。如果发送方关闭连接或接收方在接收到指定的字节数之前超时，则接收方必须认为该消息不完整并关闭连接。
7. 如果这是一个请求消息，并且以上都不为真，则消息体长度为零(没有消息体)。
8. 否则，这是一个没有声明消息体长度的响应消息，因此消息体长度由服务器关闭连接之前接收到的字节数决定。

由于无法区分成功完成的、带封闭分隔符的响应消息与被网络故障中断的部分接收消息，因此服务器应该尽可能生成编码或带长度分隔符的消息。关闭分隔符特性的存在主要是为了向后兼容 HTTP/1.0。

> 注意:请求消息从不使用封闭分隔符，因为它们总是显式地按照长度或传输编码进行框架划分，如果两者都不存在，则意味着请求在报头部分之后立即结束。

服务器可以通过返回 411(Length Required) 来拒绝包含消息体但不包含内容长度的请求。

除非应用了 chunked 以外的传输编码，否则发送包含消息体的请求的客户端应该使用有效的 Content-Length 头字段(如果事先知道消息体的长度)，而不是 chunked 传输编码，因为一些现有服务以 411(Length Required) 状态码响应 chunked，即使它们理解 chunked 传输编码。这通常是因为此类服务是通过网关实现的，网关在调用之前需要一个内容长度，并且服务器无法或不愿意在处理之前缓冲整个请求。

发送包含消息体的请求的用户代理必须发送有效的 Content-Length 头字段或使用 chunked 传输编码。除非客户端知道服务器将处理 HTTP/1.1 (或更高版本)请求，否则不能使用 chunked 传输编码;这些知识可以是特定用户配置的形式，也可以是记住之前接收到的响应的版本。

如果已完全接收到连接上最后一个请求的最终响应，并且还有剩余的数据需要读取，则用户代理可以丢弃剩余的数据，或尝试确定该数据是否属于前一个消息体的一部分，如果前一个消息的 Content-Length 值不正确，则可能会出现这种情况。客户端不能将这些额外的数据作为单独的响应处理、缓存或转发，因为这样的行为很容易受到缓存中毒的影响。

## 7. Transfer Codings

传输编码名称用于指示已经、可能或可能需要应用于消息内容的编码转换，以确保通过网络的“安全传输”。这与内容编码的不同之处在于，传输编码是消息的属性，而不是正在传输的表示的属性。

所有传输编码名称都不区分大小写，应该按照 7.3 节的定义，在 HTTP 传输编码注册表中注册。它们用于 Transfer-Encoding (章节6.1)和 TE (章节10.1.4 [HTTP])头字段(后者也定义了 "transfer-coding" 语法)。

### 7.1. Chunked Transfer Coding

chunked 传输编码对内容进行包装，以便将其作为一系列块进行传输，每个块都有自己的大小指示符，后面是一个可选的包含 trailer 字段的 trailer 部分。Chunked 允许将未知大小的内容流作为长度分隔的缓冲区序列进行传输，这使发送端能够保持连接持久性，而接收端能够知道何时已经接收到整个消息。

```
  chunked-body   = *chunk
                   last-chunk
                   trailer-section
                   CRLF

  chunk          = chunk-size [ chunk-ext ] CRLF
                   chunk-data CRLF
  chunk-size     = 1*HEXDIG
  last-chunk     = 1*("0") [ chunk-ext ] CRLF

  chunk-data     = 1*OCTET ; a sequence of chunk-size octets
```

chunk-size 字段是一个十六进制数字字符串，表示块数据的大小，以字节为单位。当接收到一个块大小为 0 的块时， chunked 传输编码就完成了，可能接下来是一个 trailer 段，最后以空行结束。

接收方必须能够解析和解码 chunked 传输编码。

HTTP/1.1 没有定义任何方法来限制 chunked 响应的大小，以便中介可以确保缓冲整个响应。此外，如果接收实现中没有准确表示非常大的块大小，则可能导致溢出或精度损失。因此，接收方必须预料到可能有很大的十六进制数字，并防止由于整数转换溢出或整数表示导致精度损失而导致解析错误。

chunked 编码不定义任何参数。他们的出现应该被视为错误。

#### 7.1.1. Chunk Extensions

chunked 编码允许每个 chunked 包含零个或多个块扩展，紧跟在 chunked 大小之后，以便为每个 chunked 提供元数据(例如签名或散列)、消息中间控制信息或消息体大小的随机化。

```
  chunk-ext      = *( BWS ";" BWS chunk-ext-name
                      [ BWS "=" BWS chunk-ext-val ] )

  chunk-ext-name = token
  chunk-ext-val  = token / quoted-string
```

chunked 编码是特定于每个连接的，在任何高级应用程序有机会检查扩展之前，可能会被每个接收方(包括中间设备)删除或重新编码。因此，块扩展的使用通常仅限于特殊的 HTTP 服务，如“长轮询”(客户端和服务器可以对块扩展的使用有共同的期望)或端到端安全连接中的填充。

接收方必须忽略无法识别的块扩展。服务器应该将请求中收到的块扩展的总长度限制在对所提供的服务合理的范围内，就像它对消息的其他部分应用长度限制和超时，并在超过该长度时生成适当的4xx(客户端错误)响应一样。

#### 7.1.2. Chunked Trailer Section

trailer 允许发送者在 chunked 消息的末尾包含额外的字段，以便提供可能在发送内容时动态生成的元数据，例如消息完整性检查、数字签名或后处理状态。拖车字段的正确使用和限制在 [HTTP] 的第 6.5 节中定义。

```
  trailer-section   = *( field-line CRLF )
```

从消息中删除 chunked 编码的收件人可以选择性地保留或丢弃所接收的 trailer 字段。保留接收到的 trailer 字段的接收方必须将 trailer 字段与接收到的报头字段分开存储/转发，或者将接收到的 trailer 字段合并到报头部分。除非其对应的头字段定义明确允许并指示如何安全地合并尾字段值，否则接收方不得将接收到的尾字段合并到头部分。

#### 7.1.3. Decoding Chunked

A process for decoding the chunked transfer coding can be represented in pseudo-code as:

```
  length := 0
  read chunk-size, chunk-ext (if any), and CRLF
  while (chunk-size > 0) {
     read chunk-data and CRLF
     append chunk-data to content
     length := length + chunk-size
     read chunk-size, chunk-ext (if any), and CRLF
  }
  read trailer field
  while (trailer field is not empty) {
     if (trailer fields are stored/forwarded separately) {
         append trailer field to existing trailer fields
     }
     else if (trailer field is understood and defined as mergeable) {
         merge trailer field with existing header fields
     }
     else {
         discard trailer field
     }
     read trailer field
  }
  Content-Length := length
  Remove "chunked" from Transfer-Encoding
```

### 7.2. Transfer Codings for Compression

The following transfer coding names for compression are defined by the same algorithm as their corresponding content coding:

- compress (and x-compress) See Section 8.4.1.1 of [HTTP].
- deflate See Section 8.4.1.2 of [HTTP].
- gzip (and x-gzip) See Section 8.4.1.3 of [HTTP].

The compression codings do not define any parameters. The presence of parameters with any of these compression codings SHOULD be treated as an error.

### 7.3. Transfer Codings Registry

“HTTP传输编码注册表”为传输编码名称定义了命名空间。它维护在<https://www.iana.org/assignments/http-parameters>。

注册必须包括以下字段:

- Name
- Description
- Pointer to specification text

传输编码的名称不得与内容编码的名称重叠([HTTP]第8.4.1节)，除非编码转换相同，如第7.2节中定义的压缩编码。

当可以接受多种传输编码时，TE头字段([HTTP]的章节10.1.4)使用一个名为 "q" 的伪参数作为 rank 值。将来的转换编码注册不应该定义称为 "q" (大小写不敏感)的参数，以避免歧义。

添加到该命名空间的值需要IETF审查(参见[RFC8126]的第4.8节)，并且必须符合本规范中定义的传输编码的目的。

不希望使用程序名称来标识编码格式，并且不建议在以后的编码中使用。

### 7.4. Negotiating Transfer Codings

TE 字段([HTTP]的10.1.4节)在 HTTP/1.1 中用于表示除了分块之外，客户端还愿意在响应中接受什么传输编码，以及客户端是否愿意在分块传输编码中保留 trailer 字段。

客户端不得以 TE 发送分块的传输编码名称;分块对于 HTTP/1.1 的接收方来说是可以接受的。

下面是三个TE用法的例子。

```
TE: deflate
TE:
TE: trailers, deflate;q=0.5
```

当可以接受多个传输编码时，客户端可以使用不区分大小写的 "q" 参数(类似于在内容协商字段中使用的 qvalues; 参见 [HTTP] 的 12.4.2 节)。排名值是一个 0 到 1 之间的实数，0.001 是最不受欢迎的，1 是最受欢迎的;值为 0 意味着“不可接受”。

如果 TE 字段值为空或没有 TE 字段，则唯一可接受的传输编码将分块。没有传输编码的消息总是可以接受的。

关键字 "trailers" 表示发送者不会丢弃 trailer 字段，如 [HTTP] 的 6.5 节所述。

由于 TE 首部字段只适用于立即连接，TE 的发送方必须在连接首部字段([HTTP]章节7.6.1)中发送一个 "TE" 连接选项，以防止 TE 首部字段被不支持其语义的中介转发。

## 8. Handling Incomplete Messages

服务器接收到不完整的请求消息(通常是由于取消的请求或触发的超时异常)，可能会在关闭连接之前发送错误响应。

接收到不完整响应消息的客户端必须将该消息记录为不完整，这种情况可能发生在连接过早关闭或对假定的分块传输编码进行解码失败时。不完整响应的缓存要求在 [CACHING] 的 3.3 节中定义。

如果响应终止于首部部分的中间(在接收到空行之前)，并且状态码可能依赖首部字段来传达响应的完整含义，那么客户端不能假设已经传达了含义;客户端可能需要重复请求，以确定下一步执行什么操作。

如果没有接收到终止编码的零大小块，则使用分块传输编码的消息体是不完整的。如果接收到的消息体大小(以八位为单位)小于 Content-Length 指定的值，则使用有效 Content-Length 的消息是不完整的。一个既没有分块传输编码也没有 Content-Length 的响应会被关闭连接终止，如果头部部分完好地接收到，则被认为是完整的，除非底层连接指示了错误(例如，TLS 中的 "incomplete close" 将使响应不完整，如 9.8 节所述)。

## 9. Connection Management

HTTP 消息传递独立于底层的传输层或会话层连接协议。HTTP 只假设可靠的传输，即按顺序交付请求和相应的按顺序交付响应。将 HTTP 请求和响应结构映射到底层传输协议的数据单元超出了本规范的范围。

如 [HTTP] 的 7.3 节所述，用于HTTP交互的特定连接协议由客户端配置和目标 URI 确定。例如，"http" URI scheme ( [http] 的 4.2.1 节)表示一个默认的 TCP 基于 IP 的连接，默认的 TCP 端口是 80，但客户端可能被配置为通过其他连接、端口或协议使用代理。

HTTP 实现需要参与连接管理，包括维护当前连接的状态、建立新连接或重用现有连接、处理从连接上接收到的消息、检测连接失败，以及关闭每个连接。大多数客户端并行地维护多个连接，包括每个服务器端点的多个连接。大多数服务器的设计初衷是维护数千个并发连接，同时控制请求队列以实现合理使用并检测拒绝服务攻击。

### 9.1. Establishment

描述如何通过各种传输层或会话层协议建立连接超出了本规范的范围。每个 HTTP 连接映射到一个底层传输连接。

### 9.2. Associating a Response to a Request

HTTP/1.1 没有将给定的请求消息与其对应的一个或多个响应消息关联起来的请求标识符。因此，它依赖于响应到达的顺序与同一个连接上的请求的顺序完全对应。只有在一个或多个信息性响应 (1xx;参见 [HTTP] 第 15.2 节)，然后对同一个请求进行最终响应。

在连接上有多个未完成请求的客户端必须按照发送的顺序维护一个未完成请求的列表，并且必须将该连接上收到的每个响应消息与尚未收到最终(非 1xx )响应的第一个未完成请求关联起来。

如果客户端从一个没有未完成请求的连接上接收到数据，那么客户端不能将该数据视为有效的响应;客户端应该关闭连接，因为消息的定界现在是不明确的，除非数据只包含一个或多个 CRLF (可以根据2.2节丢弃)。

### 9.3. Persistence

HTTP/1.1 默认使用 "持久连接"，允许多个请求和响应通过单个连接传递。HTTP 实现应该支持持久连接。

接收方根据最近收到的消息中的协议版本和连接头字段([HTTP]的7.6.1节)来确定连接是否持久:

- 如果 "close" 连接选项存在(章节9.6)，连接将不会在当前响应之后持续;
- 如果接收到的协议是 HTTP/1.1(或更高版本)，连接将在当前响应之后持续;
- 如果接收到的协议是 HTTP/1.0，则存在 "keep-alive" 连接选项，无论接收方不是代理，还是消息是响应，并且接收方希望遵守 HTTP/1.0 的 "keep-alive" 机制，连接将在当前响应之后持续;
- 不满足前面两个条件，连接将在当前响应之后关闭;
- 不支持持久连接的客户端必须在每个请求消息中发送 "close" 连接选项。

不支持持久连接的服务器必须在每个没有 1xx(Informational) 状态码的响应消息中发送 "close" 连接选项。

客户端可以在持久连接上发送额外的请求，直到它发送或接收到 "close" 连接选项，或接收到没有 "keep-alive" 选项的 HTTP/1.0 响应。

为了保持持久化，连接上的所有消息都需要有一个自定义的消息长度(即连接关闭时没有定义的长度)，如第6节所述。服务器必须读取整个请求消息体，或者在发送响应后关闭连接;否则，持久连接上的剩余数据将被误解为下一个请求。同样，如果客户端打算在后续请求中重用同一个连接，则必须读取整个响应消息体。

代理服务器不能与 HTTP/1.0 客户端保持持久连接(有关许多 HTTP/1.0 客户端实现的 Keep-Alive 首部字段问题的信息和讨论，请参见附录 C.2.2)。

有关与 HTTP/1.0 客户端向后兼容的更多信息，请参阅附录 C.2.2 。

#### 9.3.1. Retrying Requests

连接可以在任何时候关闭，有意或无意。实现应该预见到需要从异步关闭事件中恢复。客户端可以自动重试未完成请求序列的条件在 [HTTP] 的 9.2.2 节中定义。

#### 9.3.2. Pipelining

支持持久连接的客户端可以将请求 "pipeline" (即发送多个请求而不等待每个响应)。如果一系列管道请求都有安全方法([HTTP]的9.2.1节)，服务器可以并行处理这些请求，但它必须以接收请求的相同顺序发送相应的响应。

管道请求的客户端，如果在接收到所有相应响应之前连接关闭，则应该重试未应答的请求。当在一个失败的连接(服务器在最后一个完整的响应中没有显式关闭连接)之后重新尝试管道请求时，客户端不能在连接建立后立即管道，因为前一个管道中剩余的第一个请求可能会导致错误响应，如果在一个过早关闭的连接上发送多个请求，这个错误可能会再次丢失(参见 9.6 节描述的 TCP 复位问题)。

幂等方法([HTTP] 9.2.2节)对管道很重要，因为它们可以在连接失败后自动重试。用户代理不应该在非幂等方法之后对请求进行管道处理，直到接收到该方法的最终响应状态码，除非用户代理有方法检测和从涉及管道序列的部分故障条件中恢复。

接收管道请求的中介可能会在入站转发这些请求时对它们进行管道处理，因为它可以依赖出站用户代理来确定哪些请求可以安全地进行管道处理。如果入站连接在接收到响应之前失败，管道中的中介体可能会尝试重试尚未接收到响应的一系列请求(如果这些请求都具有幂等方法);否则，管道中介应该转发任何接收到的响应，然后关闭相应的出站连接，以便出站用户代理可以相应地恢复。

### 9.4. Concurrency

客户端应该限制与给定服务器同时打开的连接数。

之前的 HTTP 版本给出了一个特定的连接数上限，但这对于许多应用程序来说是不切实际的。因此，这个规范没有规定最大连接数，而是鼓励客户端在打开多个连接时保持保守。

使用多个连接通常是为了避免“队首阻塞”问题。在这种情况下，如果一个请求需要服务器端进行大量处理或传输非常大的内容，则会阻塞同一连接上的后续请求。然而，每个连接都会消耗服务器资源。

此外，在拥塞的网络中使用多个连接会导致不良的副作用。在没有拥塞的网络中，使用大量连接也会产生副作用，因为它们的聚合和初始同步发送行为会导致拥塞，如果使用较少的并行连接，就不会出现这种情况。

请注意，服务器可能会拒绝它认为是滥用或具有拒绝服务攻击特征的流量，例如来自单个客户端的打开连接数量过多。

### 9.5. Failures and Timeouts

服务器通常会有一些超时值，超过这个值，服务器将不再保持不活动连接。代理服务器可能会把这个值设得更高，因为客户端很可能会通过同一个代理服务器建立更多连接。持久连接的使用对客户端或服务器的超时长度(或是否存在)没有要求。

希望超时的客户端或服务器应该对连接发出一个优雅的关闭。实现应该不断地监视打开的连接，以获取接收到的关闭信号，并在适当的时候对其作出响应，因为及时关闭连接的两端可以使已分配的系统资源被回收。

客户端、服务器或代理可以在任何时候关闭传输连接。例如，一个客户端可能已经开始发送一个新的请求，同时服务器已经决定关闭 “空闲” 连接。从服务器的角度来看，连接在空闲时被关闭，但从客户端角度来看，请求正在进行中。

服务器应该尽可能维持持久连接，并允许底层传输的流量控制机制解决临时过载问题，而不是期望客户端重试而终止连接。后一种技术会加剧网络拥塞或服务器负载。

发送消息体的客户端应该在传输请求时监视网络连接，以确保有错误响应。如果客户端看到一个响应，表明服务器不希望接收消息体，并且正在关闭连接，客户端应该立即停止发送消息体，并关闭自己的连接。

### 9.6. Tear-down

"close" 连接选项被定义为一个信号，表示发送者将在完成响应后关闭此连接。当发送方打算关闭连接时，应该发送一个包含 "close" 连接选项的连接头字段([HTTP]章节7.6.1)。例如,

```
Connection: close
```

请求头字段表示这是客户端在此连接上发送的最后一个请求，而在响应中，相同的字段表示服务器将在响应消息完成后关闭此连接。

请注意，字段名称 "close" 是保留的，因为使用该名称作为头字段可能与 "close" 连接选项冲突。

发送 "close" 连接选项的客户端必须在该连接上(在包含 "close" 的连接之后)不再发送进一步的请求，并且必须在读取与此请求对应的最终响应消息后关闭连接。

接收到 "close" 连接选项的服务器在向包含 "close" 连接选项的请求发送最终响应后，必须发起关闭连接(见下文)。服务器应该在对该连接的最终响应中发送 "close" 连接选项。服务器不能处理从该连接接收到的任何其他请求。

发送 "close" 连接选项的服务器必须在发送包含 "close" 连接选项的响应后启动关闭连接(见下文)。服务器不能处理从该连接接收到的任何其他请求。

客户端接收到 "close" 连接选项后，必须停止在该连接上发送请求，并在读取包含 "close" 连接选项的响应消息后关闭连接;如果在连接上发送了额外的管道请求，客户端不应该假设它们会被服务器处理。

如果服务器立即关闭 TCP 连接，客户端将无法读取最后的 HTTP 响应，这是一个重大的风险。如果服务器在完全关闭的连接上从客户端收到了额外的数据，比如客户端在收到服务器响应之前发送了另一个请求，服务器的TCP 栈将向客户端发送一个 reset 包;不幸的是，在客户端 HTTP 解析器读取和解释输入缓冲区之前，reset 包可能会擦除客户端的未确认输入缓冲区。

为了避免 TCP 重置问题，服务器通常分阶段关闭连接。首先，服务器执行半关闭操作，只关闭 读/写 连接的写端。然后，服务器继续读取连接，直到收到客户端对应的关闭请求，或者直到服务器确信自己的 TCP 栈已经收到客户端对包含服务器最后一次响应的分组的确认。最后，服务器完全关闭连接。

目前还不清楚重置问题是 TCP 独占的，还是在其他传输连接协议中也可能存在。

请注意，客户端处于半封闭状态的 TCP 连接并不会限定请求消息，也并不意味着客户端不再对响应感兴趣。一般来说，由于 HTTP/1.1 是独立于传输层的，所以在边界情况下不能依赖传输层信号。

### 9.7. TLS Connection Initiation

从概念上讲，HTTP/TLS 只是通过 TLS [TLS13] 保护的连接发送 HTTP 消息。

HTTP 客户端也充当 TLS 客户端。它在适当的端口上发起到服务器的连接，并发送 TLS ClientHello 以开始 TLS 握手。当 TLS 握手完成后，客户端可以发起第一个 HTTP 请求。所有 HTTP 数据都必须作为TLS "application" 发送，否则将被视为 HTTP 的普通连接(包括作为持久连接重用的可能性)。

### 9.8. TLS Connection Closure

TLS 使用在(无错误)连接关闭之前交换关闭警告来提供安全的连接关闭;参见[TLS13]章节6.1。当接收到有效的关闭警报时，实现可以确保该连接上不会接收到进一步的数据。

当一个实现知道它已经发送或接收了它关心的所有消息数据时，通常通过检测 HTTP 消息边界，它可能会通过发送一个关闭警告来生成一个“不完全关闭”，然后关闭连接，而没有等待从另一端接收相应的关闭警告。

不完整的关闭并不会对已经接收到的数据的安全性产生问题，但它可能表明后续的数据可能已经被截断。由于 TLS 不能直接感知 HTTP 消息分帧，因此有必要检查 HTTP 数据本身，以确定消息是否完整。对不完整消息的处理在第8节中定义。

当遇到不完整的关闭请求时，客户端应该将收到的所有请求视为已完成

1. as much data as specified in the Content-Length header field or
2. the terminal zero-length chunk (when Transfer-Encoding of chunked is used).

一个既没有分块传输编码也没有内容长度的响应只有在收到有效的关闭警告时才算完成。将不完整的消息视为完整的可能会使实现暴露给攻击。

当客户端检测到关闭操作不完全时，应该能够正常地恢复。

客户端在关闭连接之前必须发送关闭警告。不希望收到更多数据的客户端可以选择不等待服务器的关闭警报，直接关闭连接，从而在服务器端生成一个不完全关闭。

服务器应该准备好接收来自客户端的不完全关闭，因为客户端通常可以定位到服务器数据的结束位置。

服务器在关闭连接之前，必须尝试与客户端交换关闭警告。服务器可能会在发送关闭警告后关闭连接，从而在客户端生成一个不完全关闭。

## 10. Enclosing Messages as Data

### 10.1. Media Type message/http

"message/http" 媒体类型可以用来封装单个 http 请求或响应消息，前提是它遵守 MIME 对所有 "message" 类型关于行长度和编码的限制。由于行长限制，"message/http" 中的字段值可以使用行折叠(obs-fold)，如5.2节所述，通过多行传递字段值。 "message/http" 数据的接收方必须在使用消息时用一个或多个 SP 字符替换任何过时的行折叠。

```
Type name: message
Subtype name: http
Required parameters: N/A
Optional parameters: version, msgtype
    version: The HTTP-version number of the enclosed message (e.g., "1.1"). If not present, the version can be determined from the first line of the body.
    msgtype: The message type -- "request" or "response". If not present, the type can be determined from the first line of the body.
Encoding considerations: only "7bit", "8bit", or "binary" are permitted
Security considerations: see Section 11
Interoperability considerations: N/A
Published specification: RFC 9112 (see Section 10.1).
Applications that use this media type: N/A
Fragment identifier considerations: N/A
Additional information:
    Magic number(s): N/A
    Deprecated alias names for this type: N/A
    File extension(s): N/A
    Macintosh file type code(s): N/A
Person and email address to contact for further information: See Authors' Addresses section.
Intended usage: COMMON
Restrictions on usage: N/A
Author: See Authors' Addresses section.
Change controller: IESG
```

### 10.2. Media Type application/http

"application/http" 媒体类型可以用来封装一个或多个 http 请求或响应消息(不混合)的管道。

```
Type name: application
Subtype name: http
Required parameters: N/A
Optional parameters: version, msgtype
    version: The HTTP-version number of the enclosed messages (e.g., "1.1"). If not present, the version can be determined from the first line of the body.
    msgtype: The message type -- "request" or "response". If not present, the type can be determined from the first line of the body.
Encoding considerations: HTTP messages enclosed by this type are in "binary" format; use of an appropriate Content-Transfer-Encoding is required when transmitted via email.
Security considerations: see Section 11
Interoperability considerations: N/A
Published specification: RFC 9112 (see Section 10.2).
Applications that use this media type: N/A
Fragment identifier considerations: N/A
Additional information:
    Deprecated alias names for this type: N/A
    Magic number(s): N/A
    File extension(s): N/A
    Macintosh file type code(s): N/A
Person and email address to contact for further information: See Authors' Addresses section.
Intended usage: COMMON
Restrictions on usage: N/A
Author: See Authors' Addresses section.
Change controller: IESG
```
