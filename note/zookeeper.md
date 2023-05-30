# ZooKeeper: Wait-free coordination for Internet-scale systems

## abstract

在本文中，我们描述了 ZooKeeper，一种用于协调分布式应用程序进程的服务。由于 ZooKeeper 是关键基础设施的一部分，因此ZooKeeper 旨在提供一个简单和高性能的内核，以便在客户端构建更复杂的协调原语。它将来自组消息传递、共享寄存器和分布式锁服务的元素合并到一个复制的集中式服务中。ZooKeeper 公开的接口具有共享寄存器的免等待方面，具有类似于分布式文件系统缓存失效的事件驱动机制，以提供简单而强大的协调服务。

ZooKeeper 接口支持高性能服务实现。除了免等待属性之外，ZooKeeper 还提供每个客户端的请求 FIFO 执行保证，并为更改 ZooKeeper 状态的所有请求提供线性化保证。这些设计决策支持实现高性能处理管道，本地服务器满足读取请求。我们展示了目标工作负载，2：1 到 100：1 的读写比率，ZooKeeper 每秒可以处理数万到数十万个事务。此性能允许客户端应用程序广泛使用 ZooKeeper。

## 1 Introduction

大规模分布式应用程序需要不同形式的协调。配置是最基本的协调形式之一。在最简单的形式中，配置只是系统进程的操作参数列表，而更复杂的系统具有动态配置参数。组成员资格和领导者选举在分布式系统中也很常见：进程通常需要知道哪些其他进程是活动的，以及这些进程负责什么。锁构成了一个强大的协调原语，可实现对关键资源的互斥访问。

协调的一种办法是为每种不同的协调需要开发服务。例如，Amazon Simple Queue Service [3] 专门关注队列。其他服务是专门为领导者选举[25]和配置[27]开发的。实现功能更强大的原语的服务可用于实现功能较弱的原语。例如，Chubby [6] 是一个具有强大同步保证的锁定服务。然后可以使用锁来实现领导者选举、组成员资格等。

在设计协调服务时，我们不再在服务器端实现特定的原语，而是选择公开一个 API，使应用程序开发人员能够实现自己的原语。这样的选择导致了协调内核的实现，该内核支持新的原语，而无需更改服务核心。此方法支持多种形式的协调，以适应应用程序的要求，而不是将开发人员限制在一组固定的原语上。

在设计 ZooKeeper 的 API 时，我们不再使用阻塞原语，例如锁。除其他问题外，阻止协调服务的原语可能会导致客户端速度慢或出现故障，从而对速度较快的客户端的性能产生负面影响。如果处理请求依赖于其他客户端的响应和故障检测，则服务本身的实现将变得更加复杂。因此，我们的系统 Zookeeper 实现了一个API，可以操作简单的无等待数据对象，这些对象按文件系统中的分层组织。事实上，ZooKeeper API 类似于任何其他文件系统的 API，仅从 API 签名来看，ZooKeeper 似乎很胖，没有锁定方法，打开和关闭。然而，实现免等待数据对象使 ZooKeeper 与基于阻塞原语（如锁）的系统显着区别开来。

尽管免等待属性对于性能和容错很重要，但它不足以进行协调。我们还必须为操作提供顺序保证。特别是，我们发现，保证所有操作的 FIFO 客户端排序和可线性化写入可以有效地实现服务，并且足以实现我们应用程序感兴趣的协调原语。事实上，我们可以使用我们的 API 为任意数量的进程实现共识，并且根据 Herlihy的层次结构，ZooKeeper 实现了一个通用对象[14]。

ZooKeeper 服务由一组服务器组成，这些服务器使用复制来实现高可用性和性能。它的高性能使包含大量进程的应用程序能够使用这样的协调内核来管理协调的各个方面。我们能够使用简单的流水线架构实现 ZooKeeper，该架构允许我们有数百或数千个未完成的请求，同时仍然实现低延迟。这样的管道自然允许按 FIFO 顺序从单个客户端执行操作。保证 FIFO 客户端顺序使客户端能够异步提交操作。使用异步操作，客户端能够一次具有多个未完成的操作。此功能是可取的，例如，当新客户端成为领导者并且它必须操作元数据并相应地更新它时。如果没有多个未完成操作的可能性，初始化时间可以是秒的数量级，而不是亚秒级。

