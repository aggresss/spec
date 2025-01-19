# The Messaging Layer Security (MLS) Protocol

> 原文 [https://datatracker.ietf.org/doc/html/rfc9420](https://datatracker.ietf.org/doc/html/rfc9420)

## 摘要

消息传递应用程序越来越多地采用端到端安全机制，以确保消息仅可由通信端点访问，而参与消息传递的任何服务器均无法访问。在群聊设置中，建立密钥以提供此类保护颇具挑战性，因为群聊中需要有两个以上的客户端就密钥达成一致，但这些客户端可能不同时在线。在本文档中，我们详细阐述了一种密钥建立协议，该协议为规模从两人到数千人的群组提供高效的异步群组密钥建立，并具有前向保密(FS)和后妥协安全(PCS)功能。

## Status of This Memo

This is an Internet Standards Track document.

This document is a product of the Internet Engineering Task Force (IETF). It represents the consensus of the IETF community. It has received public review and has been approved for publication by the Internet Engineering Steering Group (IESG). Further information on Internet Standards is available in Section 2 of RFC 7841.

Information about the current status of this document, any errata, and how to provide feedback on it may be obtained at https://www.rfc-editor.org/info/rfc9420.

## Copyright Notice

Copyright (c) 2023 IETF Trust and the persons identified as the document authors. All rights reserved.

This document is subject to BCP 78 and the IETF Trust's Legal Provisions Relating to IETF Documents (https://trustee.ietf.org/license-info) in effect on the date of publication of this document. Please review these documents carefully, as they describe your rights and restrictions with respect to this document. Code Components extracted from this document must include Revised BSD License text as described in Section 4.e of the Trust Legal Provisions and are provided without warranty as described in the Revised BSD License.

## 1 介绍

想要相互发送加密消息的一组用户需要一种派生共享对称加密密钥的方法。对于双方来说，这个问题已经得到了深入的研究，双棘轮(Double Ratchet) 作为一种通用的解决方案 [DoubleRatchet] [Signal] 出现。实现双棘轮的通道享有细粒度的前向保密以及妥协后的安全性，但仍然足够高效，可以在低带宽网络上大量使用。对于大于 2 的组，常见的策略是在现有的 1:1 安全通道上分发对称的 “发送方密钥”，然后让每个成员向使用自己的发送方密钥加密的组发送消息。一方面，使用发送方密钥提高了相对于单个消息的成对传输的效率，并且它提供了前向保密(通过添加哈希棘轮)。另一方面，使用发送方密钥很难实现泄露后的安全性，需要大量密钥更新消息，这些消息按组大小的平方进行缩放。获知发送方密钥的对手通常可以无限期地、被动地窃听该成员的消息。生成和分发新的发送方密钥为该发送方提供了一种妥协后的安全形式。然而，它需要计算和通信资源，这些资源与组的大小成线性关系。在本文中，我们描述了一个基于树结构的协议，该协议支持异步组密钥，具有前向保密和后妥协安全性。基于早期对“异步棘轮树”[ART]的研究，本文提出的协议使用树形结构的异步密钥封装机制。此机制允许组的成员派生和更新共享密钥，其成本按组大小的日志进行缩放。

## 2. Terminology

本文档中的关键词“MUST”，“MUST NOT”，“REQUIRED”，“SHALL”，“SHALL NOT”，“SHOULD”，“SHOULD NOT”，“RECOMMENDED”，“NOT RECOMMENDED”，“MAY”和“OPTIONAL”应按照BCP 14 [RFC2119] [RFC8174]中描述的解释，当且仅当它们以大写字母出现时，如下所示。

- **客户端(Client)**：使用此协议与其他客户端建立共享加密状态的代理。客户端是由它持有的加密密钥定义的。
- **组(Group)**：组表示在任何给定时间共享公共秘密值的客户端的逻辑集合。它的状态表示为一个线性序列的时代，其中每个时代依赖于它的前身。
- **时代(Epoch)**：组的一种状态，其中一组特定的经过身份验证的客户端拥有共享的加密状态。
- **成员(Member)**：包含在组的共享状态中的客户端，因此可以访问组的秘密。
- **密钥包(Key Package)**：描述客户端身份和功能的签名对象，包括可用于向该客户端加密的混合公钥加密(HPKE) [RFC9180]公钥。其他客户机可以使用客户机的KeyPackage将客户机引入新组。
- **组上下文(Group Context)**：汇总组的共享公共状态的对象。组上下文通常分布在签名的GroupInfo消息中，该消息提供给新成员以帮助他们加入组。
- **签名密钥(Signature Key)**：用于对消息发送方进行身份验证的签名密钥对。
- **提议(Proposal)**：提议对组进行更改的消息，例如，添加或删除成员。
- **提交(Commit)**：实现对一组提案中提议的组的更改的消息。
- **公共消息(PublicMessage)**：一个MLS协议消息，由其发送方签名，并被验证为来自特定时代的组成员，但未加密。
- **私有消息(PrivateMessage)**：由发送方签名的MLS协议消息，经过身份验证，它来自特定时代的组成员，并经过加密，使其对该时代的组成员保密。
- **握手消息(Handshake Message)**：一个 PublicMessage 或 PrivateMessage，携带 MLS Proposal 或 Commit 对象，而不是应用程序数据。
- **应用消息(Application Message)**：携带应用数据的私有消息。

第 4.1 节描述了特定于树计算的术语。通常，对称值可互换地称为 “key” 或 “secret”。这两个术语都表示必须对客户端保密的值。在标记单个值时，我们通常使用 “secret” 来指用于派生进一步的秘密值的值，而使用 “key” 来指与散列消息认证码(HMAC)或带关联数据的身份验证加密(AEAD)算法等算法一起使用的值。PublicMessage 和 PrivateMessage 格式在第 6 节中定义。前向保密和后妥协安全等安全概念在第 16 节中定义。如 13.5 节所述，MLS 使用 “生成随机扩展和维持可扩展性”(GREASE)方法来维持可扩展性，其中发送方在字段中插入随机值，接收方需要忽略未知值。为此目的，特定的 “GREASE values” 在相应的 IANA 注册中心进行注册。

### 2.1. Presentation Language

表示语言我们使用 TLS 表示语言 [RFC8446] 来描述协议消息的结构。除了基本语法之外，我们还添加了两个额外的特性：字段可选的能力和向量具有可变长度头的能力。

#### 2.1.1. Optional Value

一个可选的值是用一个状态信号八位字节编码的，如果存在，后面跟着值本身。在解码时，具有非 0 或 1 值的存在八字节必须被视为畸形而拒绝。

```mls
struct {
    uint8 present;
    select (present) {
        case 0: struct{};
        case 1: T value;
    };
} optional<T>;
```

在 TLS 表示语言中，向量被编码为以长度为前缀的编码元素序列。length 字段通过指定元素编码序列的最小和最大长度来设置固定的大小。在 MLS 中，有几个向量的大小在很大范围内变化。因此，我们不是使用固定长度的字段，而是使用基于 [RFC9000] 第 16 节中描述的可变长度整数编码来使用可变长度的字段。它们的不同之处在于这里的一个需要最小大小的编码。矢量描述不呈现最小值和最大值，而是简单地包含一个 v。例如：

```mls
struct {
    uint32 fixed<0..255>;
    opaque variable<V>;
} StructWithVectors;
```

这样的向量可以表示长度从 0 字节到 230 字节的值。变长整数编码保留第一个字节的两个最高位，以字节为单位对整数编码长度的以 2 为基数的对数进行编码。整数值在剩余的位上编码，因此整个值是网络字节顺序。编码后的值必须使用最小位数来表示该值。在解码时，使用比必要更多位的值必须被视为畸形。这意味着整数以 1、2 或 4 字节编码，可以分别编码 6 位、14 位或 30 位值。

Prefix | Length | Usable | Bits | Min | Max
--|--|--|--|--|--
00 | 1 | 6 | 0 | 63
01 | 2 | 14 | 64 | 16383
10 | 4 | 30 | 16384 | 1073741823
11 | invalid | - | - | -

Table 1: Summary of Integer Encodings

以“11”开头的向量是无效的，必须被拒绝。

例如：
- 4字节长度值 `0x9d7f3e7d` 解码为 `494878333`。
- 两字节长度值 `0x7bbd` 解码为 `15293`。
- 单字节长度值 `0x25` 解码为 `37`。

下图改编了 [RFC9000] 中提供的伪代码，增加了对最小长度编码的检查：

```mls
ReadVarint(data):
  // The length of variable-length integers is encoded in the
  // first two bits of the first byte.
  v = data.next_byte()
  prefix = v >> 6
  if prefix == 3:
    raise Exception('invalid variable length integer prefix')

  length = 1 << prefix

  // Once the length is known, remove these bits and read any
  // remaining bytes.
  v = v & 0x3f
  repeat length-1 times:
    v = (v << 8) + data.next_byte()

  // Check if the value would fit in half the provided length.
  if prefix >= 1 && v < (1 << (8*(length/2) - 2)):
    raise Exception('minimum encoding was not used')

  return v
```

对向量长度使用可变大小的整数可以使向量变得非常大，最大可达 230 字节。实现时应注意不要让 vector 溢出可用存储空间。为了方便调试潜在的互操作性问题，实现应该在发生这种溢出情况时提供一个明确的错误。

## 3. Protocol Overview

MLS 设计用于在 [MLS-arch] 中描述的上下文中操作。特别地，我们假设提供了以下服务：

- 身份验证服务(Authentication Service, AS)，它允许组成员对其他组成员提供的凭据进行身份验证。
- 在协议参与者之间路由 MLS 消息的传递服务(DS)。

MLS 假定一个受信任的 AS，但一个很大程度上不受信任的 DS。第 16.10 节描述了自治系统妥协或错误行为的影响。MLS 旨在保护集团数据的机密性和完整性，即使面对受损的 DS；通常，DS 只期望可靠地传递消息。第 16.9 节描述了 DS 的妥协或不当行为的影响。

MLS 的核心功能是连续的组认证密钥交换(authenticated key exchange - AKE)。与其他经过身份验证的密钥交换协议(如 TLS)一样，协议中的参与者就公共秘密值达成一致，并且每个参与者都可以验证其他参与者的身份。然后可以使用该秘密来保护使用 MLS 框架层从组中的一个参与者发送到其他参与者的消息，或者可以导出以与其他协议一起使用。MLS 提供的组 AKE 是指协议中可以有两个以上的参与者，而连续组 AKE 是指协议中的参与者集可以随时间变化。

MLS 的核心组织原则是分组和时代。组表示在任何给定时间共享公共秘密值的客户端的逻辑集合。一个群体的历史被划分为一个线性序列的时代。在每个 epoch 中，一组经过身份验证的成员就一个只有该 epoch 中的组成员知道的 epoch 秘密达成一致。组中涉及的成员集可以从一个 epoch 更改到下一个 epoch，并且 MLS 确保只有当前 epoch 中的成员才能访问 epoch secret。从 epoch 秘密中，成员派生出用于消息加密、组成员身份验证等的进一步共享秘密。

MLS 组的创建者单方面创建组的第一个 epoch，没有协议交互。之后，小组成员通过交换 MLS 消息将他们共享的加密状态从一个时代推进到另一个时代。

- KeyPackage 对象描述客户端的功能，并提供可用于将客户端添加到组的密钥。
- Proposal 消息建议在下一个 epoch 中进行更改，例如添加或删除成员。
- Commit 消息通过指示组的成员实现一组建议来开启一个新的时代。
- Welcome 消息向组提供新成员，并提供初始化其状态的信息，以便将其添加到组中，或者将自己添加到组中。

KeyPackage 和 Welcome 消息用于启动组或介绍新成员，因此它们在组成员和尚未在组中的客户端之间交换。客户机通过 DS 发布一个 KeyPackage，从而使其他客户机能够将其添加到组中。当组成员希望向组成员添加新成员时，它使用新成员的 KeyPackage 来添加新成员，并构造一个 Welcome 消息，新成员可以使用该消息初始化其本地状态。提议和提交消息从组中的一个成员发送到其他成员。MLS 为在组内发送消息提供了一个通用的框架层：PublicMessage 为未加密的 Proposal 和 Commit 消息提供发送方身份验证。PrivateMessage 为 Proposal/Commit 消息以及任何应用程序数据提供加密和身份验证。

### 3.1 Cryptographic State and Evolution

MLS 核心的加密状态分为三个责任领域：

```
                          .-    ...    -.
                         |               |
                         |       |       |
                         |       |       | Key Schedule
                         |       V       |
                         |  epoch_secret |
.                        |       |       |                             .
|\ Ratchet               |       |       |                     Secret /|
| \ Tree                 |       |       |                      Tree / |
|  \                     |       |       |                          /  |
|   \                    |       V       |                         /   |
|    +--> commit_secret --> epoch_secret --> encryption_secret -->+    |
|   /                    |       |       |                         \   |
|  /                     |       |       |                          \  |
| /                      |       |       |                           \ |
|/                       |       |       |                            \|
'                        |       V       |                             '
                         |  epoch_secret |
                         |       |       |
                         |       |       |
                         |       V       |
                         |               |
                          '-    ...    -'

              Figure 1: Overview of MLS Group Evolution
```

- 棘轮树代表群组成员身份，为群组成员提供相互验证的方法，并有效地加密发送给群组子集的消息。每个时期都有不同的棘轮树。它为密钥计划提供 seeds。
- 密钥计划表，描述了从一个时期进展到另一个时期的密钥派生链（主要使用 init_secret 和 epoch_secret），以及各种其他密钥的派生（见表4）。例如：
  - 用于初始化该时期的秘密树的加密秘密。
  - 允许其他协议利用 MLS 作为通用认证组密钥交换的导出者秘密。
  - 成员可以使用它来证明其在组中的成员身份的恢复秘密，例如在创建子组或后继组时。
- 从密钥计划派生出的秘密树，代表群组成员用于加密和验证消息的共享秘密。每个时期都有一棵不同的秘密树。

组中的每个成员都维护着组状态这些组成部分的部分视图。MLS 消息用于初始化这些视图，并在组在各个时期之间转换时保持它们同步。

每个新纪元都以提交消息开始。提交指示组中的现有成员通过应用一组提案来更新他们对棘轮树的看法，并使用更新后的棘轮树将新熵分发给组。此新熵仅提供给新纪元中的成员，而不提供给已被删除的成员。因此，提交保持了纪元秘密对当前纪元的成员保密的属性。

对于每个向组添加一个或多个成员的提交，都有一个或多个相应的欢迎消息。每条欢迎消息都为新成员提供所需的信息，以初始化他们对密钥计划和棘轮树的看法，以便这些看法与该时期组内其他成员的看法保持一致。

### 3.2 Example Protocol Execution

群组生命期主要有三个操作：

- 添加成员，由当前成员发起；
- 更新代表树中成员的键；
- 移除成员

这些操作中的每一个都是通过发送相应类型的消息（添加/更新/删除）来“提议”的。但是，直到发送提交消息为组提供新的熵时，组的状态才会改变。在本节中，我们展示了每个提案都被立即提交，但在更高级的部署案例中，应用程序可能会收集多个提案，然后一次性提交所有提案。在下面的插图中，我们直接显示了 Proposal 和 Commit 消息，而实际上它们将被封装在 PublicMessage 或 PrivateMessage 对象中发送。

在初始化一个组之前，客户端将 KeyPackages 发布到 DS 提供的目录中（参见图 2）

```
                                                   Delivery Service
                                                            |
                                                  .--------' '--------.
                                                 |                     |
                                                                 Group
  A                B                C            Directory       Channel
  |                |                |                |              |
  | KeyPackageA    |                |                |              |
  +------------------------------------------------->|              |
  |                |                |                |              |
  |                | KeyPackageB    |                |              |
  |                +-------------------------------->|              |
  |                |                |                |              |
  |                |                | KeyPackageC    |              |
  |                |                +--------------->|              |
  |                |                |                |              |

    Figure 2: Clients A, B, and C publish KeyPackages to the directory
```

图 3 显示了如何使用这些预先发布的 KeyPackage 来创建组。当客户端 A 想要与客户端 B 和 C 建立一个组时，它首先初始化一个仅包含其自身的组状态，并下载 B 和 C 的 KeyPackage。对于每个成员，A 生成一个 Add 提议和一个 Commit 消息来添加该成员，然后将这两条消息广播到组。客户端 A 还会生成一条 Welcome 消息并将其直接发送给新成员（无需将其发送给组）。只有在 A 从 Delivery Service 收到 Commit 消息后，它才会更新其状态以反映新成员的加入

一旦 A 更新了其状态，新成员处理了 Welcome，并且任何其他组成员都处理了 Commit，他们都会拥有组状态的一致表示，包括只有组成员知道的组秘密。新成员将能够读取和向组发送新消息，但在他们被添加到组之前发送的消息将无法访问


```
                                                                  Group
   A              B              C          Directory            Channel
   |              |              |              |                   |
   |         KeyPackageB, KeyPackageC           |                   |
   |<-------------------------------------------+                   |
   |              |              |              |                   |
   |              |              |              | Add(A->AB)        |
   |              |              |              | Commit(Add)       |
   +--------------------------------------------------------------->|
   |              |              |              |                   |
   |  Welcome(B)  |              |              |                   |
   +------------->|              |              |                   |
   |              |              |              |                   |
   |              |              |              | Add(A->AB)        |
   |              |              |              | Commit(Add)       |
   |<---------------------------------------------------------------+
   |              |              |              |                   |
   |              |              |              |                   |
   |              |              |              | Add(AB->ABC)      |
   |              |              |              | Commit(Add)       |
   +--------------------------------------------------------------->|
   |              |              |              |                   |
   |              |  Welcome(C)  |              |                   |
   +---------------------------->|              |                   |
   |              |              |              |                   |
   |              |              |              | Add(AB->ABC)      |
   |              |              |              | Commit(Add)       |
   |<---------------------------------------------------------------+
   |              |<------------------------------------------------+
   |              |              |              |                   |

          Figure 3: Client A creates a group with clients B and C
```

后续添加群组成员的方式相同。群组中的任何成员都可以为新客户端下载 KeyPackage，广播当前群组将用来更新其状态的 Add 和 Commit 消息，并发送新客户端可用于初始化其状态并加入群组的 Welcome 消息。

为了加强消息的前向保密性和泄露后安全性，每个成员都会定期向组更新代表他们的密钥。成员通过发送提交（可能不包含任何提议）或发送由另一个成员提交的更新消息来执行此操作（参见图 4 ）。一旦组的其他成员处理了这些消息，攻击者就会知道组的秘密，因为他们已经泄露了树中发送者叶子对应的秘密。在图 4 所示的场景结束时，该组对 A 和 B 都具有泄露后安全性

只要群组处于活动状态，就应该定期发送更新消息，不更新的成员最终应该从群组中删除。应用程序需要确定更新之间的适当时间。由于发送更新的目的是主动限制妥协窗口，因此正确的频率通常是以小时或天为单位，而不是以毫秒为单位。例如，应用程序可以在成员在收到来自另一个成员的任何消息后每次发送应用程序消息时发送更新，或者如果没有发送应用程序消息，则每天发送更新。

MLS 架构建议 MLS 通过安全传输进行操作（参见[MLS-ARCH]的第 7.1 节）。此类传输协议通常会提供拥塞控制等功能，以管理使用 MLS 的应用程序对共享同一网络的其他应用程序的影响。应用程序应注意不要以会导致网络拥塞等问题的速率发送 MLS 消息，特别是当它们不遵循上述建议时（例如，直接通过 UDP 发送 MLS）

```
                                                             Group
   A              B     ...      Z          Directory        Channel
   |              |              |              |              |
   |              | Update(B)    |              |              |
   |              +------------------------------------------->|
   |              |              |              | Update(B)    |
   |<----------------------------------------------------------+
   |              |<-------------------------------------------+
   |              |              |<----------------------------+
   |              |              |              |              |
   | Commit(Upd)  |              |              |              |
   +---------------------------------------------------------->|
   |              |              |              | Commit(Upd)  |
   |<----------------------------------------------------------+
   |              |<-------------------------------------------+
   |              |              |<----------------------------+
   |              |              |              |              |

        Figure 4: Client B proposes to update its key, and client A
                            commits the proposal
```

成员以类似的方式从组中删除，如图5所示。组中的任何成员都可以发送删除提议，然后发送提交消息。提交消息为组内除被删除成员之外的所有成员提供新的熵。这个新的熵被添加到新纪元的纪元秘密中，因此被删除的成员不知道它。请注意，这并不一定意味着任何成员实际上都被允许驱逐其他成员；组可以在这些基本机制之上实施访问控制策略。

```
                                                             Group
   A              B     ...      Z          Directory       Channel
   |              |              |              |              |
   |              |              | Remove(B)    |              |
   |              |              | Commit(Rem)  |              |
   |              |              +---------------------------->|
   |              |              |              |              |
   |              |              |              | Remove(B)    |
   |              |              |              | Commit(Rem)  |
   |<----------------------------------------------------------+
   |              |<-------------------------------------------+
   |              |              |<----------------------------+
   |              |              |              |              |

             Figure 5: Client Z removes client B from the group
```

请注意，本节中的流程只是示例；应用程序可以以其他方式安排消息流。例如：

- 欢迎信息不一定需要直接发送给新加入者。由于它们对新加入者进行了加密，因此可以更广泛地分发，例如，如果应用程序只能访问该组的广播频道
- 提案消息不需要立即发送给所有组成员。它们需要在生成提交之前提供给提交者，并在处理提交之前提供给其他成员
- Commit 的发送者不必等待收到自己的 Commit 才能推进其状态。它只需要知道它的 Commit 将是该组应用的下一个 Commit，例如基于来自编排服务器的承诺

### 3.3. External Joins

除了基于 Welcome 的流程来向群组添加新成员之外，新成员还可以通过“外部提交”的方式加入。当现有成员没有新成员的 KeyPackage 时，可以使用此机制，例如，在“开放”群组中，新成员可以加入而无需征得现有成员的许可。

图 6 显示了外部加入的典型消息流。为了使新成员能够以这种方式加入组，组成员（A，B）发布一个 GroupInfo 对象，该对象包含该组的 GroupContext 以及可用于向组现有成员加密秘密的公钥。当新成员 Z 希望加入时，他们会下载 GroupInfo 对象并使用它来形成特殊形式的提交，将 Z 添加到组中（如第 12.4.3.2 节中所述）。组中的现有成员以类似于正常提交的方式处理此外部提交，前进到 Z 现在是组成员的新时期。

```
                                                             Group
   A              B              Z          Directory        Channel
   |              |              |              |              |
   | GroupInfo    |              |              |              |
   +------------------------------------------->|              |
   |              |              | GroupInfo    |              |
   |              |              |<-------------+              |
   |              |              |              |              |
   |              |              | Commit(ExtZ) |              |
   |              |              +---------------------------->|
   |              |              |              | Commit(ExtZ) |
   |<----------------------------------------------------------+
   |              |<-------------------------------------------+
   |              |              |<----------------------------+
   |              |              |              |              |

       Figure 6: Client A publishes a GroupInfo object, and Client Z
                         uses it to join the group
```

### 3.4. Relationships between Epochs

一个组具有单一线性的时期序列。组和时期通常彼此独立。但是，有时在组内或跨组以加密方式链接时期会很有用。MLS 从每个时期派生出一个恢复预共享密钥 (PSK)，以允许将从一个时期提取的熵注入到未来的时期。希望注入 PSK 的组成员发出 PreSharedKey 提案（第 12.1.4 节），描述要注入的 PSK。提交此提案后，相应的 PSK 将纳入密钥计划，如 8.4 节所述。

以这种方式链接时期可保证进入新时期的成员就密钥达成一致当且仅当他们是提取恢复密钥的时期内的组成员

MLS 支持两种将新组与现有组绑定的方式，如图7和图8所示。重新初始化会关闭一个组并创建一个由相同成员和不同参数组成的新组。分支会启动一个新组，该组由原始组的参与者子集组成（对原始组没有影响）。在这两种情况下，新组都通过恢复 PSK 链接到旧组



```
   epoch_A_[n-1]
        |
        |
        |<-- ReInit
        |
        V
   epoch_A_[n]           epoch_B_[0]
        .                     |
        .  PSK(usage=reinit)  |
        .....................>|
                              |
                              V
                         epoch_B_[1]

                      Figure 7: Reinitializing a Group

   epoch_A_[n]           epoch_B_[0]
        |                     |
        |  PSK(usage=branch)  |
        |....................>|
        |                     |
        V                     V
   epoch_A_[n+1]         epoch_B_[1]

                        Figure 8: Branching a Group
```

应用程序还可以选择使用恢复 PSK 以其他方式链接 epoch。例如，图 9展示了一个来自 epoch 的恢复 PSKn被注入 epoch 的情况n+k。这表明 epoch 的组成员n+k也是 epoch 的成员n，无论这些成员的密钥因更新或提交而发生任何变化。

```
   epoch_A_[n]
        |
        |  PSK(usage=application)
        |.....................
        |                    .
        |                    .
       ...                  ...
        |                    .
        |                    .
        V                    .
   epoch_A_[n+k-1]           .
        |                    .
        |                    .
        |<....................
        |
        V
   epoch_A_[n+k]

            Figure 9: Reinjecting Entropy from an Earlier Epoch
```

## 4.  Ratchet Tree Concepts

该协议使用“棘轮树”来获取一组客户端之间的共享秘密。棘轮树是组成员之间秘密和密钥对的一种安排，其方式允许秘密高效更新以反映组中的变化

棘轮树允许群组通过将新熵加密到群组的子集来有效地移除任何成员。棘轮树将共享密钥分配给整个群组的子组，因此，例如，对群组中除一名成员以外的所有成员进行加密只需要log(N)对子树进行加密，而不是N-1 对每个参与者单独进行加密（其中 N 是群组中的成员数）

此移除操作允许 MLS 高效地实现后妥协安全性。在更新提议或完整提交消息中，成员的旧（可能已妥协）表示被高效地从组中删除，并用新生成的实例替换

### 4.1.  Ratchet Tree Terminology

#### 4.1.1.  Ratchet Tree Nodes


```
                  ...
                  /
                 _
           ______|______
          /             \
         X[B]            _
       __|__           __|__
      /     \         /     \
     _       _       Y       _
    / \     / \     / \     / \
   A   B   _   D   E   F   _   H

   0   1   2   3   4   5   6   7

  Figure 10: A Tree with Blanks and Unmerged Leaves
```

#### 4.1.2.  Paths through a Ratchet Tree

```
                 W = root
                 |
           .-----+-----.
          /             \
         _=U             Y
         |               |
       .-+-.           .-+-.
      /     \         /     \
     T       _=V     X       _=Z
    / \     / \     / \     / \
   A   B   _   _   E   F   G   _=H

   0   1   2   3   4   5   6   7

  Figure 11: A Complete Tree with Five Members, with Labels for Blank Parent Nodes
```

### 4.2.  Views of a Ratchet Tree

```
            Public Tree
   ============================
               pk(ABCD)
             /          \
       pk(AB)            _
        / \             / \
   pk(A)   pk(B)   pk(C)   pk(D)

    Private @ A       Private @ B       Private @ C       Private @ D
   =============     =============     =============     =============
        ABCD              ABCD              ABCD              ABCD
       /   \             /   \             /   \             /   \
     AB      _         AB      _         ?       _         ?       _
    / \     / \       / \     / \       / \     / \       / \     / \
   A   ?   ?   ?     ?   B   ?   ?     ?   ?   C   ?     ?   ?   ?   D

```
