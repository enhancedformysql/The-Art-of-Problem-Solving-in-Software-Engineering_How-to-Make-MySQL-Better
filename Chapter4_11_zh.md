## 4.11 软件架构设计

常见的软件架构包括分层架构、主从/多主架构和事件驱动架构。独立的 MySQL 实例处理采用分层架构，而集群利用主从或多主架构。在通信的较低级别，Paxos 采用事件驱动架构。因此，MySQL 可以被描述为多种架构风格的混合体。

### 4.11.1 分层架构

分层架构是一种广泛采用的架构风格，它将不同的关注点分离到层中，以独立地满足不同的需求。例如，TCP/IP 协议栈是最成功的分层架构之一，在互联网领域得到广泛使用。MySQL 本身基于分层架构，使其能够支持各种类型的存储引擎。Group Replication 的处理模型同样遵循分层架构，如图所示[13]。

![](media/90cde9e64d700b21ced54c3610d9a88c.png)

图 4-69. Group Replication 插件框图。

MySQL Server 通过 API 调用与 Group Replication 插件交互。当事务需要经过 Group Replication 过程时，MySQL Server 通过 Group Communication System API 将事务信息发送到指定队列。随后，用户线程进入等待状态，等待负责 Group Replication 的线程激活。底层 Paxos 层（在图中称为 Group Communication Engine）负责将队列内容广播到组成员。一旦通过 Paxos 协议达成共识，上层就会收到通知以继续进一步处理。

这种架构的可扩展性如何？下图说明了 Group Replication 吞吐量与并发之间的关系。此外，利用事务节流机制来改善 InnoDB 在 2000+ 并发场景下的可扩展性，确保不会有太多用户线程进入 InnoDB 事务系统。

<img src="media/image-20240829090434767.png" alt="image-20240829090434767" style="zoom:150%;" />

图 4-70. 使用事务节流机制的 BenchmarkSQL 中的 Group Replication TPC-C 吞吐量。

从图中可以看出，Group Replication 的架构表现出良好的固有可扩展性，因为吞吐量不会随着线程数的增加而急剧下降。

### 4.11.2 主从/多主架构

MySQL 异步复制、半同步复制和 Group Replication 单主都采用主从架构。在 MySQL 中，主节点执行事务，而从节点重放事务，主节点和从节点之间不需要同步协调写入。

在 Group Replication 多主架构中，虽然事务可以在任何节点上执行，但存在几个已知的缺陷：

1. 缺乏全局事务管理器。
2. 事务隔离级别有限。
3. 在重写压力下可能出现流量倾斜。

根据用户反馈，用户通常以以下方式使用 Group Replication 多主：

1. 他们要求节点之间的事务不相互冲突。
2. 尽管是多主，但事务仅在一个节点上执行，以避免切换主节点的开销。

下图显示了 SysBench 随时间推移的读写性能，其中每个节点访问相同的数据库并处理读写任务。

<img src="media/image-20240829090550499.png" alt="image-20240829090550499" style="zoom:150%;" />

图 4-71. SysBench 读写测试中 Group Replication 随时间推移的吞吐量。

从图中可以看出，节点之间的读写测试相对和谐地共存。下图显示了 SysBench 随时间推移的仅写测试，其中 Group Replication 多主架构表现出不稳定的吞吐量。

<img src="media/image-20240829090612617.png" alt="image-20240829090612617" style="zoom:150%;" />

图 4-72. SysBench 仅写测试中 Group Replication 随时间推移的吞吐量。

接下来，让我们继续检查 Group Replication 多主架构中的测试场景，其中节点之间的事务不冲突。在主一上对数据库一进行测试，在主二上对数据库二进行测试。此设置确保主一和主二执行的事务没有冲突的可能性。具体结果如下图所示：

<img src="media/image-20240829090632045.png" alt="image-20240829090632045" style="zoom:150%;" />

图 4-73. SysBench 仅写测试中的 Group Replication 吞吐量：高并发且随时间推移无冲突。

从图中可以看出，即使不同节点上的事务之间没有冲突，仍然存在吞吐量倾斜。"主一"节点主要重放事务，严重限制了其接受新事务的能力。

值得注意的是，这些测试是在 100 并发下进行的。将压力降低到 10 并发可以缓解不均匀的流量倾斜，如下图所示。

<img src="media/image-20240829090658864.png" alt="image-20240829090658864" style="zoom:150%;" />

图 4-74. SysBench 仅写测试中的 Group Replication 吞吐量：低并发且随时间推移无冲突。

测试表明，Group Replication 多主架构的可扩展性存在问题，只能支持小规模流量。

### 4.11.3 事件驱动架构

基于事件驱动架构，在像 Nginx 这样的 Web 服务器中很常见，Percona 的线程池可以看作是事件驱动系统的粗略近似。然而，事件驱动架构并非免费——它会产生系统调用的开销，这种额外开销构成了使用线程池的成本。

MySQL 的底层 Paxos 通信也可以被视为异步事件驱动。此通信以单线程模式运行，需要避免任何同步通信过程。不幸的是，MySQL 在其持续的功能扩展中逐渐违反了这些原则，导致某些场景中由于延长的同步过程而导致吞吐量问题降至零。有关 Group Replication 同步的问题，请参阅下面的代码片段。

```c++
/* Try to connect to another node */
static int dial(server *s) {
  DECL_ENV
  int dummy;
  ENV_INIT
  END_ENV_INIT
  END_ENV;
  TASK_BEGIN
  IFDBG(D_BUG, FN; STRLIT(" dial "); NPUT(get_nodeno(get_site_def()), u);
        STRLIT(s->srv); NDBG(s->port, u));
  // Delete old connection
  reset_connection(s->con);
  X_FREE(s->con);
  s->con = nullptr;
  s->con = open_new_connection(s->srv, s->port, 1000); // synchronous call
  if (!s->con) {
    s->con = new_connection(-1, nullptr);
  }
```

[下一页](Chapter4_12_zh.md)
