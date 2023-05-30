# Raft

## 1 Introduction

一致性算法允许一组机器作为一个连贯的群体工作，这个群体能够经受住一些成员的失败。正因为如此，它们在构建可靠的大型软件系统方面发挥着关键作用。Paxos 在过去十年中主导了共识算法的讨论，大多数共识的实现都是基于 Paxos 或受其影响，Paxos 已经成为教授学生共识的主要工具。

鉴于 Paxos 的难理解性，论文作者实现了 Raft 一致性算法来更易于理解的教育学生和更优秀的扩展性。

Raft 算法有三个最主要的特性：
- **Strong leader:** Raft 使用了比其他一致性算法更强大的领导形式。例如，日志条目仅从 leader 流向其他服务器。这简化了对 replicated log 的管理，并使 Raft 更容易理解。 
- **Leader election:** Raft 使用随机计时器来选举 leader 。这为任何一致性算法已经需要的 heartbeats 仅仅添加了少量的机制，同时简单而快速地解决冲突。
- **Membership changes:** Raft 改变集群中服务器集的机制使用了一种新的 *joint consensus* 方法，即在转换过程中两种不同配置的大部分重叠。这允许集群在配置更改期间继续正常运行。

## 2 Replicated state machines

在 Replicated state machines 这种方法中，服务器集合上的状态机可以计算到相同状态的相同副本，并且即使有些服务器停机，状态机也可以继续运行。所以在分布式系统下，Replicated state machines 通常用于解决大量的错误容忍问题。

![](../.gitbook/assets/Replicated%20state%20machine%20architecture.png)

Replicated state machines 通常使用 replicated log 实现，如图 1 所示。每个服务器存储一个包含一系列命令的日志，其状态机按顺序执行这些命令。每个日志以相同的顺序包含相同的命令，因此每个状态机处理相同的命令序列。由于状态机是确定的，因此每个状态机计算相同的状态和相同的输出序列。

## 3 What’s wrong with Paxos?

简单的用三个字概况：难理解。

## 4 Designing for understandability

我们在设计 Raft 时有几个目标: 
- 它必须为系统构建提供一个完整和实用的基础，以便大大减少开发人员所需的设计工作量; 
- 它必须在所有条件下都是安全的，在典型的操作条件下都是可用的; 
- 它必须对常规操作有效。但我们最重要的目标ーー也是最困难的挑战ーー是让人理解。大量的读者必须能够轻松地理解算法。
- 此外，必须有可能开发关于算法的直观性，以便系统构建者可以进行在现实世界实现中不可避免的扩展。

## 5 The Raft consensus algorithm

Raft 是一种用于管理第 2 节所述形式的 replicated log 的算法。图 2 概括了该算法的浓缩形式以供参考，图 3 列出了该算法的关键属性；这些数字的元素将在本节的其余部分逐一讨论。

![](../.gitbook/assets/A%20condensed%20summary%20of%20the%20Raft%20consensus%20algorithm.png)

![](../.gitbook/assets/Raft%20Protocol%20Summary.png)

![](../.gitbook/assets/Raft%20guarantees%20that%20each%20of%20these%20properties%20is%20true%20at%20all%20times.png)

Raft 通过首先选举一个杰出的 leader 来实现一致性算法，然后让 leader 完全负责管理 replicated log 。Leader 接受来自客户端的日志条目，将其复制到其他服务器上，并告诉服务器何时可以安全地将日志条目应用到他们的状态机。有一个 leader 可以简化对 replicated log 的管理。例如，leader 可以决定在日志中放置新条目的位置，而不需要咨询其他服务器，并且数据以一种简单的方式从 leader 流向其他服务器。一个 leader 可能会失败或与其他服务器断开连接，在这种情况下，会选出一个新的 leader。

