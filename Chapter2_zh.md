# 第2章：神秘的 MySQL 问题

MySQL 是最流行的开源数据库软件,拥有数十年的历史,以其简单性和用户友好性而闻名,成为互联网公司的基石选择。尽管被广泛采用,MySQL 仍面临各种挑战。

本章介绍了九个令人困惑的 MySQL 问题或现象,作为示例,为后续主题的深入探索奠定基础。

## 2.1 SysBench 读写测试表现出超线性吞吐量增长

例如,在 MySQL 8.0.27 发布版本中,在 x86 架构的 4 路 NUMA 环境中,使用 SysBench 远程测试 MySQL 的读写能力。MySQL 事务隔离级别设置为读已提交(Read Committed)。MySQL 实例 1 和实例 2 部署在同一台机器上,测试持续时间为 60 秒。MySQL 实例 1 和实例 2 的单独 SysBench 测试结果如下图所示。

<img src="media/image-20240829081346732.png" alt="image-20240829081346732" style="zoom:150%;" />

图 2-1. MySQL 单独运行时的吞吐量。

每个实例的吞吐量都不高,分别为 172,781 QPS 和 155,387 QPS。当合并时,两个实例的总吞吐量为 328,168 QPS。当使用 SysBench 同时测试这两个实例的读写能力时,获得的吞吐量分别为 271,232 QPS 和 275,197 QPS。

<img src="media/image-20240829204745429.png" alt="image-20240829204745429" style="zoom:150%;" />

图 2-2. MySQL 一起运行时的吞吐量。

两个 MySQL 实例的合并吞吐量为 546,429 QPS。这些数据表明,当这两个 MySQL 实例共享同一台机器时,合并吞吐量明显高于单独运行时各自吞吐量的总和。详细的统计比较请参见下图。

<img src="media/image-20240829081541149.png" alt="image-20240829081541149" style="zoom:150%;" />

图 2-3. 单独运行与一起运行的总吞吐量。

从数学逻辑上讲,如果两个实例一起运行时的总吞吐量大致等于单独运行时吞吐量的总和,则表示线性关系。如果合并吞吐量超过这个总和,则表示超线性关系。

是什么驱动超线性关系?MySQL 是否表现出超线性行为?理解这一点需要深入研究计算机基础知识和高级 MySQL 概念。

## 2.2 应用 PGO 后 TPC-C 吞吐量意外下降

配置文件引导优化(Profile-guided optimization,PGO)是一种成熟的技术,用于改进编译时优化决策。通过对可执行文件进行插桩或采样来收集配置文件信息,这些数据用于优化从中收集数据的可执行文件 [45]。尽管其有效性,但由于繁琐的双重编译模型,PGO 并未被软件项目广泛采用。尽管如此,PGO 仍然是一种非常有价值的优化技术,值得考虑用于改进 MySQL 性能,因为从理论上讲,它有可能显著提高 MySQL 的效率。

下图展示了将 PGO 应用于更高版本的 MySQL 8.0 的过程。

![](media/cb771856f5b0a021a92acc162c9c13bd.png)

图 2-4. 在更高版本的 MySQL 8.0 中使用 PGO:分步指南。

从图中可以看出,配置文件引导优化(PGO)机制涉及几个步骤:

1. 首先,使用编译选项 *"-DFPROFILE_GENERATE=ON"* 编译特定版本的 MySQL。
2. 启动这个 MySQL 版本,并通过运行性能测试(如 TPC-C)来捕获训练数据,这有助于收集性能指标。
3. 完成训练阶段后,使用选项 *"-DFPROFILE_USE=ON"* 进行第二次编译。在此编译期间,编译器自动利用收集的统计数据来优化条件分支和相关方面,显著提高生成的 MySQL 可执行文件的性能。

下图展示了在 MySQL 8.0.27 中应用 PGO 前后吞吐量与并发度之间的关系。

<img src="media/image-20240829081620633.png" alt="image-20240829081620633" style="zoom:150%;" />

图 2-5. MySQL 8.0.27 中使用 PGO 前后的性能比较测试。