为了保证更新操作满足线性化，我们实现了一个基于领导者的原子广播协议[23]，称为Zab [24]。但是，ZooKeeper 应用程序的典型工作负载由读取操作主导，因此需要扩展读取吞吐量。在 ZooKeeper 中，服务器在本地处理读取操作，我们不会使用 Zab 来完全排序它们。

在客户端缓存数据是提高读取性能的重要技术。例如，对于进程来说，缓存当前领导者的标识符而不是每次需要知道领导者时都探测 ZooKeeper 很有用。ZooKeeper 使用监视机制使客户端能够缓存数据，而无需直接管理客户端缓存。使用此机制，客户端可以监视给定数据对象的更新，并在更新时接收通知。Chubby 直接管理客户端缓存。它阻止更新以使缓存正在更改的数据的所有客户端的缓存失效。在此设计下，如果这些客户端中的任何一个运行缓慢或有故障，则更新将延迟。Chubby 使用租约来防止有故障的客户端无限期地阻塞系统。然而，租约只约束缓慢或有故障的客户端的影响，而 ZooKeeper 监视完全避免了这个问题。

在本文中，我们将讨论 ZooKeeper 的设计和实现。使用 ZooKeeper，我们能够实现应用程序所需的所有协调原语，即使只有写入是线性化的。为了验证我们的方法，我们展示了如何使用 ZooKeeper 实现一些协调原语。

总而言之，在本文中，我们的主要贡献是： 

**协调内核：** 我们提出了一种无等待的协调服务，具有宽松的一致性保证，用于分布式系统。特别是，我们描述了协调内核的设计和实现，我们已经在许多关键应用程序中使用它来实现各种协调技术。
 
**协调配方：** 我们展示了如何使用 ZooKeeper 来构建更高级别的协调原语，甚至是通常用于分布式应用程序的阻塞和强一致性原语。

**协调经验：** 我们分享了我们使用 ZooKeeper 的一些方法并评估其性能。

## 2 The ZooKeeper service



## References