鉴于领导者的方法，Raft 将一致性问题分解为三个相对独立的子问题，这将在下面的小节中讨论：
- **Leader election:** 当现有的 leader 失败时，必须选择一个新的 leader。
- **Log replication:** leader 必须接受来自客户端的日志条目，并在整个集群中复制它们，强制其他日志与其自己的日志一致。
- **Safety:** 如果任何服务器已将特定日志条目应用于其状态机，则任何其他服务器都不能对同一日志索引应用不同的命令。

### 5.1 Raft basics

Raft 集群包含多个服务器。5 是一个典型数字，它允许系统容忍两次故障。在任何给定时间，每个服务器都处于以下三种状态之一：leader，follower，or candidate。在正常操作中，只有一个 leader，所有其他服务器都是 follower。Follower 是被动的：他们自己不发出任何请求，而只是回应 leader 和 candidate 的请求。Leader 处理所有客户端请求（如果客户端联系 follower，则追随者将其重定向到 leader）。第三个状态，candidate，用于选举新的 leader。图 4 显示了状态及其转换;下面将讨论这些转换。

![](../.gitbook/assets/Server%20states.png)

Raft 将时间划分为任意长度的 terms，如图 5 所示。Terms 被编号成连续的整数。每个 term 都以选举开始，其中一个或多个候选人试图成为的 leader。如果 candidate 赢得选举，那么它将在剩余的任期内担任 leader。在某些情况下，选举将导致投票分裂。在这种情况下，任期将以没有 leader 结束;新 term（新选举）将很快开始。Raft 确保在给定 term 内最多有一个领导者。

![](../.gitbook/assets/Time%20is%20divided%20into%20terms.png)

不同的服务器可能会在不同的时间观察 term 之间的转换，在某些情况下，一个服务器可能不会观察一个选举甚至整个 term。Term 在 Raft 中充当了 logic clock，它们允许服务器检测过时的信息，如过时的 leader。每个服务器都存储一个 current term number，该编号随时间单调地增加。每当服务器进行通信时，就会交换当前 term；如果一个服务器的当前 term 比另一个服务器的小，那么它就将其当前 term 更新为较大的值。如果一个 candidate 或 leader 发现它的 term 已经过时，它将立即恢复到 follower 状态。如果一个服务器收到的请求是一个过时的 term，它将拒绝该请求。

Raft 服务器使用远程过程调用（RPC） 进行通信，基本一致性算法只需要两种类型的 RPCs。 RequestVote RPCs 由 candidate 在选举期间启动， AppendEntries RPCs 由 leader 启动，以 replicated log entries 并提供检测信号形式。第 7 节添加了第三个 RPC，用于在服务器之间传输快照。如果服务器未及时收到响应，则会重试 RPC，并并行发出 RPC 以获得最佳性能。

### 5.2 Leader election

Raft 使用 heartbeat 机制来触发 candidate 选举。当服务器启动时，它们作为 follower 开始。只要服务器从 leader 或 candidate 接收有效的 RPCs，它就会保持 follower 状态。Leader 定期向所有 follower 发送 heartbeats（不携带日志条目的 AppendEntries RPC），以维护其权限。如果 follower 在称为 *election out* 的时间段内没有收到任何通信，那么它就会假设没有可行的 leader，并开始选举以选择新的 leader。

为了开始一场选举，follower 增加其当前 term 并过渡到 candidate 状态。然后，它为自己投票，并与集群中的每个其他服务器并行发出 RequestVote RPCs。Candidate 继续保持这种状态，直到发生以下三种情况之一：
- （a）它赢得选举。
- （b）另一个服务器将自己确立为领导者。
- （c）一段时间过去了，没有获胜者。

