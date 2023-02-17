# HTTP Caching

> 原文 [https://www.rfc-editor.org/rfc/rfc9111](https://www.rfc-editor.org/rfc/rfc9111)

## 摘要

超文本传输​​协议 (HTTP) 是用于分布式协作超文本信息系统的无状态应用程序级协议。本文档定义了 HTTP 缓存以及控制缓存行为或指示可缓存响应消息的相关标头字段。

## 1. Introduction

超文本传输​​协议 (HTTP) 是一种无状态应用程序级请求/响应协议，它使用可扩展语义和自描述消息与基于网络的超文本信息系统进行灵活交互。它通常用于分布式信息系统，其中使用响应缓存可以提高性能。本文档定义了与缓存和重用响应消息相关的 HTTP 方面。

HTTP “缓存” 是响应消息的本地存储和控制其中消息的存储、检索和删除的子系统。缓存存储可缓存的响应，以减少未来等效请求的响应时间和网络带宽消耗。任何客户端或服务器都可以使用缓存，但在充当隧道时不能使用（ [ HTTP ] 第 3.7 节）。

“共享缓存”是一种存储响应以供多个用户重用的缓存；共享缓存通常（但不总是）部署为中介的一部分。相比之下，“私有缓存” 专供单个用户使用；通常，它们被部署为用户代理的一个组件。

HTTP 缓存的目标是通过重用先前的响应消息来满足当前请求，从而显着提高性能。缓存认为存储的响应是 “新鲜的”，如 第 4.2 节中所定义的，如果它可以在没有“验证”的情况下被重用（检查原始服务器以查看缓存的响应是否对该请求仍然有效）。因此，每次缓存重新使用新响应时，都可以减少延迟和网络开销。当缓存的响应不新鲜时，如果验证可以更新它（第 4.3 节）或者如果源不可用（第 4.2.4 节），它可能仍然是可重用的。

本文档废弃了 [RFC7234] ，附录 B 中总结了更改。

### 1.1. Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when, they appear in all capitals, as shown here.

Section 2 of [HTTP] defines conformance criteria and contains considerations regarding error handling.

### 1.2. Syntax Notation

本规范使用[ RFC5234 ] 的增强巴科斯范式 (ABNF) 表示法 ，扩展了[ RFC7405 ]中定义的字符串中区分大小写的表示法。

它还使用[ HTTP ]的第 5.6.1 节中定义的列表扩展，允许使用 "#" 运算符（类似于 "*" 运算符表示重复的方式）来紧凑定义逗号分隔的列表。附录 A 显示了收集到的语法，其中所有列表运算符都扩展为标准 ABNF 表示法。

#### 1.2.1. Imported Rules

以下核心规则通过引用包含在[ RFC5234 ]附录 B.1中定义：DIGIT（十进制 0-9）。

[ HTTP ]定义了以下规则：

```
  HTTP-date     = <HTTP-date, see [HTTP], Section 5.6.7>
  OWS           = <OWS, see [HTTP], Section 5.6.3>
  field-name    = <field-name, see [HTTP], Section 5.1>
  quoted-string = <quoted-string, see [HTTP], Section 5.6.4>
  token         = <token, see [HTTP], Section 5.6.2>
```

#### 1.2.2. Delta Seconds

delta-seconds 规则指定一个非负整数，以秒为单位表示时间。

```
  delta-seconds  = 1*DIGIT
```

解析 delta-seconds 并将其转换为二进制形式的接收者应该使用至少 31 位非负整数范围的算术类型。如果缓存接收到的 delta-seconds 值大于它可以表示的最大整数，或者如果它的任何后续计算溢出，则缓存必须将该值视为 2147483648 (2^31) 或它可以方便地表示的最大正整数。

> 注意：这里的值 2147483648 是历史原因，代表无穷大（超过 68 年），不需要以二进制形式存储；如果发生任何溢出，实现可以将其生成为字符串，即使计算是使用无法直接表示该数字的算术类型执行的。这里重要的是检测到溢出，而不是在以后的计算中将其视为负值。

## 2. Overview of Cache Operation

## 3. Storing Responses in Caches

### 3.1. Storing Header and Trailer Fields

### 3.2. Updating Stored Header Fields

### 3.3. Storing Incomplete Responses

### 3.4. Combining Partial Content

### 3.5. Storing Responses from Caches

## 4. Constructing Responses from Caches

### 4.1. Calculating Cache Keys with the Vary Header Field

### 4.2. Freshness

#### 4.2.1. Calculating Freshness Lifetime

#### 4.2.2. Calculating Heuristic Freshness

#### 4.2.3. Calculating Age

#### 4.2.4. Serving Stale Responses

### 4.3. Validation

#### 4.3.1. Sending a Validating Request

#### 4.3.2. Handling a Received Validation Request

#### 4.3.3. Handling a Validation Response

#### 4.3.4. Freshening Stored Responseds upon Validation

#### 4.3.5. Freshening Responses with HEAD

### 4.4. Invalidating Stored Responses

## 5. Field Deinitions

### 5.1. Age

### 5.2. Cache-Control

#### 5.2.1. Request Directives

##### 5.2.1.1. max-age

##### 5.2.1.2. max-stale

##### 5.2.1.3. min-fresh

##### 5.2.1.4. no-cache

##### 5.2.1.5. no-store

##### 5.2.1.6. no-transform

##### 5.2.1.7. only-if-cached

#### 5.2.2. Response Directives

##### 5.2.2.1. max-age

##### 5.2.2.2. must-revalidate

##### 5.2.2.3. must-understand

##### 5.2.2.4. no-cache

##### 5.2.2.5. no-store

##### 5.2.2.6. no-transform

##### 5.2.2.7. private

##### 5.2.2.8. proxy-revalidate

##### 5.2.2.9. public

##### 5.2.2.10. s-maxage

#### 5.2.3. Extension Directives

#### 5.2.4. Cache Directive Registry

### 5.3. Expires

### 5.4. Pragma

### 5.5. Warning

## 6. Relationship to Applications and Other Caches