根据图表,很明显 PGO 在低并发级别下显著改进了 MySQL 吞吐量。然而,超过 150 并发后,总体吞吐量和峰值性能都出现了下降。

PGO 主要是否只对低并发场景有益,还是有其他因素限制了其有效性?这个问题深入探讨了排队论和系统架构。进一步探索实用的计算机基础知识将为这个问题提供更深入的见解。

## 2.3 可扩展性增强后线程池对 MySQL 的不利影响

在对 MySQL 8.0.27 应用各种可扩展性补丁后,评估 Percona 线程池是否仍然有效解决可扩展性问题至关重要。下图描述了使用 BenchmarkSQL 对独立 MySQL 实例进行 TPC-C 测试的结果。深蓝色线表示启用 Percona 线程池的配置(线程池大小 = 128),而深红色线表示禁用线程池的配置。测试涵盖了从 50 到 2000 的并发级别,使用了具有 1000 个仓库的数据库。

<img src="media/image-20240829081719592.png" alt="image-20240829081719592" style="zoom:150%;" />

图 2-6. 启用 Percona 线程池导致吞吐量显著下降,相比于禁用时。

从图中可以清楚地看出,启用 Percona 线程池明显导致吞吐量显著下降,相比于禁用时。值得注意的是,即使没有启用线程池,MySQL 8.0 在可扩展性方面相比 MySQL 5.7 也显示出显著改进。这表明使用 Percona 线程池来改进 TPC-C 测试的额外好处是有限的。此外,Percona 线程池机制本身引入了开销,这在图表的结果中得到了体现。

重要的是要承认,Percona 线程池在涉及频繁连接创建和严重争用的场景中仍然有价值。然而,关键问题仍然是:究竟是什么对 MySQL 可扩展性做出了如此重大的改进?未来的章节将进一步探索这些奥秘。

## 2.4 在 MySQL 8.0 中,TPC-C 吞吐量下降太快

长期 TPC-C 测试的标准如下:TPC-C 基准要求数据库至少运行八小时,两小时内的抖动小于 2% [14]。

基于 MySQL 8.0.27,使用 BenchmarkSQL 工具进行了长期 TPC-C 测试。以下是 BenchmarkSQL 测试参数:

```
warehouses=1000
loadWorkers=100
terminals=200
warehouses-begin=1
warehouses-end=1000
//To run specified transactions per terminal- runMins must equal zero
runTxnsPerTerminal=0
//To run for specified minutes- runTxnsPerTerminal must equal zero
runMins=480
//Number of total transactions per minute
limitTxnsPerMin=0
//Set to true to run in 4.x compatible mode. Set to false to use the
//entire configured database evenly.
terminalWarehouseFixed=false
//The following five values must add up to 100
//The default percentages of 45, 43, 4, 4 & 4 match the TPC-C spec
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4
```

从上面可以看出,有 1000 个仓库,并发度为 200,terminalWarehouseFixed 设置为 false。此设置使每个事务每次都使用不同的仓库 ID,从而访问所有仓库的数据。

下图展示了长期测试期间随时间变化的吞吐量。TPC-C 吞吐量显示出的下降率大大超出预期,接近 50% 的下降。

<img src="media/image-degrade.png" alt="image-degrade" style="zoom:150%;" />

图 2-7. MySQL 8.0.27 的 BenchmarkSQL 测试期间暴露的性能下降。

这个问题是在使用 BenchmarkSQL 进行测试时发现的,不一定会在其他 TPC-C 测试工具中出现。截至当前版本 MySQL 8.0.40,吞吐量快速下降的问题尚未完全解决。后续章节将深入详细解释此问题的根本原因。

## 2.5 可重复读意外地优于读已提交

事务隔离是数据库处理的基础,由 ACID 缩写中的"I"表示。隔离级别决定了当多个事务并发地进行更改和查询时,性能与结果的可靠性、一致性和可预测性之间的平衡。常用的隔离级别是读已提交(Read Committed)、可重复读(Repeatable Read)和串行化(Serializable)。默认情况下,InnoDB 使用可重复读。