> （a）如果 candidate 在同一 term 内从整个集群中的大多数服务器获得投票，则 candidate 将赢得选举。每个服务器在给定 term 内最多投票给一名 candidate，先到先得。多数规则确保最多一名 candidate 可以赢得特定 term 的选举（图 3 中的 Election Safety Property）。一旦 candidate 赢得选举，他就成为 leader。然后，它将 heartbeat 消息发送到所有其他服务器，以建立其权限并防止新的选举。
（b） 在等待投票时，candidate 可能会从另一个声称是 leader 的服务器收到 AppendEntries RPC。如果 leader 的 term（包含在 RPC 中）至少与 candidate 的当前 term 一样长，则 candidate 将 leader 视为合法并返回 follower 状态。如果 RPC 中的 term 小于 candidate 的当前 term，则 candidate 将拒绝 RPC 并继续处于 candidate 状态。
（c）第三种可能的结果是，candidate 既不赢得选举，也不输：如果许多 followers 同时成为 candidate，选票可能会被分割，这样没有 candidate 获得多数票。发生这种情况时，每个 candidate 都将超时并通过增加其 term 并启动另一轮 RequestVote RPC 来开始新的选举。然而，如果没有额外的措施，分裂投票可能会无限期地重复。

Raft 使用随机选举超时来确保分裂选票很少见，并且能够快速解决。为了首先防止分裂投票，选举超时是从固定间隔（例如，150-300ms）中随机选择的。这会分散服务器，以便在大多数情况下只有一台服务器超时;它赢得选举并在任何其他服务器超时之前发送检测信号。相同的机制用于处理拆分投票。每个 candidate 在选举开始时重新启动其随机选举超时，并在开始下一次选举之前等待该超时过去;这降低了在新选举中再次分裂投票的可能性。

### 5.3 Log replication

一旦一个 leader 被选出，它就开始为客户请求提供服务。每个客户请求都包含一个要由复制的状态机执行的命令。Leader 将命令作为一个新的条目附加到它的日志中，然后向其他每个服务器并行地发出 AppendEntries RPCs 来复制该条目。当条目被安全复制后（如下所述），Leader 将条目应用于其状态机，并将执行结果返回给客户端。如果 follower 崩溃或运行缓慢，或者网络数据包丢失，leader 会无限期地重试 AppendEntries RPCs（甚至在它回应了客户端之后），直到所有 follower 最终存储所有日志条目。


日志的组织方式如图 6 所示。每个日志条目都存储了一个状态机命令，以及 leader 收到该条目时的 term 编号。日志条目中的 term 编号被用来检测日志之间的不一致，并确保图 3 中的一些属性。每个日志条目也有一个整数索引，用于识别它在日志中的位置。

![](../.gitbook/assets/Logs%20are%20composed%20of%20entries,%20which%20are%20numbered%20sequentially.png)

Leader 决定何时将日志条目应用于状态机是安全的；这样的条目被称为 *committed*。Raft 保证所 committed 条目是持久的，最终会被所有可用的状态机执行。一旦创建该条目的 Leader 将其复制到大多数服务器上，该日志条目就会 committed（例如，图6 中的条目 7）。这也会 commit leader 日志中所有之前的条目，包括之前 leader 创建的条目。Leader 会跟踪它所知道的 committed 最高索引，并且它在未来的 AppendEntries RPC（包括心跳）中包括该索引，以便其他服务器最终发现。一旦 follower 得知一个日志条目 committed，它就会将该条目应用到它的本地状态机（按日志顺序）。

我们设计的 Raft 日志机制在不同服务器上的日志之间保持高度的一致性。这不仅简化了系统的行为，使其更具可预测性，而且是确保安全的重要组成部分。Raft 维护以下属性，它们共同构成了图 3 中的日志匹配属性：
- 如果不同日志中的两个条目具有相同的索引和 term，那么它们存储的是同一个命令。
- 如果不同日志中的两个条目具有相同的索引和 term，那么日志中的所有前面的条目都是相同的。

第一个属性来自于这样一个事实，即一个 leader 在一个给定的 term 中最多创建一个具有给定日志索引的条目，并且日志条目永远不会改变它们在日志中的位置。
第二个属性由 AppendEntries 执行的简单一致性检查来保证。当发送 AppendEntries RPC 时，leader 包括其日志中紧接新条目之前的条目的索引和 term。如果 follower 在其日志中没有找到具有相同索引和 term 的条目，那么它将拒绝新条目。
一致性检查作为一个归纳步骤：日志的初始空状态满足日志匹配属性，并且每当日志被扩展时，一致性检查会保留日志匹配属性。因此，每当 AppendEntries 成功返回时，leader 知道 follower 的日志与自己的日志在新条目之前是相同的。

