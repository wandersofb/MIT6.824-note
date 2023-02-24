# Lab2A

Lab2 是实现 Raft 算法，简单来说 Raft 算法是一个分布式存储系统中的容错算法，它可防止分布式系统的小部分机器出现错误，而让整个系统正常运行。具体内容可以看一下[论文](https://raft.github.io/raft.pdf)和[官方网站](https://raft.github.io)。

Lab2A 是实现其中的 leader election 这一部分，我在实现的时候，其实 coding 和 debug 的时间并不长，原因之一是这一部分的逻辑比较符合人的直觉的，另外是代码的整体情况是简单的和稀少的。但让我花费时间最长的是前期的论文阅读，Raft 逻辑梳理和 go thread Debug。

总共 `go test -run 2A` 测试 300 多次 2A，都成功了。
其实官网的 hints 我觉得还是比较少的，所以补充了一些 hints。

## Hints

- Election Timer 这个提示上说的不是很清楚，不是简单的睡眠到超时然后进行选举。它具体的行为是：rf 内部维护一个 timestamp 来记录最近一次重置 Timer 的时间，Timer（aka rf.ticker）会在开始的时候记录一个 timestamp，然后去睡眠，等到睡眠结束，会判断 rf timestamp 和 Timer timestamp 的大小，前者小则意味着在睡眠的这段时间没有重置 Timer 则开始选举。
- 当 candidate 发给 RequestVote RPCs 时，只要这个 RPC 没有得到回复，它会一直给相应的 client 发送 RPC，直到自己 killed || timeout || poll >= majority || be follower。
- 而 heartbeat 则不是上述行为，它是一致的给所有 clients 不停的发，没有回复就算了。
- Raft.RequestVote 是一个 RPC handler，这个函数的首字母要大写，因为我其他的函数命名范式都是首字符不大写，这个函数改成小写而不能使用了，这是写代码时的小错误，望注意。
- 线程同步问题，WaitGroup 是一个陷阱。
- Debug 一定要看看这个 [debug](https://blog.josejg.com/debugging-pretty/)。A picture is worth a thousand words。