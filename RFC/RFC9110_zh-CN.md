# HTTP Semantics

> 原文 [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)

## 摘要

超文本传输协议 (HTTP) 是一种用于分布式、协作的超文本信息系统的无状态应用层协议。本文档描述了 HTTP 的整体架构，建立了通用术语，并从协议中所有版本共享的层面进行定义。在这个定义中包括核心协议元素、可扩展性机制以及 "http" 和 "https" 统一资源标识符 (URI) 方案。

本文档更新了RFC 3864，废止了 RFC 2818、7231、7232、7233、7235、7538、7615、7694 和 7230 的部分内容。

## 1. Introduction

### 1.1. Purpose

### 1.2. History and Evolution

### 1.3. Core Semantics

### 1.4. Specifications Obsoleted by This Document

## 2. Conformance

### 2.1. Syntax Notation

### 2.2. Requirements Notation

### 2.3. Length Requirements

### 2.4. Error Handing

### 2.5. Protocol Version

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