在正常运行期间，leader 和 follower 的日志保持一致，所以 AppendEntries 一致性检查从未失败。然而，leader 崩溃会使日志不一致（老 leader 可能没有完全复制其日志中的所有条目）。这些不一致会在一系列 leader 和 follower 的崩溃中加剧。图 7 说明了 follower 的日志可能与新 leader 的日志不同的方式。Follower 可能会丢失 leader 的条目，可能会有 leader 没有的额外条目，或者两者都有。日志中缺失和多余的条目可能跨越多个 term。

![](../.gitbook/assets/When%20the%20leader%20at%20the%20top%20comes%20to%20power,%20it%20is%20possible%20that%20any%20of%20senarios%20could%20in%20follower%20logs.png)

在 Raft 中，leader 通过强迫 follower 的日志重复自己的日志来处理不一致的情况。这意味着 follower 日志中的冲突条目将被 leader 日志中的条目覆盖。

为了使 follower 的日志与自己的日志保持一致，leader 必须找到两个日志一致的最新日志条目，删除 follower 日志中此后的任何条目，并向 follower 发送此后 leader 的所有条目。所有这些动作都是为了响应 AppendEntries RPCs 所进行的一致性检查而发生的。Leader 为每个 follower 维护一个 nextIndex，它是 leader 将发送给该 follower 的下一个日志条目的索引。当 leader 第一次上台时，它将所有的 nextIndex 值初始化为其日志中最后一条的索引（图 7 中的 11）。如果 follower 的日志与 leader 的日志不一致，在下一个 AppendEntries RPC 中，AppendEntries 一致性检查将失败。在拒绝之后，leader 会递减 nextIndex 并重试 AppendEntries RPC。最终，nextIndex将达到一个点，即 leader 和 follower 的日志匹配。当这种情况发生时，AppendEntries 将会成功，它将删除 follower 日志中任何冲突的条目，并从 follower 的日志中添加条目（如果有的话）。一旦 AppendEntries 成功，follower 的日志就与 leader 的日志一致了，并且在剩下的时间里，它将保持这种状态。

如果需要，该协议可以被优化以减少被拒绝的 AppendEntries RPCs 的数量。例如，当拒绝一个 AppendEntries 请求时，follower 可以包括冲突条目的 term 和它为该 term 存储的第一个索引。有了这些信息，leader 可以递减 nextIndex 以绕过该 term 中的所有冲突条目；每个有冲突条目的 term 将需要一个 AppendEntries RPC，而不是每个条目一个 RPC。在实践中，我们怀疑这种优化是否有必要，因为故障不常发生，而且不太可能有很多不一致的条目。

使用此机制，leader 在掌权时无需采取任何特殊操作来恢复日志一致性。它刚刚开始正常操作，日志会自动收敛以响应 AppendEntries 一致性检查的失败。Leader 从不覆盖或删除其自己的日志中的条目（图 3 中的 Leader Append-Only 属性）。

此日志复制机制表现出第 2 节中描述的理想共识属性：只要大多数服务器启动，Raft 就可以接受、复制和应用新的日志条目;在正常情况下，可以使用一轮 RPC 将新条目复制到大多数群集;单个慢速追随者不会影响性能。

### 5.4 Safety

前面的部分介绍了 Raft 如何选举 leader 和复制日志条目。但是，到目前为止描述的机制还不足以确保每个状态机以相同的顺序执行完全相同的命令。例如，当 leader commits 多个日志条目时，follower 可能不可用，然后它可以被选为 leader 并用新条目覆盖这些条目;因此，不同的状态机可能会执行不同的命令序列。