InnoDB 对每个隔离级别采用不同的锁定策略,影响并发条件下的查询锁定行为。根据隔离级别,查询可能需要等待其他会话当前持有的锁,然后才能开始执行 [13]。普遍认为更严格的隔离级别会降低性能。MySQL 在实际场景中的表现如何?

使用两种基准类型在串行化(Serializable)、可重复读(RR)和读已提交(RC)隔离级别上进行了测试:SysBench uniform 和 pareto 测试。SysBench uniform 测试模拟低冲突场景,而 SysBench pareto 测试模拟高冲突情况。由于 SysBench pareto 测试期间产生的过多死锁日志严重干扰了性能分析,因此通过修改源代码来抑制这些日志,以确保公平的测试条件。此外,MySQL 测试程序使用了修改版本以确保准确性,而非原始版本。

下图展示了 SysBench uniform 测试的结果,其中并发度从 50 增加到 800,以加倍的增量增加。鉴于此测试类型中的冲突很少,在低并发级别下,三个事务隔离级别的吞吐量变化很小。然而,超过 400 并发后,串行化隔离级别的吞吐量出现显著下降。

<img src="media/image-20240829151823981.png" alt="image-20240829151823981" style="zoom:150%;" />

图 2-8. 不同隔离级别下低冲突的 SysBench 读写性能比较。

在 400 并发以下,差异很小,因为 uniform 测试中的冲突较少。冲突较少时,不同事务隔离级别下锁策略的影响就会降低。然而,读已提交主要受到频繁获取 MVCC ReadView 的限制,导致性能不如可重复读。

继续在 pareto 分布条件下进行 SysBench 测试,具体的比较测试结果可以在下图中看到。

<img src="media/image-20240829081950365.png" alt="image-20240829081950365" style="zoom:150%;" />

图 2-9. 不同隔离级别下高冲突的 SysBench 读写性能比较。

图表清楚地说明,在冲突显著的场景中,不同事务隔离级别下由于锁策略导致的性能差异很明显。正如预期的那样,更高的事务隔离级别通常表现出更低的吞吐量,特别是在严重冲突条件下。

在冲突很少的场景中,性能主要受 MVCC 中获取 ReadView 的开销限制。这是因为,在读已提交隔离级别下,MySQL 每次从全局活动事务列表读取时都必须复制整个活动事务列表,而在可重复读下,它只需要在事务开始时获取活动事务列表的副本。

总之,在像 SysBench uniform 这样的低冲突测试中,MVCC ReadView 的开销是主要瓶颈,超过了锁开销。因此,可重复读的性能优于读已提交。相反,在像 SysBench pareto 这样的高冲突测试中,锁开销成为主要瓶颈,导致读已提交优于可重复读。

## 2.6 Group Replication 吞吐量低于半同步复制

在 Group Replication 运行期间,会维护一个认证数据库。定期清理过时的认证信息对于高效管理内存使用至关重要。然而,这个清理过程涉及获取全局锁存器,暂时暂停 MySQL 主节点执行,直到认证信息被清除。

相比之下,传统的半同步复制要求 MySQL 从节点处理大量的中继日志事件信息。只有在这些中继日志事件写入磁盘后,从节点才能将确认(ack)信息发送回 MySQL 主节点。这个过程包括网络交互、处理大量中继日志事件和磁盘刷新,导致半同步复制的响应时间相对较长。

总体而言,使用半同步复制时,MySQL 主节点必须等待 MySQL 从节点在中继日志事件写入磁盘后的确认,然后才能继续。相比之下,Group Replication 在 Paxos 层达成共识后就继续处理,而无需等待该层的日志写入。理论上,Group Replication 可以实现更高的吞吐量。

在两节点集群的场景中,在 Group Replication 和半同步复制之间进行了基于并发度的 TPC-C 吞吐量比较。详情请参见下图。

<img src="media/image-20240829082024079.png" alt="image-20240829082024079" style="zoom:150%;" />

图 2-10. Group Replication 与半同步复制的性能比较。

图表表明,在低并发下,半同步复制优于 Group Replication,而 Group Replication 在高并发下表现出优越的性能。半同步复制在 100 并发时达到峰值性能,而 Group Replication 在 250 并发时达到峰值,但峰值性能低于半同步复制。这些测试结果出乎意料。可能出了什么问题?

