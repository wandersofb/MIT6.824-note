# Fault-Tolerant Virtual Machines

The Design of a Practical System for Fault-Tolerant Virtual Machines 论文笔记。
感觉这篇论文读下来，没深度，没细节，可能是技术上并不是很难理解，感觉没有 GFS 质量高，有点 VMware 给自己的技术打广告的意思，但是可以建立起 FT 的概念了。

## INTRODUCTION

实现 **容错服务器** 的一种常见方法是 **primary/backup** 方法，其中 backup 服务器始终可以在 primary 服务器发生故障时接管。Backup 服务器的状态必须始终保持与 primary 服务器几乎相同，以便在 primary 服务器发生故障时 backup 服务器可以立即接管，并且故障对外部客户端是隐藏的并且没有数据丢失。在 backup 服务器上复制状态的一种方法是几乎连续地将 primary 服务器的所有状态（包括 CPU、内存和 I/O 设备）的更改发送到 backup 服务器。但是，发送此状态（尤其是内存中的更改）所需的带宽可能非常大。
一种可以使用更少带宽的复制服务器的方法被称为 **状态机方法**。这个想法是将服务器建模为确定性状态机，通过从相同的初始状态启动它们并确保它们以相同的顺序接收相同的输入请求来保持同步。由于大多数服务器或服务都有一些不确定的操作，因此必须使用额外的协调来确保 primary/backup 保持同步。但是，保持 primary/backup 同步所需的额外信息量远远少于 primary/backup 中正在更改的状态量（主要是内存更新）。
实施协调以确保物理服务器的确定性执行是困难的，尤其是随着处理器频率的增加。相比之下，运行在管理程序之上的 **虚拟机 (VM)** 是实现状态机方法的绝佳平台。可以将 VM 视为定义明确的状态机，其操作是被虚拟化的机器（包括其所有设备）的操作。与物理服务器一样，虚拟机有一些不确定的操作（例如读取时钟或发送中断），因此必须将额外的信息发送到备份以确保它保持同步。由于管理程序可以完全控制 VM 的执行，包括所有输入的传递，因此管理程序能够捕获有关主 VM 上的非确定性操作的所有必要信息，并在备份 VM 上正确地重放这些操作。

## BASIC FT DESIGN

![](../.gitbook/assets/Basic%20FT%20Configuration.png)

图 1 显示了我们的容错 VM 系统的基本设置。对于我们希望为其提供容错功能的给定 VM（Primary VM），我们在不同的物理服务器上运行一个 backup VM，该服务器与 primary 服务器保持同步并以相同的方式执行，但有一段时间的延迟。我们说这两个虚拟机处于虚拟同步状态。VM 的虚拟磁盘位于共享存储（例如光纤通道或 iSCSI 磁盘阵列）上，因此可供 primary VM 和 backup VM 访问以进行输入和输出。只有 primary VM 在网络上通告它的存在，因此所有网络输入都到达 primary VM。同样，所有其他输入（例如键盘和鼠标）仅进入 primary VM。

Primary VM 接收到的所有输入都通过称为 *logging channel* 的网络连接发送到 backup VM。对于服务器工作负载，主要的输入流量是网络和磁盘。附加信息将在必要时传输，以确保 backup VM 以与 primary VM 相同的方式执行非确定性操作。结果是 backup VM 始终以与 primary VM 相同的方式执行。然而，4backup VM 的输出被管理程序丢弃，因此只有 primary VM 产生返回给客户端的实际输出。

为了检测 primary VM 或 backup VM 是否发生故障，我们的系统不仅使用了相关服务器之间的 heartbeating 和对日志通道上流量的监控。此外，我们必须确保只有 primary 或 backup VM 中的一个接管执行，即使出现 primary and backup 服务器彼此失去通信的 split-brain 情况。

### Deterministic Replay(确定性重演) Implementation

如果两个确定性状态机以相同的初始状态启动并以相同的顺序提供完全相同的输入，那么它们将经历相同的状态序列并产生相同的输出。虚拟机具有广泛的输入集，包括传入的网络数据包、磁盘读取以及来自键盘和鼠标的输入。非确定性事件（如虚拟中断）和非确定性操作（如读取处理器的时钟周期计数器）也会影响 VM 的状态。这对复制运行任何操作系统和工作负载的任何 VM 的执行提出了三个挑战：
1. 捕获所有必要的输入和不确定性以确保备份虚拟机的确定性执行。
2. 将输入和不确定性应用于备份虚拟机。
3. 以不会降低性能的方式执行此操作。
此外，x86 微处理器中的许多复杂操作具有未定义的、因此是不确定的副作用。捕获这些未定义的副作用并重演它们以产生相同的状态提出了额外的挑战。