本节通过添加对哪些服务器可以当选 leader 的限制来完成 Raft 算法。该限制可确保任何给定 term 的 leader 包含以前 term 中 commit 的所有条目（图 3 中的 Leader Completeness Property 属性）。鉴于选举限制，我们随后使承诺规则更加精确。最后，我们提供了 leader 完整性属性的证明草图，并展示了它如何导致复制状态机的正确行为。

#### 5.4.1 Election restriction

在任何基于 leader 的共识算法中，leader 最终必须存储所有 committed 日志条目。在某些共识算法中，例如 Viewstamped Replication ，即使 leader 最初不包含所有 committed 条目，也可以选出 leader 。这些算法包含额外的机制，用于识别缺失的条目并将其传输给新 leader ，无论是在选举过程中还是之后不久。不幸的是，这导致了相当大的额外机制和复杂性。Raft 使用了一种更简单的方法，它保证从每个新 leader 当选的那一刻起，每个新 leader 就存在以前任期的所有 committed 条目，而无需将这些条目转移给 leader。这意味着日志条目只沿一个方向流动，从 leader 流向 follower，并且 leader 永远不会覆盖其日志中的现有条目。

Raft 使用投票过程来阻止 candidate 赢得选举，除非其日志包含所有 committed 条目。Candidate 必须联系集群的大多数成员才能当选，这意味着每个 committed 条目必须至少存在于其中一个服务器中。如果 candidate 的日志至少与该多数中的任何其他日志一样最新（下面精确定义“最新”），那么它将保存所有 committed 条目。RequestVote RPC 实现了此限制：RPC 包含有关 candidate 日志的信息，如果选民自己的日志比 candidate 的日志更新，则选民拒绝其投票。

Raft 通过比较日志中最后一个条目的索引和 term 来确定两个日志中哪个是最新的。如果日志具有不同 term 的最后一个条目，则具有较晚 term 的日志是最新的。如果日志以相同的 term 结尾，则日志越长越最新。

#### 5.4.2 Committing entries from previous terms

如前节所述，leader 知道一旦当前条目存储在大多数服务器上，就会 commits 该条目。如果 leader 在 commits 条目之前崩溃，则未来的 leader 将尝试完成该条目的复制。但是，leader 不能立即得出结论，一旦将前一个 term 的条目存储在大多数服务器上，它就会 committed。图 8 说明了一种情况，即旧的日志条目存储在大多数服务器上，但仍可被未来的 leader 覆盖。

![](../.gitbook/assets/A%20time%20sequence%20showing%20why%20a%20leader%20cannot%20determine%20commitment%20using%20log%20entries%20from%20older%20terms.png)

为了消除图 8 中的问题，Raft 从不通过计数副本来 commits 以前 term 的日志条目。仅通过计算副本来 commits leader 当前 term 的日志条目;以这种方式 commits 当前 term 中的条目后，由于日志匹配属性，将间接 commit 所有先前的条目。在某些情况下，leader 可以安全地得出结论，commits 较旧的日志条目（例如，如果该条目存储在每台服务器上），但 Raft 为了简单起见采取了更保守的方法。

Raft 在承诺规则中会产生这种额外的复杂性，因为当 leader 从以前的 term 复制条目时，日志条目会保留其原始 term 编号。在其他共识算法中，如果新的 leader 从以前的“term”中重新复制条目，则必须使用新的“term 编号”来执行此操作。Raft 的方法可以更轻松地推理日志条目，因为它们随着时间的推移和跨日志保持相同的 term 编号。此外，与其他算法相比，Raft 中的新 leader 发送的先前 term 的日志条目更少（其他算法必须发送冗余日志条目以重新编号它们才能提交）。

#### 5.4.3 Safety argument

给定完整的 Raft 算法，我们现在可以更准确地论证 Leader Completeness Property 成立。我们假设 Leader Completeness Property 不成立，那么我们证明了一个矛盾。假设 term T 的 leader （leaderT）从其 term commits日志条目，但该日志条目未由某个未来 term 的 leader 存储。考虑最小的 term U > T，其 leader （leaderU） 不存储条目。

