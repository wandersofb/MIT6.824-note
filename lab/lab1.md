# Lab 1: MapReduce

终于开始写 MIT6.824 了，还是学习这些好玩的课程最有意思。学校的破事，找工作的破事和社会上的破事是真的烦人。不过这个 lab 我觉得难度还是有的。
- 其一是 golang 这是一门崭新的语言，之前完全没有接触过，但是学过之后竟然有点喜欢这个语言，感觉很像 python，而且对并发的支持是其他语言不能比肩的，说实话，golang 以后肯定会成为主流之一。
- 其二是全英论文，这点一度让我十分困扰，因为我的英文水平一般，使得我每次准备读论文根本没有耐心，都是读一点就结束了，反反复复读了好几次才搞懂 MapReduce 整个系统得逻辑（虽然 BackUp 后面之后的都没看）。
- 其三是 debug 的过程刚开始着实让我摸不着头脑。因为测试文件是 shell 脚本，要学习一下 shell 的语法规则，看懂测试文件，而 shell 脚本的语法又是极度晦涩的。幸亏 go 的调试工具是 dlv 和 gdb 极度相似，才不用在学新的 debug 工具。

在这个 lab 中，我们要实现一个 MapReduce 系统，具体的细节看论文和 lab 主页，我在这里简单介绍自己对 MapReduce 的理解和坑点，补充一些 lab 简介没有提到的 hints，并且简单介绍一下我是如何 debug 的。

## MapReduce

MapReduce 是 Google 的三大组件之一，是上个世纪末很重要的分布式计算模型，可以说是分布式的起始之一。论文里介绍了 MapReduce 的由来，总览，细节和性能等各个方面。我在这里简单说明一下，以便于后续的引用。
MapReduce 从架构上是一种客户端服务器架构，他有两个重要进程，worker 和 coordinator。worker 在客户端分布，它通常用于处理从 coordinator 那里分发过来的任务，其中任务包含了 map 和 reduce。而 coordinator 则是在服务器分布，它主要承担了准备任务，切割任务，每当有 workers 连接过来，再分配给他们的任务。
其中 map 和 reduce 则是解决问题的主要逻辑，map 是形成 key-value 的过程，并且把这些 kv 存储在中间文件。reduce 则是聚合相同的 key ，使 value 按照提出的逻辑处理。

## hints

补充一下 lab 页没给到但是我觉得还挺重要的提示。

### worker

- worker 应该不停的向 coordinator 请求任务，直到没有回复（coordinator 离开）或者 coordinator 提醒 worker 已经没有任务。
- 无论是 map 还是 reduce 任务都应该使用临时文件（ioutil.TempFile）
- 改名（rename）应该在最后的阶段，也就是结果已经确认了或者说是中间文件或者结果文件的输出都应该是原子的，要么不输出（crash）要么输出的结果正确。
- 在 reduce 任务里，我要读取目录下所有的文件，使用了 os.ReadDir 这个 API，希望对你有帮助。

### coordinator

- 在未完成 map 任务之前，不允许开始 reduce 任务，我使用的是条件变量来让这些线程睡眠。
- 对于 crash recovery，我的处理方案是在 MakeCoordinator 开启一个 goroutine，让它来监视那些正在运转的任务，每次增加一秒这些运转任务的生命周期，而他则休眠一秒之后继续监视。这样可以处理那些经过 10 秒还没有完成的任务，修改他们的状态为未完成。
- coordinator 是实实在在的多线程状态，由于我是用的条件变量，锁只有一个是条件变量内部的。

## debug

重点说一下 debug 的吧，主要还是网上并没有太多关于 debug 的说明。

我的调试流程是：先单线程，后多线程。
单线程主要看结果正不正确，也就是 worker 的工作是否正确，当然单线程下即便结果正确也无法保证 worker 的逻辑正确，rename 就是一个例子，如果不发生 crash，在多线程下不原子的 rename，worker 也会完成的很好。但是一旦 crash，就不好说了。
多线程主要看是否发生死锁，crash recovery 是否正常运转。

当然这些的前提是看懂 shell 脚本，建议把单个测试用例 c-v 到新的脚本下测试。

### vscode 远程调试

其实我最开始是这样搞的，但是不建议这样做，虽然 vscode 的调试界面很方便，但是这次是多线程，多个界面共同操作，vscode 我还没有这样搞过。即便是单线程调试，gdlv 也很好用，它的调试界面不比 vscode 差。

### dlv

dlv 和 gdb 是一样的，但还是建议使用 dlv，不过我是使用的 gdlv，有图形界面还挺舒服的。
我写了一个测试脚本，使用 gdlv 用来测试 word-count 这个任务。其实最主要的是其中的 `gdlv attach $pid` 和 `gdlv exec ../mrworker ../wc.so`，前者是从 mrcoordinator 中 attach 线程调试，这是 http 这种等待请求的经典调试方法，后者是直接执行 mrworker 这个客户端。

```shell
#!/usr/bin/env bash

go build -race -gcflags=all="-N -l" mrcoordinator.go
go build -race -gcflags=all="-N -l" mrworker.go
go build -race -gcflags=all="-N -l" -buildmode=plugin ../mrapps/wc.go


# run the test in a fresh sub-directory.
rm -rf mr-tmp
mkdir mr-tmp || exit 1
cd mr-tmp || exit 1
rm -f mr-*

../mrcoordinator ../pg-*txt &
pid=$!

gdlv attach $pid &
gdlv exec ../mrworker ../wc.so

rm -rf mr-tmp
rm -f mr-*

kill $pid
``` 

后面的 early exit 和 crash recovery，我是直接打印打法进行调试的。

```shell
bash test-mr-crash.sh 2>&1 | tee mylog.log
```

其中 test-mr-crash.sh 是单独从 test-mr.sh 摘出来的。