根本问题在于 Group Replication 使用的认证数据库机制,这在半同步复制中是不存在的。这个机制涉及大量的内存分配和释放,显著限制了吞吐量的提升。尽管不需要 Paxos 日志持久化,但这个瓶颈抵消了 Group Replication 的优势。显然,认证数据库机制的实现构成了 Group Replication 的主要性能挑战。

## 2.7 修改后的 Group Replication 优于半同步复制

在解决 MySQL 8.0.32 中的可扩展性问题时,Group Replication 得到了广泛的增强。为了验证这些改进,同时测试了半同步复制和带有 Paxos 日志持久化的 Group Replication。部署设置包括半同步和 Group Replication 的两节点配置,托管在同一台机器上,具有独立的 NVMe SSD 并绑定 NUMA 以隔离每个节点。具体来说,MySQL 主节点使用 NUMA 节点 0 到 2,而 MySQL 从节点使用 NUMA 节点 3。除了与半同步或 Group Replication 配置直接相关的设置外,所有设置都保持相同。

下图显示了在不同并发级别下半同步复制和带有 Paxos 日志持久化的 Group Replication 的吞吐量比较。

<img src="media/image-20240829082055680.png" alt="image-20240829082055680" style="zoom:150%;" />

图 2-11. 带有 Paxos 日志持久化的 Group Replication 与半同步复制的性能比较。

两者都采用持久化机制,Group Replication 使用 Paxos 日志持久化,半同步复制使用中继日志持久化。由于这些不同的机制,带有 Paxos 日志持久化的 Group Replication 表现出明显优于半同步复制的性能。

Meta 公司实现了基于 Raft 的 MySQL 高可用性解决方案。根据 Meta 公司开发人员进行的测试,基于 Raft 的改进版本的性能与半同步复制相当。具体详情请参见下图 [42]。

![](media/d00bee2b821eb3d268b99fb6ea94f916.png)

图 2-12. 从 Meta 论文中借用的吞吐量比较。

理论上,Group Replication 可以利用基于批处理的磁盘持久化机制,在磁盘写入期间无需处理 binlog 事件,从而实现更高的预期吞吐量。未来的章节将深入讨论修改 Group Replication 以及检查导致原生半同步复制可扩展性挑战的具体因素。

## 2.8 SysBench 无效果,TPC-C 表现良好

MySQL 8.0 中 lock-sys 优化的具体内容如下:

```c++
commit 1d259b87a63defa814e19a7534380cb43ee23c48
Author: Jakub Łopuszański <jakub.lopuszanski@oracle.com>
Date:   Wed Feb 5 14:12:22 2020 +0100
    WL#10314 - InnoDB: Lock-sys optimization: sharded lock_sys mutex

    The Lock-sys orchestrates access to tables and rows. Each table, and each row,
    can be thought of as a resource, and a transaction may request access right for
    a resource. As two transactions operating on a single resource can lead to
    problems if the two operations conflict with each other, Lock-sys remembers
    lists of already GRANTED lock requests and checks new requests for conflicts in
    which case they have to start WAITING for their turn.

    Lock-sys stores both GRANTED and WAITING lock requests in lists known as queues.
    To allow concurrent operations on these queues, we need a mechanism to latch
    these queues in safe and quick fashion.

    In the past a single latch protected access to all of these queues.
    This scaled poorly, and the management of queues become a bottleneck.

    In this WL, we introduce a more granular approach to latching.
```

这代表了 MySQL 8.0 中的可扩展性改进,通过解决全局锁存器瓶颈和优化 InnoDB 中的锁调度。为了验证此 lock-sys 优化的有效性,请参见下图中使用 SysBench 读写测试的具体测试结果。

<img src="media/image-20240829082201268.png" alt="image-20240829082201268" style="zoom:150%;" />

图 2-13. lock-sys 优化前后的 SysBench 读写测试比较。

