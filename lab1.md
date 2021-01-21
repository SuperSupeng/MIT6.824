原文地址：[https://pdos.csail.mit.edu/6.824/labs/lab-mr.html](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)

---

# Getting started

您将使用git(一个版本控制系统)获取初始的程序。要了解更多关于git的信息，请查看 [Pro Git book](https://git-scm.com/book/en/v2)或[git user's manual](http://www.kernel.org/pub/software/scm/git/docs/user-manual.html)。获取6.824程序:

```bash
$ git clone git://g.csail.mit.edu/6.824-golabs-2020 6.824
$ cd 6.824
$ ls
Makefile src
```

我们在`src/main/mrsequential.go`中提供了一个简单的顺序mapreduce实现。它在单个进程中一起运行maps函数和reduce函数。我们还为您提供了几个MapReduce应用程序:`mrapps/wc.go`中的单词计数应用以及·mrapps/indexer.go·中的文本索引器程序。你可以按如下顺序运行单词计数程序:

```bash
$ cd ~/6.824
$ cd src/main
$ go build -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
```

`mrsequential.go`将其输出保留在文件mr-out-0中。输入来自名为pg-xxx.txt的文本文件。

可以随意从`mrsequential.go`借用代码。您还应该看看`mrapps/wc.go`文件，看看MapReduce应用程序代码是什么样的。

# Your Job

你的工作是实现一个分布式的MapReduce，包括两个程序，master和worker。只有一个master进程和一个或多个worker进程并行执行。在真实的系统中worker将在许多不同的机器上运行，但在本实验室中，您将在一台机器上运行所有的worker。workers将通过RPC与master通信。每个worker进程将向master进程请求任务，从一个或多个文件读取任务的输入，执行任务，并将任务的输出写入一个或多个文件。如果一个worker没有在合理的时间内完成任务(对于这个实验，使用10秒)，master应该注意到该情况，并把同样的任务交给另一个worker。

我们已经给了你一些代码来帮助你完成实验。master和worker的“main”程序在`main/mrmaster.go`和`main/mrworker.go`中，不要更改这些文件。你应该把你的实现放在`mr/master.go`, `mr/worker.go`, and `mr/rpc.go`中。

下面是如何在单词计数的MapReduce程序上运行代码。首先，确保word-count插件是新构建的:

```bash
$ go build -buildmode=plugin ../mrapps/wc.go
```

在主目录中，运行master。

```bash
$ rm mr-out*
$ go run mrmaster.go pg-*.txt
```

`mrmaster.go`文件的`pg-*.txt`参数是输入文件，每一个文件都需要被“split”，并且对应到一个Map任务中去。

在一个或多个窗口中运行一些worker:

```bash
$ go run mrworker.go wc.so
```

当worker和master任务结束后，查看mr-out-*中的输出。当你完成实验后，输出文件的排序联合应该匹配顺序输出，如下所示:

```bash
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
```

我们提供给你一个`main/test-mr.sh`的测试脚本。测试检查`wc`和`indexer`的MapReduce应用程序在将`pg-xxx.txt`文件作为输入时产生正确的输出。测试还检查实现是否并行运行Map和Reduce任务，以及是否实现了故障恢复机制。

如果你现在运行测试脚本，它会挂起，因为master永远不会结束:

```bash
$ cd ~/6.824/src/main
$ sh test-mr.sh
*** Starting wc test.
```

你可以在`mr/master.go`的`Done`函数中将`ret:= false`改为`true`通过这种方式让master程序结束，然后:

```bash
$ sh ./test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
```

测试脚本期望在名为`mr-out-X`的文件中看到输出，每个reduce任务对应一个文件。`mr/master.go`和`mr/worker.go`的空实现不会生成这些文件(或做很多其他的事情)，这样测试就会失败。

当你完成测试后，测试脚本的输出应该像这样:

```bash
$ sh ./test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
```

您还将看到来自Go RPC包的一些错误，如下所示

```bash
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
```

请忽略这些消息。

## 一些规则

- map阶段应该将intermediate keys划分为“nReduce”的桶以供reduce任务执行，其中“nReduce”是`main/mrmaster.go`文件的参数将会被传给`MakeMaster()`函数。
- `mr-out-X`文件应该包含每个Reduce函数输出的一行，应该使用Go的"%v %v"格式生成这行，即键和值的对应。查看`main/mrsequential.go`选择注释为“this is the correct format”的行。如果您的实现与这种格式偏离太多，那么测试脚本将会失败。
- 你可以修改`mr/worker.go`, ``mr/master.go`, and `mr/rpc.go`。你可以临时修改其他文件进行测试，但要确保你的代码与原始版本兼容，我们将用原始版本进行测试。
- worker应该将中间Map输出放在当前目录下的文件中，稍后worker可以将它们读取为输入执行reduce任务。
- `main/mrmaster.go`期望`mr/master.go`去实现一个`Done()`方法，当MapReduce任务完全完成时返回true，此时`mrmaster.go`将会退出。
- 当任务完全完成时，worker进程应该退出。实现这一点的一个简单方法是使用`call()`的返回值：如果worker未能联系到master，它可以假定master已经退出，因为任务已经完成，所以worker也可以终止。根据您的设计，您可能还会发现有一个“please exit”的伪任务(master可以给worker)会很有帮助。

## 提示

- 开始的一种方法是修改`mr/worker.go`的`Worker()`发送RPC给master请求任务。然后修改master以响应一个尚未启动的map任务的文件名。然后修改worker以读取该文件并调用应用程序的map函数，如mrsequential.go。

- 应用程序的Map和Reduce函数在运行时使用的Go插件包是从文件名以`.so`结尾的文件中加载得到的。

- 如果你改变了`mr/`目录下的任何东西，你可能需要重新构建你使用的任何MapReduce插件，比如使用`go build -buildmode=plugin ../mrapps/wc.go`。

- 这个实验依赖于workers共享一个文件系统。当所有的worker程序都运行在同一台机器上时，这是很简单的，但是如果worker程序运行在不同的机器上，就需要像GFS这样的全局文件系统。

- 中间文件的合理命名约定为`mr-X-Y`，其中X为Map任务号，Y为reduce任务号。

- worker的map任务代码将需要一种将`中间键/值对`存储在文件中的方法，这种方法可以在reduce任务期间正确地读回来。一种可能是使用Go的`encoding/json`,将键值对写入JSON文件。

  ```go
  enc := json.NewEncoder(file)
  for _, kv := ... {
    err := enc.Encode(&kv)
  ```

  并将这样的文件读回来:

  ```go
  dec := json.NewDecoder(file)
  for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
      break
    }
    kva = append(kva, kv)
  }
  ```

- worker的map部分可以使用`ihash(key)`函数(在`worker.go`中)为给定的key选择reduce任务。

- 你可以从`mrsequential.go`中参考一些代码读取Map输入文件对Map和Reduce之间的`中间键/值对`进行排序，并将Reduce输出存储在文件中。

- `master`作为RPC服务器将是并发的，不要忘记锁定共享数据。

- 使用Go的竞争检测器，例如Go build -race和Go run -race。test-mr.sh有一个注释，向您展示如何为测试启用race检测器。

- workers有时需要等待，例如，reduce不能开始，直到最后一个map完成。一种可能是worker周期性地向master请求工作，在每个请求之间用time.Sleep()休眠。另一种可能是master中的相关RPC处理程序有一个等待的循环，可以使用time.Sleep()或sync.Cond。Go在它自己的线程中为每个RPC运行处理程序，因此一个处理程序正在等待的事实不会阻止master处理其他RPC。

- master无法可靠地区分崩溃的worker、活着但由于某种原因停止工作的worker，以及正在执行但太慢而无法发挥作用的worker。你能做的最好的事情就是让master等待一段时间，然后放弃并重新将任务发给另一个worker。在这个实验中，让master等待10秒钟;在那之后，master应该假定worker已经死了(当然，它也可能没有)。

- 要测试崩溃恢复，可以使用`mrapps/crash.go`应用程序的插件。它随机地种植一些Map和Reduce函数。

- 为了确保在崩溃时没有人注意到部分写入的文件，MapReduce论文提到了使用临时文件并在它完全写入后自动重命名它的技巧。你可以用`ioutil.TempFile`创建临时文件和`os.Rename`做原子性的重命名。

- `test-mr.sh`运行子目录`mr-tmp`中的所有进程，因此，如果出现错误，您想查看中间文件或输出文件，请查看这里。