[1] M. Abd-El-Malek, G. R. Ganger, G. R. Goodson, M. K. Reiter,
and J. J. Wylie. Fault-scalable byzantine fault-tolerant services.
In SOSP ’05: Proceedings of the twentieth ACM symposium on
Operating systems principles, pages 59–74, New York, NY, USA,
2005. ACM.
[2] M. Aguilera, A. Merchant, M. Shah, A. Veitch, and C. Karamanolis. Sinfonia: A new paradigm for building scalable distributed
systems. In SOSP ’07: Proceedings of the 21st ACM symposium
on Operating systems principles, New York, NY, 2007.
[3] Amazon. Amazon simple queue service. http://aws.amazon.com/sqs/, 2008.
[4] A. N. Bessani, E. P. Alchieri, M. Correia, and J. da Silva Fraga.
Depspace: A byzantine fault-tolerant coordination service. In
Proceedings of the 3rd ACM SIGOPS/EuroSys European Systems
Conference - EuroSys 2008, Apr. 2008.
[5] K. P. Birman. Replication and fault-tolerance in the ISIS system.
In SOSP ’85: Proceedings of the 10th ACM symposium on Operating systems principles, New York, USA, 1985. ACM Press.
[6] M. Burrows. The Chubby lock service for loosely-coupled distributed systems. In Proceedings of the 7th ACM/USENIX Symposium on Operating Systems Design and Implementation (OSDI),
2006.
[7] M. Castro and B. Liskov. Practical byzantine fault tolerance and
proactive recovery. ACM Transactions on Computer Systems,
20(4), 2002.
[8] T. Chandra, R. Griesemer, and J. Redstone. Paxos made live: An
engineering perspective. In Proceedings of the 26th annual ACM
symposium on Principles of distributed computing (PODC), Aug.
2007.
[9] A. Clement, M. Kapritsos, S. Lee, Y. Wang, L. Alvisi, M. Dahlin,
and T. Riche. UpRight cluster services. In Proceedings of the 22
nd ACM Symposium on Operating Systems Principles (SOSP),
Oct. 2009.
[10] J. Cowling, D. Myers, B. Liskov, R. Rodrigues, and L. Shira. Hq
replication: A hybrid quorum protocol for byzantine fault tolerance. In SOSP ’07: Proceedings of the 21st ACM symposium on
Operating systems principles, New York, NY, USA, 2007.
[11] G. DeCandia, D. Hastorun, M. Jampani, G. Kakulapati, A. Lakshman, A. Pilchin, S. Sivasubramanian, P. Vosshall, and W. Vogels. Dynamo: Amazons highly available key-value store. In
SOSP ’07: Proceedings of the 21st ACM symposium on Operating systems principles, New York, NY, USA, 2007. ACM Press.
[12] J. Gray, P. Helland, P. O’Neil, and D. Shasha. The dangers of
replication and a solution. In Proceedings of SIGMOD ’96, pages
173–182, New York, NY, USA, 1996. ACM.
[13] A. Hastings. Distributed lock management in a transaction processing environment. In Proceedings of IEEE 9th Symposium on
Reliable Distributed Systems, Oct. 1990.
[14] M. Herlihy. Wait-free synchronization. ACM Transactions on
Programming Languages and Systems, 13(1), 1991.
[15] M. Herlihy and J. Wing. Linearizability: A correctness condition for concurrent objects. ACM Transactions on Programming
Languages and Systems, 12(3), July 1990.
[16] J. H. Howard, M. L. Kazar, S. G. Menees, D. A. Nichols,
M. Satyanarayanan, R. N. Sidebotham, and M. J. West. Scale
and performance in a distributed file system. ACM Trans. Comput. Syst., 6(1), 1988.
[17] Katta. Katta - distribute lucene indexes in a grid. http://
katta.wiki.sourceforge.net/, 2008.
[18] R. Kotla, L. Alvisi, M. Dahlin, A. Clement, and E. Wong.
Zyzzyva: speculative byzantine fault tolerance. SIGOPS Oper.
Syst. Rev., 41(6):45–58, 2007.
[19] N. P. Kronenberg, H. M. Levy, and W. D. Strecker. Vaxclusters (extended abstract): a closely-coupled distributed system.
SIGOPS Oper. Syst. Rev., 19(5), 1985.
[20] L. Lamport. The part-time parliament. ACM Transactions on
Computer Systems, 16(2), May 1998.
[21] J. MacCormick, N. Murphy, M. Najork, C. A. Thekkath, and
L. Zhou. Boxwood: Abstractions as the foundation for storage
infrastructure. In Proceedings of the 6th ACM/USENIX Symposium on Operating Systems Design and Implementation (OSDI),
2004.
[22] L. Moser, P. Melliar-Smith, D. Agarwal, R. Budhia, C. LingleyPapadopoulos, and T. Archambault. The totem system. In Proceedings of the 25th International Symposium on Fault-Tolerant
Computing, June 1995.
[23] S. Mullender, editor. Distributed Systems, 2nd edition. ACM
Press, New York, NY, USA, 1993.
[24] B. Reed and F. P. Junqueira. A simple totally ordered broadcast protocol. In LADIS ’08: Proceedings of the 2nd Workshop
on Large-Scale Distributed Systems and Middleware, pages 1–6,
New York, NY, USA, 2008. ACM.
[25] N. Schiper and S. Toueg. A robust and lightweight stable leader
election service for dynamic systems. In DSN, 2008.
[26] F. B. Schneider. Implementing fault-tolerant services using the
state machine approach: A tutorial. ACM Computing Surveys,
22(4), 1990.
[27] A. Sherman, P. A. Lisiecki, A. Berkheimer, and J. Wein. ACMS:
The Akamai configuration management system. In NSDI, 2005.
[28] A. Singh, P. Fonseca, P. Kuznetsov, R. Rodrigues, and P. Maniatis. Zeno: eventually consistent byzantine-fault tolerance.
In NSDI’09: Proceedings of the 6th USENIX symposium on
Networked systems design and implementation, pages 169–184,
Berkeley, CA, USA, 2009. USENIX Association.
[29] Y. J. Song, F. Junqueira, and B. Reed. BFT for the
skeptics. http://www.net.t-labs.tu-berlin.de/
˜petr/BFTW3/abstracts/talk-abstract.pdf.
[30] R. van Renesse and K. Birman. Horus, a flexible group communication systems. Communications of the ACM, 39(16), Apr.
1996.
[31] R. van Renesse, K. Birman, M. Hayden, A. Vaysburd, and
D. Karr. Building adaptive systems using ensemble. Software Practice and Experience, 28(5), July 1998.