![](../.gitbook/assets/If%20S1%20(leader%20for%20term%20T)%20commits%20a%20new%20log%20entry%20from%20its%20term.png)

1. The committed 的条目在选举时必须从 leaderU 的日志中消失（leader 从不删除或覆盖条目）。
2. leaderT 在集群的大多数上复制了该条目，leaderU 获得了集群多数的投票。因此，至少有一个服务器（“投票者”）都接受了来自 leaderT 的条目并投票给 leaderU，如图 9 所示。选民是达成矛盾的关键。
3. 投票者在投票给 leaderU 之前必须接受 leaderT 的 committed entry;否则，它将拒绝来自 leaderT 的 AppendEntries 请求（其当前 term 将高于 T）。
4. 投票者在投票给 leaderU 时仍然存储条目，因为每个干预的 leader 都包含该条目（通过假设），leader 从不删除条目，follower 只有在与 leader 冲突时才删除条目。
5. 选民将其投票授予 leaderU，因此 leaderU 的日志必须与选民的日志一样最新。这导致了两个矛盾之一。
6. 首先，如果选民和 leaderU 共享相同的最后一个 log term，那么 leaderU 的日志必须至少与选民的日志一样长，因此其日志包含选民日志中的每个条目。这是一个矛盾，因为选民包含承诺的条目，而 leaderU 被假定没有。
7. 否则，领袖U的最后一个日志任期必须大于选民的任期。此外，它大于 T，因为选民的最后一个对数项至少是 T（它包含项 T 的提交条目）。创建 leaderU 最后一个日志条目的较早领导者必须在其日志中包含已提交的条目（通过假设）。然后，通过日志匹配属性，leaderU 的日志也必须包含提交的条目，这是一个矛盾。
8.这样就完成了矛盾。因此，所有大于 T 的项的领导者必须包含术语 T 中在术语 T 中提交的所有条目。
1. 日志匹配属性保证未来的领导者也将包含间接提交的条目，例如图 8（d） 中的索引 2。

### 5.5 Follower and candidate crashes

在这之前，我们一直专注于 leader 的失败。Follower 和 candidate 的崩溃比 leader 的崩溃要简单得多，而且它们的处理方式都是一样的。如果一个 follower 或 candidate 崩溃了，那么未来发送给它的 RequestVote 和 AppendEntries RPC 将会失败。Raft 通过无限期地重试来处理这些失败；如果崩溃的服务器重新启动，那么 RPC 将成功完成。如果服务器在完成 RPC 后但在响应前崩溃，那么它将在重新启动后再次收到相同的 RPC。Raft RPC 是等效的，所以这不会造成任何 harm。例如，如果一个 follower 收到一个 AppendEntries 请求，其中包括已经存在于其日志中的日志条目，那么它将忽略新请求中的这些条目。

### 5.6 Timing and availability

我们对 Raft 的要求之一是安全不能依赖于时间：系统不能因为某些事件发生得比预期快或慢而产生错误的结果。然而，可用性（系统及时响应客户的能力）必须不可避免地取决于时间。例如，如果信息交流的时间超过了服务器崩溃之间的典型时间，那么 candidate 就不会保持足够长的时间来赢得选举；没有一个稳定的 leaer ，Raft 就不能取得进展。

leader election 是 Raft 中时间最关键的方面。只要系统满足以下时间要求，Raft 就能选出并维持一个稳定的 leader。

$ broadcastTime \ll electionTimeou \ll MTBF $

在这个不等式中，broadcastTime 是一台服务器向集群中的每台服务器并行发送 RPC 并接收其响应所需的平均时间；electionTimeout 是第5.2节中描述的选举超时；MTBF 是单台服务器的平均故障间隔时间。广播时间应该比选举超时少一个数量级，这样领导者就可以可靠地发送心跳信息，以防止 follower 开始选举；考虑到选举超时使用的随机方法，这个不等式也使得分裂投票不太可能发生。选举超时应该比 MTBF 小几个数量级，这样系统才会有稳定的进展。当 leader 崩溃时，系统将在大约选举超时的时间内不可用；我们希望这只占总体时间的一小部分。

