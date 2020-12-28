# MIT6.824
MIT6.824-2020课程学习笔记与总结，遵循协议实验代码不开放，仅提供思路。

## 相关资料

- 视频地址：[bilibili](https://www.bilibili.com/video/BV1R7411t71W?from=search&seid=17665770671693338916), [YouTube](https://www.youtube.com/channel/UC_7WrbZTCODu1o_kfUMq88g/videos)
- 课程安排：通过[Schedule](https://pdos.csail.mit.edu/6.824/schedule.html)可以获得包括课件，实验，源码，Q&A 等一系列资源。

> 感谢雷神项目组提供的翻译资源：[Github](https://github.com/ivanallen/thor)

## 课程

### lab1: MapReduce
核心思想：

![mapreduce](./img/mapreduce.png)

算法流程：

<details><summary>英文版</summary>

- The MapReduce library in the user program first splits the input files into M pieces of typically 16 megabytes to 64 megabytes (MB) per piece (controllable by the user via an optional parameter). It then starts up many copies of the program on a cluster of machines.
- One of the copies of the program is special – the master. The rest are workers that are assigned work by the master. There are M map tasks and R reduce tasks to assign. The master picks idle workers and assigns each one a map task or a reduce task.
- A worker who is assigned a map task reads the contents of the corresponding input split. It parses key/value pairs out of the input data and passes each pair to the user-defined Map function. The intermediate key/value pairs produced by the Map function are buffered in memory.
- Periodically, the buffered pairs are written to local disk, partitioned into R regions by the partitioning function. The locations of these buffered pairs on the local disk are passed back to the master, who is responsible for forwarding these locations to the reduce workers.
- When a reduce worker is notified by the master about these locations, it uses remote procedure calls to read the buffered data from the local disks of the map workers. When a reduce worker has read all intermediate data, it sorts it by the intermediate keys so that all occurrences of the same key are grouped together. The sorting is needed because typically many different keys map to the same reduce task. If the amount of intermediate data is too large to fit in memory, an external sort is used.
- The reduce worker iterates over the sorted intermediate data and for each unique intermediate key encountered, it passes the key and the corresponding set of intermediate values to the user’s Reduce function. The output of the Reduce function is appended to a final output file for this reduce partition.
- When all map tasks and reduce tasks have been completed, the master wakes up the user program. At this point, the MapReduce call in the user program returns back to the user code.

</details>

### lab2: Raft

### lab3: KV_Raft

### lab4: Sharded_KV

## 推荐资源

 - [![DDIA](https://img2.doubanio.com/view/subject/s/public/s29872642.jpg)](https://book.douban.com/subject/30329536//)

书中引用：[https://github.com/ept/ddia-references](https://github.com/ept/ddia-references)

- [Distributed-Systems](https://github.com/feixiao/Distributed-Systems)