Deterministic Replay 将 VM 的输入以及与 VM 执行相关的所有可能的不确定性记录在写入日志文件的日志条目流中。稍后可以通过从文件中读取日志条目来准确重演 VM 执行。
对于不确定性操作，将记录足够的信息以允许使用相同的状态更改和输出来进行重演操作。
对于定时器或 IO 完成中断等非确定性事件，还会记录事件发生的确切指令。
在重演期间，事件在指令流中的同一点传送。VMware Deterministic Replay 实现了一种高效的事件记录和事件传递机制，该机制采用各种技术，包括使用与 AMD 和 Intel 联合开发的硬件性能计数器。

### FT Protocol

对于 VMware FT，我们使用 deterministic replay 来生成必要的日志条目来记录主 VM 的执行，但我们不是将日志条目写入磁盘，而是通过 logging channel 将它们发送到备份 VM。
备份 VM 实时重演条目，因此执行与主 VM 相同。但是，我们必须在 logging channel 上使用严格的 FT 协议来增加日志条目，以确保我们实现容错。我们的基本要求如下：
**输出规则**：Primary VM 不得向外部世界发送输出，直到 Backup VM 收到并确认与产生输出的操作相关的日志条目。

![](../.gitbook/assets/FT%20Protocol.png)

作为一个例子，我们在图 2 中展示了一个说明 FT 协议要求的图表。该图显示了 Primary VM 和 Backup VM 上的事件时间线。
从主线到备份线的箭头代表日志条目的转移，从备份线到主线的箭头代表同步。关于异步事件、输入和输出操作的信息必须作为日志条目发送到备份中并确认。如图所示，对外部世界的输出被延迟，直到 Primary VM 到来自 Backup VM 的确认，即它已经收到了与输出操作相关的日志条目。鉴于输出规则被遵守，Backup VM 将能够在输出一致的状态下接管 Primary VM。


### Detecting and Responding to Failure

VMware FT 的错误检测方法主要是流量检测，通过监测 logging traffic 来观察是否发生了故障。因为通常情况，logging channel 中传输的 log entries 和 acknowledgments 是相对平稳的，一旦出现停止，则预示着故障出现。

然而，任何此类故障检测方法都容易受到裂脑问题的影响。如果 Backup 服务器停止接收来自 Primary 服务器的心跳，这可能表明 primary 服务器出现故障，或者可能只是意味着仍在运行的服务器之间的所有网络连接都已丢失。如果 Backup VM 随后上线，而 Primary VM 实际上仍在运行，则可能会出现数据损坏以及与虚拟机通信的客户端出现问题。因此，我们必须确保在检测到故障时，只有 Primary VM or Backup VM 之一上线。为避免脑裂问题，我们使用存储 VM 虚拟磁盘的共享存储。当 Primary VM 或 Backup VM 想要上线时，它会在共享存储上执行原子测试和设置操作。如果操作成功，则允许 VM 上线。如果操作失败，那么另一个 VM 一定已经上线，因此当前 VM 实际上会自行停止（“自杀”）。如果 VM 在尝试执行原子操作时无法访问共享存储，那么它只会等待直到可以访问为止。请注意，如果由于存储网络中的某些故障而无法访问共享存储，那么 VM 可能无论如何都无法执行有用的工作，因为虚拟磁盘位于同一共享存储上。因此，使用共享存储来解决裂脑情况不会引入任何额外的不可用性。

## PRACTICAL IMPLEMENTATION OF FT

### Starting and Restarting FT VMs

对于想要以 primary state 启动 backup 或者重启 backup 之后恢复到 primary state，VMware 提供了一种状态迁移机制 VMotion，它可以把正在运行在服务器的 VM 迁移到另一台服务器上，其中 VM 的中断少于一秒。

针对于 FT 的情形，VMware 创造了一种新的 FT VMotion，它类似于上述的迁移过程，不过优化成了 primary 服务器通过 logging channel 克隆到新的 backup 服务器上，其中中断少于一秒。

### Managing the Logging Channel