广播时间和 MTBF 是底层系统的属性，而选举超时是我们必须选择的。Raft 的 RPC 通常要求接收者将信息持久化到稳定的存储中，所以广播时间可能在 0.5ms 到 20ms 之间，这取决于存储技术。因此，选举超时可能是在 10ms 到 500ms 之间。典型的服务器MTBF 是几个月或更长时间，这很容易满足时间要求。

## 6 Cluster membership changes

## 7 Log compaction

Raft 的日志在正常操作期间会增长以包含更多客户端请求，但在实际系统中，它不能无限制地增长。随着日志变长，它会占用更多空间并需要更多时间来重播。这最终会导致可用性问题，而没有某种机制来丢弃日志中累积的过时信息。

快照是最简单的压缩方法。在快照中，整个当前系统状态将写入稳定存储上的快照，然后丢弃该点之前最多 11 个的整个日志。Snapshotting 在 Chubby 和 ZooKeeper 中使用，本节的其余部分介绍 Raft 中的快照。

增量压缩方法，例如日志清理和日志结构合并树，也是可能的。它们一次对一小部分数据进行操作，因此它们会随着时间的推移更均匀地分配压缩负载。他们首先选择一个累积了许多已删除和覆盖对象的数据区域，然后更紧凑地重写该区域的活动对象并释放该区域。与快照相比，这需要大量的额外机制和复杂性，快照通过始终对整个数据集进行操作来简化问题。虽然日志清理需要修改 Raft，但状态机可以使用与快照相同的接口实现 LSM 树。

![](../.gitbook/assets/A%20server%20replaces%20the%20committed%20entries%20in%20its%20log%20with%20a%20new%20snapshot.png)

图 12 显示了 Raft 中快照的基本思想。
每个服务器独立拍摄快照，仅覆盖其日志中提交的条目。大部分工作包括状态机将其当前状态写入快照。Raft 还在快照中包含少量元数据：最后一个包含的索引是快照替换的日志中最后一个条目的索引（状态机应用的最后一个条目），最后一个包含的任期是此条目的任期。保留这些条目是为了支持快照后第一个日志条目的 AppendEntries 一致性检查，因为该条目需要以前的日志索引和 term。若要启用群集成员身份更改（第 6 节），快照还包括日志中截至上次包含索引的最新配置。服务器完成快照写入后，可能会删除所有日志条目，直至最后一个包含的索引，以及任何先前的快照。

尽管服务器通常独立拍摄快照，但领导者必须偶尔将快照发送给落后的追随者。当领导者已经丢弃了需要发送给追随者的下一个日志条目时，就会发生这种情况。幸运的是，这种情况在正常操作中不太可能发生：跟上领导者的追随者已经拥有此条目。但是，异常慢的追随者或加入集群的新服务器（第 6 节）则不会。使此类关注者保持最新状态的方法是让领导者通过网络向其发送快照。

![](../.gitbook/assets/A%20summary%20of%20the%20InstallSnapshot%20RPC.png)

领导者使用名为 InstallSnapshot 的新 RPC 将快照发送给落后太远的关注者;参见图 13。当追随者收到具有此 RPC 的快照时，它必须决定如何处理其现有日志条目。通常，快照将包含收件人日志中尚未包含的新信息。在这种情况下，追随者会丢弃其整个日志;它全部被快照取代，并且可能包含与快照冲突的未提交条目。相反，如果追随者收到描述其日志前缀的快照（由于重新传输或错误），则快照涵盖的日志条目将被删除，但快照后面的条目仍然有效，必须保留。

这种快照方法背离了 Raft 的强领导者原则，因为追随者可以在领导者不知情的情况下拍摄快照。但是，我们认为这种离开是合理的。虽然有一个领导者有助于避免在达成共识时做出相互冲突的决定，但在快照时已经达成共识，因此没有决策冲突。数据仍然只从领导者流向下层，只有追随者现在可以重新组织他们的数据。

