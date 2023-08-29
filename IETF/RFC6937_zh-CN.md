# Proportional Rate Reduction for TCP

> 原文 [https://datatracker.ietf.org/doc/html/rfc6937](https://datatracker.ietf.org/doc/html/rfc6937)

## 摘要

本文档描述了一种实验性比例速率降低 (PRR) 算法，作为广泛部署的快速恢复和速率减半算法的替代方案。 这些算法决定了丢失恢复期间 TCP 发送的数据量。 PRR 最大限度地减少了过多的窗口调整，并且恢复结束时的实际窗口大小将尽可能接近 ssthresh，如拥塞控制算法所确定的。

## 1. Introduction

本文档描述了一种实验性算法 PRR，用于提高丢失恢复期间 TCP 发送数据量的准确性。

标准拥塞控制 [RFC5681] 要求 TCP（和其他协议）减少其拥塞窗口 (cwnd) 以响应丢失。 同一文档中描述的快速恢复是进行此调整的参考算法。 其既定目标是通过在恢复期间返回 ACK 来恢复 TCP 的自身时钟，以将更多数据计时到网络中。 快速恢复通常通过在发送任何数据之前等待 ACK 的二分之一往返时间 (RTT) 过去来调整窗口。 它是脆弱的，因为它无法补偿损失本身造成的隐性窗口减少。

RFC 6675 [RFC6675] 通过计算 “管道”（发送方对网络中尚未完成的字节数的估计），使选择性确认 (SACK) [RFC2018] 的快速恢复更加准确。 通过 RFC 6675，快速恢复是通过在每个 ACK 上根据需要发送数据来实现的，以防止管道低于慢启动阈值 (ssthresh)，窗口大小由拥塞控制算法确定。 在许多存在严重丢失的情况下，这可以保护快速恢复免于超时，但如果数据或 ACK 窗口的整个后半部分丢失，则不会。 然而，携带意味着大量丢失数据的 SACK 选项的单个 ACK 可能导致管道估计器中的步骤不连续，这可能导致快速重传发送数据突发。

速率减半算法在恢复期间通过备用 ACK 发送数据，以便在 1 个 RTT 后窗口减半。 速率减半是在非正式发布 [RHweb] 之后在 Linux 中实现的，其中包括一个未完成的文档 [RHID]。 速率减半也不能充分补偿由损失引起的隐式窗口减少，并假设净窗口减少 50%，这在编写时完全是标准的，但不适合现代拥塞控制算法，例如 CUBIC [CUBIC ]，将窗口减少了不到 50%。 因此，利率减半通常会导致窗口进一步下降，从而降低性能并增加超时风险（如果存在额外损失）。

PRR 避免了这些过多的窗口调整，以便在恢复结束时，实际窗口大小将尽可能接近 ssthresh，即由拥塞控制算法确定的窗口大小。 它按照速率减半进行模式化，但使用适合拥塞控制算法选择的目标窗口的分数。 在 PRR 期间，两个额外的缩减边界算法之一会由于所有机制（包括瞬时应用程序停顿和损失本身）而限制总窗口缩减。

我们描述了两种略有不同的 Reduction Bound 算法：Conservative Reduction Bound（CRB），它是严格的数据包保守； Slow Start Reduction Bound (SSRB)，它比 CRB 更激进，每个 ACK 最多 1 个段。 PRR-CRB 满足附录 A 中描述的强数据包保护界限； 然而，在实际网络中，它的性能不如 RFC 6675 中描述的算法，事实证明 RFC 6675 在大量情况下更具攻击性。 在某些情况下，SSRB 提供了一种折衷方案，允许 TCP 相对于 CRB 为每个 ACK 发送 1 个附加段。 尽管 SSRB 不如 RFC 6675 激进（传输较少的段或花费更多的时间来传输它们），但由于恢复期间发生额外丢失的可能性较低，它的性能优于 RFC 6675。

PRR 和两个缩减界限所基于的强数据包守恒界限是根据 Van Jacobson 的数据包守恒原则构建的：传送到接收器的数据段用作时钟，以触发将相同数量的数据段发送回网络。 PRR 和 Reduction Bound 算法尽可能依赖于这个自时钟过程，并且仅受到其他估计器（例如 pipeline [RFC6675] 和 cwnd）的准确性的轻微影响。 这使得算法在出现导致其他估计器不确定的事件时具有精确性。

数据包保护原则的原始定义 [Jacobson88] 将假定丢失的数据包（例如，标记为重传候选者）视为已离开网络。 这个想法反映在 RFC 6675 中定义的管道估计器中并在此处使用，但它与附录 A 中描述的强数据包保护界限不同，后者仅根据到达接收器的数据进行定义。

我们在配套论文 [IMC11] 中提出的大规模测量研究中评估了这些算法和其他算法，并在第 5 节中进行了总结。该测量研究基于 RFC 3517 [RFC3517]，该标准已被 RFC 6675 取代。 虽然两个规范之间存在细微差别，而且我们对 RFC 3517 的实施非常谨慎，但我们不愿意无条件断言我们的测量结果适用于 RFC 6675，尽管我们相信情况确实如此。 相反，我们选择迂腐地描述相对于 RFC 3517 的测量结果，它们实际上是基于 RFC 3517 的。 算法及其属性的一般讨论已更新为参考 RFC 6675。

我们发现，对于真实的网络流量，PRR-SSRB 的性能优于 RFC 3517 和 Linux 速率减半，尽管它不如 RFC 3517 激进。我们相信这些结果也适用于 RFC 6675。

这些算法被描述为对 RFC 5681 [RFC5681]“TCP 拥塞控制”的修改，使用从管道算法 [RFC6675] 中得出的概念。 它们是最准确的，并且更容易使用 SACK [RFC2018] 实现，但不需要 SACK。

## 2. Definitions

以下术语、参数和状态变量按照早期文档中的定义使用：

- RFC 793: snd.una (send unacknowledged)
- RFC 5681: duplicate ACK, FlightSize, Sender Maximum Segment Size (SMSS)
- RFC 6675: covered (as in "covered sequence numbers")
- Voluntary window reductions: 选择不发送数据来响应某些ACK，目的是减少发送窗口大小和数据速率

我们定义一些额外的变量：

- **SACKd**：记分板指示已传送到接收器的字节总数。 这可以通过扫描记分板并计算所有 SACK 块覆盖的字节总数来计算。 如果未使用 SACK，则不定义 SACKd。
- **DeliveredData**：当前 ACK 指示已传送到接收方的总字节数。 当不处于恢复状态时，DeliveredData 是 snd.una 中的更改。 使用 SACK，DeliveredData 可以精确地计算为 snd.una 中的变化，加上 SACKd 中的（有符号）变化。 在没有 SACK 的恢复中，重复确认时，DeliveredData 估计为 1 SMSS，而在后续的部分或完整 ACK 上，DeliveredData 估计为 snd.una 中的更改，每个先前的重复 ACK 减去 1 SMSS。

请注意，DeliveredData 是稳健的； 对于使用 SACK 的 TCP，只需检查返回的 ACK，就可以在网络中的任何位置精确计算 DeliveredData。 丢失 ACK 的后果是稍后的 ACK 将显示更大的 DeliveredData。 此外，对于任何 TCP（有或没有 SACK），DeliveredData 的总和必须与相同时间间隔内的转发进度一致。

我们引入一个局部变量 “sndcnt”，它准确指示应该发送多少字节来响应每个 ACK。 请注意，发送哪些数据的决定（例如，重新传输丢失的数据或发送更多新数据）超出了本文档的范围。

## 3. Algorithms

在恢复开始时，初始化PRR状态。 这假设采用现代拥塞控制算法 CongCtrlAlg()，该算法可能将 ssthresh 设置为 FlightSize/2 以外的值：

```
   ssthresh = CongCtrlAlg()  // Target cwnd after recovery
   prr_delivered = 0         // Total bytes delivered during recovery
   prr_out = 0               // Total bytes sent during recovery
   RecoverFS = snd.nxt-snd.una // FlightSize at the start of recovery
```

在恢复计算期间的每个 ACK 上：

```
   DeliveredData = change_in(snd.una) + change_in(SACKd)
   prr_delivered += DeliveredData
   pipe = (RFC 6675 pipe algorithm)
   if (pipe > ssthresh) {
      // Proportional Rate Reduction
      sndcnt = CEIL(prr_delivered * ssthresh / RecoverFS) - prr_out
   } else {
      // Two versions of the Reduction Bound
      if (conservative) {    // PRR-CRB
        limit = prr_delivered - prr_out
      } else {               // PRR-SSRB
        limit = MAX(prr_delivered - prr_out, DeliveredData) + MSS
      }
      // Attempt to catch up, as permitted by limit
      sndcnt = MIN(ssthresh - pipe, limit)
   }

```

关于任何数据传输或重传：

```
   prr_out += (data sent) // strictly less than or equal to sndcnt
```

### 3.1. Examples

我们通过展示这些算法在两种情况下的不同行为来说明这些算法：TCP 经历单次丢失或突发 15 个连续丢失。 在所有情况下，我们都假设批量数据（无应用程序暂停）、标准加法增加乘法减少 (AIMD) 拥塞控制，并且 cwnd = FlightSize = pipeline = 20 段，因此 ssthresh 在恢复开始时将设置为 10。 我们还假设标准快速重传和有限传输 [RFC3042]，因此 TCP 将发送 2 个新段，然后发送 1 个重传，以响应丢失后的前 3 个重复 ACK。

下面的每个图表显示了当第 0 个段丢失时，各种恢复算法对第一次往返的每个 ACK 响应。 最上面一行表示触发 ACK 的传输段编号，X 表示丢失的段。 “cwnd”和“pipe”表示处理每个返回的ACK后这些算法的值。 “已发送”表示将发送多少“新”或“重新”传输的数据。 请注意，决定发送哪些数据的算法超出了本文档的范围。

当存在单个丢失时，使用任一 Reduction Bound 算法的 PRR 具有相同的行为。 我们显示“RB”，一个标志，指示哪个归约边界子表达式最终确定了 sndcnt 的值。 当损失最小时，“limit”（两种算法）将始终大于 ssthresh - 管道，因此 sndcnt 将是 ssthresh - 管道，由“RB”行中的“s”表示。

```
   RFC 6675
   ack#   X  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19
   cwnd:    20 20 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11 11
   pipe:    19 19 18 18 17 16 15 14 13 12 11 10 10 10 10 10 10 10 10
   sent:     N  N  R                          N  N  N  N  N  N  N  N


   Rate-Halving (Linux)
   ack#   X  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19
   cwnd:    20 20 19 18 18 17 17 16 16 15 15 14 14 13 13 12 12 11 11
   pipe:    19 19 18 18 17 17 16 16 15 15 14 14 13 13 12 12 11 11 10
   sent:     N  N  R     N     N     N     N     N     N     N     N


   PRR
   ack#   X  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19
   pipe:    19 19 18 18 18 17 17 16 16 15 15 14 14 13 13 12 12 11 10
   sent:     N  N  R     N     N     N     N     N     N        N  N
   RB:                                                          s  s

       Cwnd is not shown because PRR does not use it.

   Key for RB
   s: sndcnt = ssthresh - pipe                 // from ssthresh
   b: sndcnt = prr_delivered - prr_out + SMSS  // from banked
   d: sndcnt = DeliveredData + SMSS            // from DeliveredData
   (Sometimes, more than one applies.)
```

请注意，所有 3 种算法发送的数据总量相同。 RFC 6675 经历了“半沉默窗口”，而速率减半和 PRR 将自愿窗口减少扩展到整个 RTT。

接下来，我们考虑前 15 个数据包（0-14）丢失时的相同初始条件。 在有损 RTT 的剩余时间内，仅向发送方返回 5 个 ACK。 我们依次检查这些算法。

```
   RFC 6675
   ack#   X  X  X  X  X  X  X  X  X  X  X  X  X  X  X 15 16 17 18 19
   cwnd:                                              20 20 11 11 11
   pipe:                                              19 19  4 10 10
   sent:                                               N  N 7R  R  R

   Rate-Halving (Linux)
   ack#   X  X  X  X  X  X  X  X  X  X  X  X  X  X  X 15 16 17 18 19
   cwnd:                                              20 20  5  5  5
   pipe:                                              19 19  4  4  4
   sent:                                               N  N  R  R  R

   PRR-CRB
   ack#   X  X  X  X  X  X  X  X  X  X  X  X  X  X  X 15 16 17 18 19
   pipe:                                              19 19  4  4  4
   sent:                                               N  N  R  R  R
   RB:                                                       b  b  b

   PRR-SSRB
   ack#   X  X  X  X  X  X  X  X  X  X  X  X  X  X  X 15 16 17 18 19
   pipe:                                              19 19  4  5  6
   sent:                                               N  N 2R 2R 2R
   RB:                                                      bd  d  d
```

在这种特定情况下，RFC 6675 更加激进，因为一旦触发快速重传（在段 17 的 ACK 上），TCP 会立即重传足够的数据以使管道达到 cwnd。 我们的测量数据（参见第 5 节）表明 RFC 6675 显着优于 Rate-Halving、PRR-CRB 和我们测试的其他一些类似的保守算法，这表明实际损失超过由 拥塞控制算法。

速率减半的 Linux 实现包括保守缩减边界 [RHweb] 的早期版本。 在这种情况下，5 个 ACK 每个恰好触发 1 次传输（2 个新数据，3 个旧数据），并且 cwnd 设置为 5。在窗口大小为 5 时，需要 3 个往返才能重新传输所有 15 个丢失的分段。 速率减半在恢复期间根本不会提高窗口，因此当恢复最终完成时，TCP 会缓慢启动 cwnd，从 5 到 10。在本示例中，TCP 在拥塞控制选择的窗口的一半上运行超过 3 个时间。 RTT，增加经过的时间，并在出现额外丢失的情况下使其超时。

PRR-CRB 实施保守的减少界限。 由于总损耗使管道低于 ssthresh，因此发送数据时，传输的总数据 prr_out 遵循返回 ACK 报告的传送到接收器的总数据。 传输由发送限制控制，该限制设置为 prr_delivered - prr_out。 这由图中的 RB:b 标记指示。 在这种情况下，PRR-CRB面临着与Rate-Halving完全相同的问题； 过多的窗口减少会导致恢复损失所需的时间过长，并导致额外的超时。

PRR-SSRB 将每个 ACK 的窗口正好增加 1 个段，直到恢复期间管道上升到 ssthresh。 这是通过将限制设置为比在该 ACK 上报告的已传送到接收器的数据大 1 来实现的，在恢复期间实现慢启动，并由图中的 RB:d 标记指示。 尽管在恢复期间增加窗口似乎是不明智的，但重要的是要记住，这实际上比 RFC 5681 允许的攻击性要小，RFC 5681 发送与单个突发相同数量的附加数据，以响应触发快速重传的 ACK 。

对于不太极端的事件，其中总损失小于 FlightSize 和 ssthresh 之间的差异，PRR-CRB 和 PRR-SSRB 具有相同的行为。

## 4. Properties

除另有说明外，以下属性是 PRR-CRB 和 PRR-SSRB 所共有的：

PRR 在大多数恢复事件（包括突发丢失）中维护 TCP 的 ACK 时钟。 RFC 6675 可以在突发丢失后发送大量非时钟突发。

通常，PRR 会在整个 RTT 中均匀分布自愿窗口缩减。 这有可能普遍减少互联网流量的突发性，并且可以被视为一种软节奏。 假设，任何调速都会增加不同流交织的可能性，从而减少 ACK 压缩和其他增加流量突发现象的机会。 然而，这些影响尚未得到量化。

如果损失最小，PRR 将精确收敛到拥塞控制算法选择的目标窗口。 请注意，当 TCP 接近恢复结束时，prr_delivered 将接近 RecoverFS，并且将计算 sndcnt，以便 prr_out 接近 ssthresh。

由于恢复期间多次孤立的损失而导致隐式窗口减少，导致随后的自愿减少被跳过。 对于少量丢失，窗口大小恰好以拥塞控制算法选择的窗口结束。

对于突发丢失，可以通过发送额外的段来响应恢复期间稍后到达的 ACK，从而撤消早期的自愿窗口减少。 请注意，只要不撤消某些自愿窗口减少，pipe 的最终值将与 ssthresh（拥塞控制算法选择的目标 cwnd 值）相同。

具有任一缩减界限的 PRR 可以改善应用程序停顿时的情况，例如，当发送应用程序没有足够快地对数据进行排队或接收器停止推进 rwnd（接收器窗口）时。 当恢复期间出现应用程序停顿时，prr_out 将落后于 sndcnt 允许的传输总和。 由于停顿而错过的发送机会被视为银行自愿减少窗口； 具体来说，它们导致 prr_delivered - prr_out 显着为正。 如果应用程序在 TCP 仍在恢复中时赶上，TCP 将发送部分窗口突发以准确赶上应用程序从未停止的情况。 尽管这种突发可能被视为对网络造成影响，但这正是每次 RTT 应用程序部分停顿且未恢复时发生的情况。 我们已在所有州中统一部分 RTT 失速行为。 更改此行为超出了本文档的范围。

具有缩减界限的 PRR 对管道估计器中的错误不太敏感。 在恢复过程中，pipe 本质上是一个估计器，它使用不完整的信息来估计未 SACK 的段是否确实丢失或仅仅是网络中无序。 在某些情况下，管道可能会出现明显的误差； 例如，当过早地假定重新排序的数据突发丢失并标记为重传时，管道就会被低估。 如果传输像 RFC 6675 一样由管道直接调节，则管道估计器中的步骤不连续性会导致数据突发，一旦管道估计器在几个 ACK 后被纠正，就无法撤回数据突发。 对于 PRR，pipe 仅确定使用哪种算法（PRR 或 Reduction Bound）来根据 DeliveredData 计算 sndcnt。 虽然管道被低估，但算法每个 ACK 最多有 1 个段不同。 一旦管道被更新，它们就会在恢复结束时收敛到相同的最终窗口。

在恢复期间的所有条件和事件序列下，PRR-CRB 严格限制传输的数据等于或小于传送到接收器的数据量。 我们声称这种强数据包保护界限是最激进的算法，在某些环境中不会导致额外的强制丢失。 它具有这样的特性：如果在瓶颈处存在一个没有交叉流量的常设队列，则该队列将在恢复期间保持精确恒定的长度，除了由于数据包到达和退出时间差异而导致的 +1/-1 波动之外 。 有关此属性的详细讨论，请参阅附录 A。

尽管出于多种原因，强数据包保护界限非常有吸引力，但我们在第 5 节中总结的测量表明，它的攻击性较小，并且性能不如 RFC 6675，后者允许在出现突发丢失时突发数据。 PRR-SSRB 是一种折衷方案，与数据包保存界限相比，它允许 TCP 每个 ACK 发送 1 个额外的分段。 从严格的Packet Conserving Bound的角度来看，PRR-SSRB确实在恢复期间打开了窗口； 然而，在存在突发丢失的情况下，它的攻击性明显低于 RFC 6675。

## 5. Measurements

在另一篇 IMC11 论文 [IMC11] 中，我们描述了一些测量方法，比较了恢复期间减少窗口的各种策略。 这些实验是在承载 Google 生产流量的服务器上进行的，这里简要总结一下。

各种窗口缩减算法和广泛的检测均在 Linux 2.6 中实现。 我们使用了基本 Linux 实现中存在的统一算法集，包括 CUBIC [CUBIC]、有限传输 [RFC3042]、阈值传输（[FACK] 中的第 3.1 节）（该算法未出现在 RFC 3517 中，但有一个类似的算法 已添加到 RFC 6675），以及丢失重传检测算法。 我们确认，速率减半（Linux 默认值）、RFC 3517 和 PRR 的行为符合其各自的规范，并且性能和功能与生产使用中的内核相当。 所有不同的窗口缩减算法都存在于一个通用内核中，并且可以使用 sysctl 进行选择，这样我们就有了一个绝对统一的基线来比较它们。

我们的实验包括一个额外的算法，即无限制的 PRR (PRR-UB)，当管道低于 ssthresh 时，该算法会发送 ssthresh-pipe 突发。 此行为与 RFC 3517 类似。

此配置的一个重要细节是，CUBIC 仅将窗口减少了 30%，而传统拥塞控制算法则减少了 50%。 这加剧了 RFC 3517 和 PRR-UB 在触发快速重传时发送突发的趋势，因为管道可能已经低于 ssthresh。 在 32% 的恢复事件中观察到了这种情况：在触发快速重传之前管道降至 ssthresh 以下，因此各种 PRR 算法在缩减绑定阶段启动，并且 RFC 3517 通过快速重传发送了段突发。

在配套论文中，我们观察到 PRR-SSRB 在所有测试的算法中花费的恢复时间最少，这主要是因为一旦恢复，它就会经历更少的超时。

与 PRR-SSRB 相比，RFC 3517 检测到的丢失重传多出 29%，超时多出 2.6%（可能是由于未检测到的丢失重传）。 这些结果代表了 PRR-UB 和其他在管道低于 ssthresh 时发送突发的算法。

速率减半在恢复结束时会经历 5% 的超时时间和显着更小的最终 cwnd 值。 较小的 cwnd 有时会导致恢复本身需要额外的往返次数。 这些结果代表了 PRR-CRB 和在恢复期间实现严格数据包保护的其他算法。

## 6. Conclusion and Recommendations

尽管出于多种原因，PRR-CRB 中使用的强数据包保护界限非常有吸引力，但我们的测量表明，它的攻击性较小，并且性能不如 RFC 3517（并暗示 RFC 6675），后者允许数据突发 当出现突发性损失时。 RFC 3517 和 RFC 6675 在 Van Jacobson 数据包守恒原则的原始意义上是保守的，其中包括假定丢失的数据段确实已离开网络的假设。 PRR-CRB 没有做出这样的假设，而是遵循强数据包保护界限，其中只有实际到达接收器的数据包才被视为已离开网络。 PRR-SSRB 是一种折衷方案，允许 TCP 相对于强数据包保护界限为每个 ACK 发送 1 个额外分段，以部分补偿多余的丢失。

从强数据包保守界的角度来看，PRR-SSRB确实在恢复过程中打开了窗口； 然而，在存在突发丢失的情况下，它的攻击性明显低于 RFC 3517（和 RFC 6675）。 即便如此，它的性能通常优于 RFC 3517（大概还有 RFC 6675），因为它避免了突发造成的一些自造成损失。

目前，我们认为没有理由不大规模测试和部署 PRR-SSRB。 担心恢复期间提高窗口的任何潜在影响的实施者可能希望选择支持 PRR-CRB（实际上更容易实施）进行比较研究。 此外，PRR 的一个小细节可以通过用总管道替换管道来改进，如 Laminar TCP [Laminar] 所定义。

关于术语的最后一点评论：我们预计常见用法将从算法名称中删除“慢启动减少界限”。 该文件需要迂腐地规定 PRR 和归约界限的每个变体都有不同的名称。 然而，我们预计未来不会对替代减少界限进行任何探索。

## 7. Acknowledgements

## 8. Security Considerations

## 9. References

## 9.1. Normative References

## 9.2. Informative References

## Appendix A. Strong Packet Conservation Bound
