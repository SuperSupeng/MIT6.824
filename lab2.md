原文地址：[https://pdos.csail.mit.edu/6.824/labs/lab-raft.html](https://pdos.csail.mit.edu/6.824/labs/lab-raft.html)

---

# Introduction
这是一系列实验中的第一个实验，您将在其中构建一个容错的key/value存储系统。在本实验室中，您将实现Raft，一个复制的状态机协议。在下一个实验中，您将在Raft之上构建一个key/value服务。然后您将在多个复制状态机上"Shard"您的服务，以获得更高的性能。

复制的服务通过在多个复制服务器上存储其状态（即数据）的完整副本来实现容错。复制允许服务继续运行，即使它的一些服务器发生故障（崩溃或网络中断）。面临的挑战是，故障可能导致副本持有不同的数据副本。

Raft将客户端请求组织成一个序列，称为日志，并确保所有副本服务器都能看到相同的日志。每个副本按照日志顺序执行客户端请求，将它们应用到服务状态的本地副本中。由于所有的实时副本都看到相同的日志内容，因此它们都会以相同的顺序执行相同的请求，从而继续拥有相同的服务状态。如果一个服务器发生故障，但后来又恢复了，Raft会负责将其日志更新。只要至少有大部分的服务器还活着，并且能够相互对话，Raft就会继续运行。如果没有这样的多数服务器，Raft将不会运行，但一旦多数服务器能够再次通信，Raft就会重新开始工作。

在本实验中，您将把Raft作为一个带有相关方法的Go对象类型来实现，意在作为一个更大服务中的模块来使用。一组Raft实例通过RPC相互对话来维护复制的日志。您的Raft接口将支持无限期的编号命令序列，也称为日志条目。这些条目用索引号进行编号。具有给定索引的日志条目最终将被提交。这时，你的Raft应该将日志条目发送到大的服务中让它执行。

你应该遵循[扩展的Raft论文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)中的设计，特别注意图2。你将实现论文中的大部分内容，包括保存持久化状态，并在节点失败后重新启动时读取它。您不会实现集群成员资格的改变（第6节）。您将在以后的实验室中实现日志压缩/快照（第7节）。

你可能会发现这个[指南](https://thesquareplanet.com/blog/students-guide-to-raft/)以及这个关于[锁](https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt)和[并发结构](https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt)的建议很有用。要想获得更广泛的视角，可以看看Paxos、Chubby、Paxos Made Live、Spanner、Zookeeper、Harp、Viewstamped Replication和[Bolosky等人的文章](https://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)。

这个实验室要分三部分提交。你必须在相应的到期日提交每个部分。

# Getting Started
如果您已经完成了实验室1，您已经有一份实验室源代码的副本。如果没有，您可以在Lab 1的说明中找到通过git获取源代码的方法。

我们为您提供了骨架代码`src/raft/raft.go`。我们还提供了一组测试，你应该用它来推动你的实现工作，我们将用它来给你提交的实验室打分。这些测试在`src/raft/test_test.go`中。

要启动和运行，请执行以下命令。不要忘了使用 git pull 来获取最新的软件。

```bash
$ cd ~/6.824
$ git pull
...
$ cd src/raft
$ go test
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.04s)
        config.go:326: expected one leader, got none
Test (2A): election after network failure ...
--- FAIL: TestReElection2A (5.03s)
        config.go:326: expected one leader, got none
...
$
```

# The code
通过在`raft/raft.go`中添加代码来实现Raft。在该文件中，你会找到骨架代码，以及如何发送和接收RPC的例子。

你的实现必须支持以下接口，测试者和（最终）你的key/value服务器将使用这些接口。你会在`raft.go`的注释中找到更多细节。

```go
// create a new Raft server instance:
rf := Make(peers, me, persister, applyCh)

// start agreement on a new log entry:
rf.Start(command interface{}) (index, term, isleader)

// ask a Raft for its current term, and whether it thinks it is leader
rf.GetState() (term, isLeader)

// each time a new entry is committed to the log, each Raft peer
// should send an ApplyMsg to the service (or tester).
type ApplyMsg
```

一个服务调用`Make(peers,me,...)`来创建一个Raft peer。`peers`的参数是Raft peers（包括这个）的网络标识符数组，用于RPC。`me`参数是peers数组中这个peer的索引。`Start(command)`要求Raft开始处理，将命令附加到复制的日志中。Start()应该立即返回，而不需要等待日志追加完成。服务希望你的实现为每个新提交的日志条目发送一个ApplyMsg到Make()的applyCh通道参数。

raft.go包含发送RPC(sendRequestVote())和处理传入RPC(RequestVote())的示例代码。您的Raft peers应该使用 labrpc Go 包交换 RPC（源码在`src/labrpc`中）。tester可以告诉 labrpc 延迟 RPC、重新排序和丢弃它们，以模拟各种网络故障。虽然你可以临时修改 labrpc，但请确保你的 Raft 与原始 labrpc 一起工作，因为我们将用它来测试和评定你的实验室。您的 Raft 实例必须只与 RPC 交互；例如，它们不允许使用共享的 Go 变量或文件进行通信。

后续的实验室是建立在这个实验室的基础上的，所以要给自己足够的时间来编写可靠的代码。

# Part 2A