我们考虑了一种基于领导者的替代方法，其中只有领导者会创建一个快照，然后将此快照发送给其每个追随者。但是，这有两个缺点。首先，将快照发送给每个追随者会浪费网络带宽并减慢快照过程。每个关注者都已经拥有生成自己的快照所需的信息，并且服务器从其本地状态生成快照通常比通过网络发送和接收快照便宜得多。其次，领导者的实施会更加复杂。例如，领导者需要将快照发送给追随者，同时将新的日志条目复制到他们，以免阻止新的客户端请求。

还有两个问题会影响快照性能。首先，服务器必须决定何时创建快照。如果服务器快照过于频繁，则会浪费磁盘带宽和能源;如果快照频率太低，则可能会耗尽其存储容量，并且会增加重新启动期间重播日志所需的时间。一个简单的策略是在日志达到固定大小（以字节为单位）时拍摄快照。如果此大小设置为明显大于快照的预期大小，则快照的磁盘带宽开销将很小。

第二个性能问题是编写快照可能需要大量时间，我们不希望这会延迟正常操作。解决方案是使用写入时复制技术，以便可以接受新的更新，而不会影响正在写入的快照。例如，使用功能数据结构构建的状态机自然支持这一点。或者，操作系统的写入时复制支持（例如，Linux 上的 fork）可用于创建整个状态机的内存中快照（我们的实现使用这种方法）。

## 8  Client interaction

本节介绍客户端如何与 Raft 交互，包括客户端如何找到集群领导者以及 Raft 如何支持线性化语义。这些问题适用于所有基于共识的系统，Raft 的解决方案与其他系统类似。

Raft 的客户将所有请求发送给领导者。当客户端首次启动时，它会连接到随机选择的服务器。如果客户端的第一选择不是领导者，则该服务器将拒绝客户端的请求，并提供有关它从中听到的最新领导者的信息（AppendEntries 请求包括领导者的网络地址）。如果领导者崩溃，客户端请求将超时;然后，客户端使用随机选择的服务器重试。

Raft 的客户将所有请求发送给领导者。当客户端首次启动时，它会连接到随机选择的服务器。如果客户端的第一选择不是领导者，则该服务器将拒绝客户端的请求，并提供有关它从中听到的最新领导者的信息（AppendEntries 请求包括领导者的网络地址）。如果领导者崩溃，客户端请求将超时;然后，客户端使用随机选择的服务器重试。我们对 Raft 的目标是实现线性化的语义（每个操作似乎在其调用和响应之间的某个点立即执行一次）。但是，如前所述，Raft 可以多次执行命令：例如，如果潜在客户在提交日志条目后但在响应客户端之前崩溃，则客户端将使用新的领导者重试命令，从而导致第二次执行。解决方案是让客户端为每个命令分配唯一的序列号。然后，状态机跟踪为每个客户端处理的最新序列号以及关联的响应。如果它收到序列号已执行的命令，它将立即响应，而无需重新执行请求。

可以处理只读操作，而无需将任何内容写入日志。但是，如果没有其他措施，这将面临返回过时数据的风险，因为响应请求的领导者可能已被它不知道的新领导者所取代。线性化的读取不能返回过时的数据，Raft 需要两个额外的预防措施来保证这一点，而无需使用日志。首先，领导者必须拥有有关提交哪些条目的最新信息。领导者完整性属性保证领导者拥有所有已提交的条目，但在任期开始时，它可能不知道哪些条目是。要找出答案，它需要从其术语中提交一个条目。Raft 通过让每个领导者在其任期开始时在日志中提交一个空白的 no-op 条目来处理这个问题。其次，领导者必须在处理只读请求之前检查它是否已被废黜（如果最近选出的领导者，其信息可能会过时）。Raft 通过让领导者在响应只读请求之前与集群的大多数成员交换心跳消息来处理此问题。或者，领导者可以依靠心跳机制来提供一种形式的租赁，但这依赖于安全时序（它假设有界时钟偏差）。

