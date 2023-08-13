# CUBIC for Fast Long-Distance Networks

> 原文 [https://datatracker.ietf.org/doc/html/rfc8312](https://datatracker.ietf.org/doc/html/rfc8312)

## 摘要

CUBIC 是当前 TCP 标准的扩展。 它与当前的 TCP 标准的区别仅在于发送端的拥塞控制算法。 特别是，它使用三次函数代替当前 TCP 标准的线性窗口增加函数，以提高快速和长距离网络下的可扩展性和稳定性。 CUBIC 及其前身算法已被 Linux 默认采用并已使用多年。 本文档提供了 CUBIC 规范，以支持第三方实现并通过 CUBIC 性能实验来征求社区反馈。

## 1. Introduction

TCP 在快速长距离网络中的低利用率问题在 [K03] 和 [RFC3649] 中有详细记录。 此问题是由于在具有大带宽延迟积 (BDP) 的网络中发生拥塞事件后拥塞窗口缓慢增加而引起的。 [HKLRX06]表明，即使在数百个数据包的拥塞窗口大小范围内，也经常观察到此问题。 这个问题同样适用于所有 Reno 风格的 TCP 标准及其变体，包括 TCP-RENO [RFC5681]、TCP-NewReno [RFC6582] [RFC6675]、SCTP [RFC4960] 和 TFRC [RFC5348]，它们使用相同的线性 用于窗口增长的 increase 函数，我们下面将其统称为 “标准TCP”。

CUBIC 最初在 [HRX08] 中提出，是对标准 TCP 拥塞控制算法的修改，以解决这个问题。 本文档描述了 CUBIC 的最新规范。 具体来说，CUBIC使用三次函数代替标准TCP的线性窗口增加函数，以提高快速和长距离网络下的可扩展性和稳定性。

二进制递增拥塞控制 (BIC-TCP) [XHR04] 是 CUBIC 的前身，于 2005 年被 Linux 选为默认的 TCP 拥塞控制算法，并已被整个互联网社区使用了数年。 CUBIC 使用与 BIC-TCP 类似的窗口增加功能，其设计目的是在带宽使用方面比 BIC-TCP 更不激进且更公平，同时保持 BIC-TCP 的优势，如稳定性、窗口可扩展性和 RTT 公平性。 CUBIC已经取代BIC-TCP成为Linux中默认的TCP拥塞控制算法，并已被Linux在全球范围内部署。 通过在各种互联网场景中的广泛测试，我们相信CUBIC在全球互联网上的测试和部署是安全的。

在接下来的章节中，我们首先简要解释 CUBIC 的设计原理，然后提供 CUBIC 的准确规范，最后按照[RFC5033]中指定的指南讨论 CUBIC 的安全特性。

## 2. Conventions
## 3. Design Principles of CUBIC

CUBIC是根据以下设计原则设计的：

- 原则 1：为了更好的网络利用率和稳定性，CUBIC使用三次函数的凹凸轮廓来增加拥塞窗口大小，而不是仅仅使用凸函数。
- 原则 2：为了对 TCP 友好，CUBIC 被设计为在 RTT 短、带宽小的网络中表现得像标准 TCP，而标准 TCP 在这些网络中表现良好。
- 原则 3：为了 RTT 公平性，CUBIC 旨在实现不同 RTT 的流之间的线性带宽共享。
- 原则 4：CUBIC 适当地设置其乘性窗口减小因子，以在可扩展性和收敛速度之间取得平衡。

原则 1：为了更好的网络利用率和稳定性，CUBIC [HRX08]根据上次拥塞事件的经过时间使用三次窗口增加函数。 虽然标准 TCP 的大多数替代拥塞控制算法都使用凸函数来增加拥塞窗口，但 CUBIC 使用三次函数的凹凸轮廓来实现窗口增长。 在通过重复 ACK 或显式拥塞通知回显 (ECN-Echo) ACK [RFC3168] 检测到响应拥塞事件的窗口减小后，CUBIC 将获得拥塞事件的拥塞窗口大小注册为 W_max 并执行乘法减小 的拥塞窗口。 进入拥塞避免后，它开始使用三次函数的凹轮廓来增加拥塞窗口。 三次函数设置为在 W_max 处达到稳定状态，以便凹窗口继续增加，直到窗口大小变为 W_max。 之后，三次函数变成凸轮廓并且凸窗口开始增加。 这种窗口调整方式（先凹后凸）提高了算法稳定性，同时保持较高的网络利用率[CEHRX07]。 这是因为窗口大小几乎保持不变，在 W_max 周围形成一个平台，网络利用率被认为是最高的。 在稳定状态下，CUBIC的大多数窗口大小样本都接近W_max，从而提高了网络利用率和稳定性。 请注意，那些仅使用凸函数来增加拥塞窗口大小的拥塞控制算法在 W_max 附近具有最大增量，因此在网络饱和点附近引入大量数据包突发，可能导致频繁的全局丢失同步。

原则 2：CUBIC 提升每个流对标准 TCP 的公平性。 请注意，标准 TCP 在短 RTT 和小带宽（或小 BDP）网络下表现良好。 仅在具有长 RTT 和大带宽（或大 BDP）的网络中存在可扩展性问题。 标准 TCP 的替代拥塞控制算法设计为在每个流的基础上对标准 TCP 友好，必须在小型 BDP 网络中比在大型 BDP 网络中更不积极地增加其拥塞窗口。 CUBIC 的攻击性主要取决于窗口缩小前的最大窗口大小，在小型 BDP 网络中比在大型 BDP 网络中更小。 因此，CUBIC 在小型 BDP 网络中比在大型 BDP 网络中更不积极地增加其拥塞窗口。 此外，当 CUBIC 的三次函数比标准 TCP 更不积极地增加其拥塞窗口时，CUBIC 简单地遵循标准 TCP 的窗口大小，以确保 CUBIC 在小型 BDP 网络中至少实现与标准 TCP 相同的吞吐量。 我们将 CUBIC 的行为类似于标准 TCP 的区域称为“TCP 友好区域”。

原理 3：具有不同 RTT 的两个 CUBIC 流的吞吐量比与其 RTT 比的倒数成线性正比，其中流的吞吐量大约等于其拥塞窗口的大小除以其 RTT。 具体来说，CUBIC 保持与 TCP 友好区域之外的 RTT 无关的窗口增加率，因此当具有不同 RTT 的流在 TCP 友好区域之外运行时，在稳态下具有相似的拥塞窗口大小。 这种线性吞吐量比的概念类似于高统计复用环境下的标准 TCP，其中数据包丢失与各个流量无关。 然而，在低统计复用环境下，具有不同RTT的标准TCP流的吞吐量比与其RTT比的倒数成二次方比例[XHR04]。 CUBIC 始终确保线性吞吐量比与统计复用级别无关。 这是对标准 TCP 的改进。 虽然对于不同 RTT 流的特定吞吐量比率尚未达成共识，但我们认为，在有线互联网下，使用线性吞吐量比率似乎比相等吞吐量（即具有不同 RTT 的流的相同吞吐量）或更高阶的吞吐量更合理。 吞吐量比（例如，低统计复用环境下标准 TCP 的二次吞吐量比）。

原则 4：为了平衡可扩展性和收敛速度，CUBIC 将乘性窗口减小因子设置为 0.7，而标准 TCP 使用 0.5。 虽然这提高了 CUBIC 的可扩展性，但该决定的副作用是收敛速度较慢，尤其是在低统计复用环境下。 这种设计选择是根据作者的观察

高速 TCP (HSTCP) [RFC3649] 与其他研究人员（例如，[GV02]）一起提出：当前的互联网变得更加异步，通过高统计复用减少同步丢失的频率。 在这种环境下，即使是严格的乘增乘减（MIMD）也能收敛。 具有相同 RTT 的 CUBIC 流总是收敛到相同的吞吐量，与统计复用无关，从而实现算法内的公平性。 我们还发现，在具有足够统计复用的环境下，CUBIC流的收敛速度是合理的。

## 4. CUBIC Congestion Control

本文中所有窗口大小的单位是最大段大小（MSS）的段，所有时间的单位是秒。 设 cwnd 表示流的拥塞窗口大小，ssthresh 表示慢启动阈值。

### 4.1. Window Increase Function

CUBIC 通过仅在接收到 ACK 时增加拥塞窗口来维持标准 TCP 的确认 (ACK) 时钟。 它对TCP的快速恢复和重传没有做任何改变，例如TCP-NewReno[RFC6582][RFC6675]。 在拥塞事件（通过重复的 ACK 检测到数据包丢失或通过带有 ECN-Echo 标志的 ACK 检测到网络拥塞 [RFC3168]）后的拥塞避免过程中，CUBIC 更改了标准 TCP 的窗口增加功能。 假设W_max是在最后一次拥塞事件中窗口缩小之前的窗口大小。

CUBIC 使用以下窗口增加函数：

```
   W_cubic(t) = C*(t-K)^3 + W_max (Eq. 1)
```

其中C是一个常数，用于确定高BDP网络中窗口增加的积极性，t是从当前拥塞避免开始所经过的时间，K是上述函数将当前窗口大小增加到 W_max 如果没有进一步的拥塞事件，则使用以下公式计算：

```
   K = cubic_root(W_max*(1-beta_cubic)/C) (Eq. 2)
```

其中beta_cubic为CUBIC乘法减少因子，即当检测到拥塞事件时，CUBIC将其cwnd减少为 `W_cubic(0)=W_max*beta_cubic`。 我们在第 4.5 节讨论如何设置 beta_cubic，在第 5 节讨论如何设置 C。

在拥塞避免期间收到 ACK 后，CUBIC 使用等式 1 计算下一个 RTT 周期内的窗口增加率。 1、设置W_cubic(t+RTT)为拥塞窗口候选目标值，其中RTT为Standard TCP计算的加权平均RTT。

根据当前拥塞窗口大小cwnd的值，CUBIC以三种不同的模式运行。

1. TCP友好区域，确保CUBIC至少实现与标准TCP相同的吞吐量。
2. 凹区域，如果CUBIC不在TCP友好区域并且cwnd小于W_max。
3. 凸区域，如果CUBIC不在TCP友好区域并且cwnd大于W_max。

下面，我们描述了 CUBIC 在每个地区采取的具体行动。

### 4.2. TCP-Friendly Region

标准 TCP 在某些类型的网络中表现良好，例如在短 RTT 和小带宽（或小 BDP）网络下。 在这些网络中，我们使用 TCP 友好区域来确保 CUBIC 至少实现与标准 TCP 相同的吞吐量。

TCP友好区域是根据[FHP00]中描述的分析设计的。 该分析研究了加法增加和乘法减少 (AIMD) 算法的性能，该算法具有加法因子 alpha_aimd（每个 RTT 的分段数）和乘法因子 beta_aimd，表示为 AIMD(alpha_aimd, beta_aimd)。 具体来说，AIMD(alpha_aimd, beta_aimd)的平均拥塞窗口大小可以使用式(1)计算。 3. 分析表明，α_aimd=3*(1-beta_aimd)/(1+beta_aimd) 的 AIMD(alpha_aimd, beta_aimd) 实现了与使用 AIMD(1, 0.5) 的标准 TCP 相同的平均窗口大小。

```
    AVG_W_aimd = [ alpha_aimd * (1+beta_aimd) /
                   (2*(1-beta_aimd)*p) ]^0.5 (Eq. 3)
```

基于以上分析，CUBIC 采用式（1）。 4 估计 AIMD(alpha_aimd, beta_aimd) 的窗口大小 W_est，其中 alpha_aimd=3*(1-beta_cubic)/(1+beta_cubic) 和 beta_aimd=beta_cubic，实现与标准 TCP 相同的平均窗口大小。 当在拥塞避免中接收到ACK时（cwnd可以大于或小于W_max），CUBIC检查W_cubic(t)是否小于W_est(t)。 如果是这样，则 CUBIC 位于 TCP 友好区域，并且在每次收到 ACK 时应将 cwnd 设置为 W_est(t)。

```
    W_est(t) = W_max*beta_cubic +
                [3*(1-beta_cubic)/(1+beta_cubic)] * (t/RTT) (Eq. 4)
```

### 4.3. Concave Region

当在拥塞避免中接收到ACK时，如果CUBIC不在TCP友好区域并且cwnd小于W_max，则CUBIC在凹区域。 在此区域中，对于每个收到的 ACK，cwnd 必须增加 (W_cubic(t+RTT) - cwnd)/cwnd，其中 W_cubic(t+RTT) 使用等式 1 计算。 1.

### 4.4. Convex Region

当在拥塞避免中接收到ACK时，如果CUBIC不在TCP友好区域并且cwnd大于或等于W_max，则CUBIC在凸区域。 凸区域表示自上次拥塞事件以来网络状况可能已受到干扰，这可能意味着在某些流量离开后有更多可用带宽。 由于互联网是高度异步的，因此总是可能出现一定程度的扰动，而不会导致可用带宽发生重大变化。 在这个区域，CUBIC 非常小心，非常缓慢地增加其窗口大小。 凸形轮廓确保窗口在开始时增长非常缓慢，并逐渐增加其增长速率。 我们也将此区域称为“最大探测阶段”，因为 CUBIC 正在搜索新的 W_max。 在此区域中，对于每个收到的 ACK，cwnd 必须增加 (W_cubic(t+RTT) - cwnd)/cwnd，其中 W_cubic(t+RTT) 使用等式 1 计算。 1.

### 4.5. Multiplicative Decrease

当重复 ACK 检测到数据包丢失或 ECN-Echo ACK 检测到网络拥塞时，CUBIC 会按如下方式更新其 W_max、cwnd 和 ssthresh。 参数 beta_cubic 应设置为 0.7。

```
   W_max = cwnd;                 // save window size before reduction
   ssthresh = cwnd * beta_cubic; // new slow-start threshold
   ssthresh = max(ssthresh, 2);  // threshold is at least 2 MSS
   cwnd = cwnd * beta_cubic;     // window reduction
```

将 beta_cubic 设置为大于 0.5 的值的副作用是收敛速度较慢。 我们相信，虽然 beta_cubic 的适应性更强的设置可以导致更快的收敛，但它会使 CUBIC 的分析变得更加困难。 beta_cubic的这种自适应调整是CUBIC下一版本的一项。

### 4.6. Fast Convergence

为了提高CUBIC的收敛速度，我们在CUBIC中添加了启发式。 当新流加入网络时，如果现有流已经使用了网络的所有带宽，则网络中的现有流需要放弃一些带宽，以便为新流提供一定的增长空间。 为了加速现有流的带宽释放，应该实现以下称为“快速收敛”的机制。

通过快速收敛，当拥塞事件发生时，在拥塞窗口减小之前，流会在更新当前拥塞事件的 W_max 之前记住 W_max 的最后一个值。 我们将 W_max 的最后一个值称为 W_last_max。

```
   if (W_max < W_last_max){ // should we make room for others
       W_last_max = W_max;             // remember the last W_max
       W_max = W_max*(1.0+beta_cubic)/2.0; // further reduce W_max
   } else {
       W_last_max = W_max              // remember the last W_max
   }
```

在拥塞事件中，如果W_max的当前值小于W_last_max，这表明由于可用带宽的变化，该流所经历的饱和点正在减小。 然后我们通过进一步降低 W_max 来允许该流释放更多带宽。 此操作有效地延长了该流增加其拥塞窗口的时间，因为减少的 W_max 迫使该流更早达到平稳状态。 这允许新流有更多时间赶上其拥塞窗口大小。

快速收敛是针对具有多个 CUBIC 流的网络环境而设计的。 在只有单个 CUBIC 流且没有任何其他流量的网络环境中，应该禁用快速收敛。

### 4.7. Timeout

在超时的情况下，CUBIC遵循标准TCP来减少cwnd [RFC5681]，但使用与标准TCP [RFC5681]不同的beta_cubic（与第4.5节相同）设置ssthresh。

在超时后的第一次拥塞避免期间，CUBIC 使用等式增加其拥塞窗口大小。 1，其中t是从当前拥塞避免开始以来经过的时间，K设置为0，W_max设置为当前拥塞避免开始时的拥塞窗口大小。

### 4.8. Slow Start

当 cwnd 不超过 ssthresh 时，CUBIC 必须采用慢启动算法。 在慢启动算法中，CUBIC可以在一般网络中选择标准TCP慢启动[RFC5681]，或者在快速和长距离网络中选择有限慢启动[RFC3742]或混合慢启动[HR08]。

在 CUBIC 运行混合慢启动 [HR08] 的情况下，它可能会退出第一个慢启动而不会导致任何数据包丢失，因此 W_max 未定义。 在这种特殊情况下，CUBIC 切换到拥塞避免并使用等式 1 增加其拥塞窗口大小。 1，其中t是从当前拥塞避免开始以来经过的时间，K设置为0，W_max设置为当前拥塞避免开始时的拥塞窗口大小。

## 5. Discussion

在本节中，我们将按照 [RFC5033] 中指定的指南进一步讨论 CUBIC 的安全特性。

在确定性丢失模型中，两次连续数据包丢失之间的数据包数量始终为 1/p，CUBIC 始终以凹窗口轮廓运行，这大大简化了 CUBIC 的性能分析。 CUBIC的平均窗口大小可以通过以下函数获得：

```
    AVG_W_cubic = [C*(3+beta_cubic)/(4*(1-beta_cubic))]^0.25 *
                    (RTT^0.75) / (p^0.75) (Eq. 5)
```

当 beta_cubic 设置为 0.7 时，上式简化为：

```
    AVG_W_cubic = (C*3.7/1.2)^0.25 * (RTT^0.75) / (p^0.75) (Eq. 6)
```

我们将在下一小节中使用等式确定 C 的值。 6.

### 5.1. Fairness to Standard TCP

在标准 TCP 能够合理利用可用带宽的环境中，CUBIC 不会显着改变这种状态。

    标准 TCP 在以下两种类型的网络中表现良好：

    1. 带宽延迟积（BDP）较小的网络

    2. RTT 较短的网络，但 BDP 不一定较小

    CUBIC 的设计与上述两种网络中的标准 TCP 非常相似。 以下两个表显示了标准 TCP、HSTCP 和 CUBIC 的平均窗口大小。 标准 TCP 和 HSTCP 的平均窗口大小来自 [RFC3649]。 CUBIC 的平均窗口大小使用等式计算： 6 以及三个不同 C 值的 CUBIC TCP 友好区域。

```
   +--------+----------+-----------+------------+-----------+----------+
   |   Loss |  Average |   Average |      CUBIC |     CUBIC |    CUBIC |
   | Rate P |    TCP W |   HSTCP W |   (C=0.04) |   (C=0.4) |    (C=4) |
   +--------+----------+-----------+------------+-----------+----------+
   |  10^-2 |       12 |        12 |         12 |        12 |       12 |
   |  10^-3 |       38 |        38 |         38 |        38 |       59 |
   |  10^-4 |      120 |       263 |        120 |       187 |      333 |
   |  10^-5 |      379 |      1795 |        593 |      1054 |     1874 |
   |  10^-6 |     1200 |     12279 |       3332 |      5926 |    10538 |
   |  10^-7 |     3795 |     83981 |      18740 |     33325 |    59261 |
   |  10^-8 |    12000 |    574356 |     105383 |    187400 |   333250 |
   +--------+----------+-----------+------------+-----------+----------+

                                  Table 1

```

 表1描述了RTT = 0.1秒网络中标准TCP、HSTCP和CUBIC的响应函数。 平均窗口大小以 MSS 大小的段为单位。

```

   +--------+-----------+-----------+------------+-----------+---------+
   |   Loss |   Average |   Average |      CUBIC |     CUBIC |   CUBIC |
   | Rate P |     TCP W |   HSTCP W |   (C=0.04) |   (C=0.4) |   (C=4) |
   +--------+-----------+-----------+------------+-----------+---------+
   |  10^-2 |        12 |        12 |         12 |        12 |      12 |
   |  10^-3 |        38 |        38 |         38 |        38 |      38 |
   |  10^-4 |       120 |       263 |        120 |       120 |     120 |
   |  10^-5 |       379 |      1795 |        379 |       379 |     379 |
   |  10^-6 |      1200 |     12279 |       1200 |      1200 |    1874 |
   |  10^-7 |      3795 |     83981 |       3795 |      5926 |   10538 |
   |  10^-8 |     12000 |    574356 |      18740 |     33325 |   59261 |
   +--------+-----------+-----------+------------+-----------+---------+

                                  Table 2

```

### 5.2. Using Spare Capacity
### 5.3. Difficult Environments
### 5.4. Investigating a Range of Environments
### 5.5. Protection against Congestion Collapse
### 5.6. Fairness within the Alternative Congestion Control Algorithm
### 5.7. Performance with Misbehaving Nodes and Outside Attackers
### 5.8. Behavior for Application-Limited Flows
### 5.9. Responses to Sudden or Transient Events
### 5.10. Incremental Deployment
## 6. Security Considerations
## 7. IANA Considerations
## 8. References
### 8.1. Normative References
### 8.2. Informative References