令人惊讶的是,在实施 lock-sys 优化后,吞吐量下降了,这是意料之外的。为了减轻 MySQL 代码中 NUMA 兼容性问题的干扰,运行的 MySQL 实例被绑定到 NUMA 节点 0(类似于 SMP 环境)。测试结果如下。

<img src="media/image-20240829082231096.png" alt="image-20240829082231096" style="zoom:150%;" />

图 2-14. SMP 下 lock-sys 优化前后的 SysBench 读写测试比较。

图表显示,优化前后的差异最小,几乎可以忽略不计。这表明在测试过程中,lock-sys 优化的有效性被其他因素所掩盖,导致测试结果失真。然而,绑定到 NUMA 节点 0 减少了其他瓶颈的干扰,缩小了性能差距。这也表明 lock-sys 优化对 SysBench 标准读写测试的影响有限。

使用 BenchmarkSQL 进行 TPC-C 测试,结果如下:

<img src="media/image-20240829082255780.png" alt="image-20240829082255780" style="zoom:150%;" />

图 2-15. lock-sys 优化前后的 BenchmarkSQL 测试比较。

图表显示了 lock-sys 优化带来的明显改进。然而,这引发了一个问题:为什么 SysBench 测试没有效果,而 BenchmarkSQL 测试有效果?理解这两种工具之间的差异以及测试期间的重要注意事项将在即将到来的章节中彻底讨论。

## 2.9 禁用 NUMA 对 MySQL 真的有益吗?

首先测试了禁用 NUMA 对 MySQL 主节点的影响。部署设置如下:在两台具有相同硬件配置的 x86 机器上进行 BenchmarkSQL 高压压力测试。一台机器在 BIOS 中禁用了 NUMA,而另一台启用了 NUMA。TPC-C 吞吐量与并发度的比较如下图所示。

<img src="media/image-20240829082344150.png" alt="image-20240829082344150" style="zoom:150%;" />

图 2-16. 通过在 BIOS 中禁用 NUMA 显著提高了 TPC-C 吞吐量。

图表表明,在 x86 机器上禁用 NUMA 显著提高了 TPC-C 吞吐量。这种改进源于在 BIOS 中禁用 NUMA 后有利的内存分配机制,对像 MySQL 主服务器这样的应用程序特别有益。

那么,禁用 NUMA 也对 MySQL 从节点重放有益吗?使用前面提到的相同机器进行测试,设置详情如下:在 BIOS 中禁用 NUMA 的环境中,无法使用 NUMA 绑定,允许使用所有内存。相反,在 BIOS 中启用 NUMA 的环境中,MySQL 从节点绑定到 NUMA 节点 0。下图展示了在这些不同环境中测试的平衡重放速度。

<img src="media/image-20240829082406694.png" alt="image-20240829082406694" style="zoom:150%;" />

图 2-17. 在 BIOS 中禁用 NUMA 前后的平衡重放速度比较。

图表显示,在 BIOS 级别禁用 NUMA 导致平衡重放速度仅约 570,000 tpmC,主要是由于 MySQL 从节点的 NUMA 不友好问题尚未解决。相比之下,启用 NUMA 并将 MySQL 从节点绑定到单个 NUMA 节点可以实现超过 810,000 tpmC 的平衡重放速度。这个测试突出了在 BIOS 级别禁用 NUMA 对 MySQL 从节点重放效率的劣势。在禁用 NUMA 的情况下有效地进行 MySQL 从节点重放需要解决这些 NUMA 不友好问题,否则会显著降低效率。即将到来的第 10 章关于改进 MySQL 从节点重放将详细检查这些 NUMA 不友好问题。

## 2.10 总结

本章深入探讨了经典和复杂的 MySQL 问题,这些问题对分析构成了重大挑战。这些问题的解决始于详细的逻辑分析,如下一章所述。解决这些问题需要对计算机基础知识和 MySQL 内部机制有深刻的理解。计算机基础知识涵盖了广泛的主题,包括计算机体系结构、数据结构、算法、操作系统、计算机网络、编译器、排队论和分布式系统理论等。这些主题将在第 4 章中彻底探讨。第 5 章将专门关注 MySQL 内部机制。

[下一页](Part2_zh.md)
