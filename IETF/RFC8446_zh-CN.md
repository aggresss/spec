# The Transport Layer Security (TLS) Protocol Version 1.3

> 原文 [https://datatracker.ietf.org/doc/html/rfc8446](https://datatracker.ietf.org/doc/html/rfc8446)

## 摘要

本文档指定了传输层安全性（TLS）协议的 1.3 版本。TLS 允许客户端/服务器应用程序以防止窃听、篡改和消息伪造的方式通过 Internet 进行通信。

本文档更新了 [RFC5705] 和 [RFC6066]，并废弃了 [RFC5077]、[RFC5246] 和 [RFC6961]。本文档还规定了 TLS 1.2 实现的新要求。

## 1. Introduction

TLS 的主要目标是为通信双方提供一个安全的通道；对下层传输的唯一要求是提供可靠保序的数据流。安全通道尤其应该提供如下属性：

- 认证（Authentication）：server 端应该总是需要认证的；client 端可以选择性的认证。认证可以通过非对称算法完成（如RSA、椭圆曲线数字签名算法(ECDSA)、或 Edwards曲线数字签名算法(EdDSA)）完成，或通过对称预共享密钥（PSK）。
- 保密（Confidentiality）：在建立好的通道上发送的数据只能终端可见。TLS不隐藏传输数据的长度，但终端可以填充TLS记录（pad）来隐藏真实长度，从而提升安全性。
- 完整性（Integrity）：在建立好的通道上发送的数据不能被攻击者修改（如果修改了一定会被发现）。

即使出现像 [RFC3552] 那样已经完全控制网络的攻击者，这些属性也应该保证。更完整的相关安全属性清单详见附录E。

TLS由两个主要部分构成：

- 握手协议（第4节），认证通信双方，协商加密模式和参数，并确定共享密钥。握手协议可以防止篡改，如果连接没有受到攻击，攻击者就不能强制两端协商不同的参数。
- 记录协议（第5节），使用由握手协议协商出的参数来保护通信双方的流量。记录协议将流量分为一系列记录，每个记录独立地使用流量密钥保护。

TLS是一个独立的应用协议；上层协议可以透明地运行于 TLS 之上。然而，TLS 标准并未指定怎样使用 TLS 保证安全、怎样发起 TLS 握手以及怎样理解认证证书交换，这些留给运行在 TLS 之上的协议的设计者和实现者来判断。

本文定义了 TLS 1.3. 虽然 TLS 1.3 与以前的版本不直接兼容，但所有 TLS 版本都包含一个版本控制机制，可以在客户端和服务器都支持的情况下协商出一个公共版本。

本文取代和废除了以前版本的 TLS，包括 1.2 版本 [RFC5246]。也废除了在 [RFC5077] 里面定义的 TLS ticket 机制，并用定义在第 2.2 节中的机制取代它。由于 TLS 1.3 改变了密钥的产生方式，它更新了 [RFC5705]。正如 7.5 节描述的那样。它也改变了在线证书状态协议（OCSP）消息的传输方式，因此更新了 [RFC6066]，废除了 [RFC6961]，如第 4.4.2.1 节所述。

### 1.1. Conventions and Terminology

使用了以下术语:

- 客户端（client）: 发起 TLS 连接的端点。
- 连接（connection）: 两端之间的传输层连接。
- 端点（endpoint）: 连接的客户端或者服务器。
- 握手（handshake）: 客户端和服务器之间协商 TLS 交互的后续参数的初始协商。
- 对端（peer）: 一个端点，当说到一个特定端点，对端指的是非当前讨论的端点。
- 接收端（receiver）: 接收记录的端点。
- 发送者（sender）: 发送记录的端点。
- 服务器（server）: 没有发起TLS连接的端点。

### 1.2. Major Differences from TLS 1.2

以下是 TLS 1.2 和 TLS 1.3 的主要差异。这并不是全部差异，还有很多次要的差别。

- 支持的对称算法列表已经删除了所有被认为是遗留问题的算法。列表保留了所有使用“关联数据的认证加密”（AEAD）算法。密码套件的概念已经改变，将认证和密钥交换机制与记录保护算法（包括密钥长度）和 Hash（用于秘钥导出函数和握手消息认证码 MAC）分离。
- 增加 0-RTT 模式，为一些应用数据在连接建立阶段节省了一次往返，这是以牺牲一定的安全特性为代价的。
- 静态 RSA 和 Diffie-Hellman 密码套件已经被删除；所有基于公钥的密钥交换算法现在都能提供前向安全。
- 所有 ServerHello 之后的握手消息现在都已经加密。扩展之前在 ServerHello 中以明文发送，新引入的 EncryptedExtensions 消息可以保证扩展以加密方式传输。
- 密钥导出函数被重新设计。新的设计使得密码学家能够通过改进的密钥分离特性进行更容易的分析。基于 HMAC 的提取-扩展密钥导出函数（HKDF）被用作一个基础的原始组件。
- Handshake 状态机进行了重大重构，以便更具一致性和删除多余的消息如 ChangeCipherSpec (除了中间件兼容性需要)。
- 椭圆曲线算法已经属于基本的规范，且包含了新的签名算法，如 EdDSA。TLS 1.3 删除了点格式协商以便于每个曲线使用单点格式。
- 其它的密码学改进包括改变 RSA 填充以使用 RSA 概率签名方案（RSASSA-PSS），删除压缩，数字签名算法 DSA，和定制 DHE 组（Ephemeral Diffie-Hellman）。
- 废弃了 TLS 1.2 的版本协商机制，以便在扩展中添加版本列表。这增加了不支持版本协商的 server 的兼容性。
- 之前版本中会话恢复（根据或不根据 server 端状态）和基于 PSK 的密码族已经被一个单独的新 PSK 交换所取代。
- 引用已更新至最新版本的 RFC（例如，[RFC5280] 而不是 [RFC3280]）。

### 1.3. Updates Affecting TLS 1.2

本文提出了几个可能影响 TLS 1.2 实现的变化，包括那些不支持 TLS 1.3 的实现：

- 4.1.3 节中的版本降级保护机制。
- 4.2.3 节中的 RSASSA-PSS 签名方案。
- ClientHello 中 “supported_versions” 扩展可以用于协商 TLS 使用的版本，优先于 ClientHello 中的 legacy_version 字段。
- "signature_algorithms_cert" 扩展允许一个 client 声明它使用哪种签名算法验证 X.509 证书。

此外，本文提出了支持 TLS 的早期版本需要实现的部分；见 9.3 节。

## 2. Protocol Overview

### 2.1. Incorrect DHE Share

### 2.2. Resumption and Pre-Shared Key (PSK)

### 2.3. 0-RTT Data

## 3. Presentation Language

### 3.1. Basic Block Size

### 3.2. Miscellaneous

### 3.3. Numbers

### 3.4. Vectors

### 3.5. Enumerateds

### 3.6. Constructed Types

### 3.7. Constants

### 3.8. Variants

## 4. Handshake Protocol

### 4.1. Key Exchange Messages

#### 4.1.1. Cryptographic Negotiation

#### 4.1.2. Client Hello

#### 4.1.3. Server Hello

#### 4.1.4. Hello Retry Request

### 4.2. Extensions

#### 4.2.1. Supported Versions

#### 4.2.2. Cookie

#### 4.2.3. Signature Algorithms

#### 4.2.4. Certificate Authorities

#### 4.2.5. OID Filters

#### 4.2.6. Post-Handshake Client Authentication

#### 4.2.7. Supported Groups

#### 4.2.8. Key Share

##### 4.2.8.1. Diffie-Hellman Parameters

##### 4.2.8.2. ECDHE Parameters

#### 4.2.9. Pre-Shared Key Exchange Modes

#### 4.2.10. Early Data Indication

#### 4.2.11. Pre-Shared Key Extension

##### 4.2.11.1. Ticket Age

##### 4.2.11.2. PSK Binder

##### 4.2.11.3. Processing Order

### 4.3. Server Parameters

#### 4.3.1. Encrypted Extensions

#### 4.3.2. Certificate Request

### 4.4. Authentication Messages

#### 4.4.1. The Transcript Hash

#### 4.4.2. Certificate

##### 4.4.2.1. OCSP Status and SCT Extensions

##### 4.4.2.2. Server Certificate Selection

##### 4.4.2.3. Client Certificate Selection

##### 4.4.2.4. Receiving a Certificate Message

#### 4.4.3. Certificate Verify

#### 4.4.4. Finished

### 4.5. End of Early Data

### 4.6. Post-Handshake Messages

#### 4.6.1. New Session Ticket Message

#### 4.6.2. Post-Handshake Authentication

#### 4.6.3. Key and Initialization Vector Update

## 5. Record Protocol

### 5.1. Record Layer

### 5.2. Record Payload Protection

### 5.3. Per-Record Nonce

### 5.4. Record Padding

### 5.5. Limits on Key Usage

## 6. Alert Protocol

### 6.1. Closure Alerts

### 6.2. Error Alerts

## 7. Cryptographic Computations

### 7.1. Key Schedule

### 7.2. Updating Traffic Secrets

### 7.3. Traffic Key Calculation

### 7.4. (EC)DHE Shared Secret Calculation

#### 7.4.1. Finite Field Diffie-Hellman

#### 7.4.2. Elliptic Curve Diffie-Hellman

### 7.5. Exporters

## 8. 0-RTT and Anti-Replay

### 8.1. Single-Use Tickets

### 8.2. Client Hello Recording

### 8.3. Freshness Checks

## 9. Compliance Requirements

### 9.1. Mandatory-to-Implement Cipher Suites

### 9.2. Mandatory-to-Implement Extensions

### 9.3. Protocol Invariants

## 10. Security Considerations

## 11. IANA Considerations

## 12. References

## Appendix A. State Machine

### A.1. Client

### A.2. Server

## Appendix B. Protocol Data Structures and Constant Values

### B.1. Record Layer

### B.2. Alert Messages

### B.3. Handshake Protocol

#### B.3.1. Key Exchange Messages

##### B.3.1.1. Version Extension

##### B.3.1.2. Cookie Extension

##### B.3.1.3. Signature Algorithm Extension

##### B.3.1.4. Supported Groups Extension

#### B.3.2. Server Parameters Messages

#### B.3.3. Authentication Messages

#### B.3.4. Ticket Establishment

#### B.3.5. Updating Keys

### B.4. Cipher Suites

## Appendix C. Implementation Notes

### C.1. Random Number Generation and Seeding

### C.2. Certificates and Authentication

### C.3. Implementation Pitfalls

### C.4. Client Tracking Prevention

### C.5. Unauthenticated Operation

## Appendix D. Backward Compatibility

### D.1. Negotiating with an Older Server

### D.2. Negotiating with an Older Client

### D.3. 0-RTT Backward Compatibility

### D.4. Middlebox Compatibility Mode

### D.5. Security Restrictions Related to Backward Compatibility

## Appendix E. Overview of Security Properties

### E.1. Handshake

#### E.1.1. Key Derivation and HKDF

#### E.1.2. Client Authentication

#### E.1.3. 0-RTT

#### E.1.4. Exporter Independence

#### E.1.5. Post-Compromise Security

#### E.1.6. External References

### E.2. Record Layer

#### E.2.1. External References

### E.3. Traffic Analysis

### E.4. Side-Channel Attacks

### E.5. Replay Attacks on 0-RTT

#### E.5.1. Replay and Exporters

### E.6. PSK Identity Exposure

### E.7. Sharing PSKs

### E.8. Attacks on Static RSA



