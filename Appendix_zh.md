# 附录

## 术语表

本书包含众多术语。提前熟悉它们的含义将帮助您更好地理解所讨论的问题。

**1 在读写操作中实现强一致性**

MySQL Group Replication 通过两种机制确保强一致性：

**强一致性读（Before 机制）**：

读写（RW）事务在应用之前等待所有先前的事务完成。只读（RO）事务在执行之前等待所有先前的事务。这保证了事务读取最新的值，仅影响读取延迟。

**强一致性写（After 机制）**：

RW 事务等待直到其更改应用到所有组成员。这确保了一旦在本地成员上提交，任何成员上的后续读取都将看到已提交的值或更新的值。RO 事务不受影响。

**2 AI**

人工智能（AI）通过开发用于感知、学习和目标导向行动的方法和软件，使机器能够展现智能行为。机器学习（ML）是 AI 的一个子集，它创建能够从数据中学习以执行任务的算法，而无需显式指令。神经网络的进步显著提高了许多领域的性能。

深度学习是 ML 的一个重要方向，由于计算能力（特别是 GPU）的提升和大型训练数据集（如 ImageNet）的出现，它极大地增强了计算机视觉、语音识别和自然语言处理等领域。

在数据库中，AI 可以自动化操作、调整性能参数和构建 SQL 语句，从而提高效率和功能。MySQL HeatWave 将事务、分析和机器学习整合到一个托管服务中。

**3 算法**

设计算法是解决特定问题的指令。逻辑推理至关重要，因为它涉及创建逻辑组织的步骤序列以达到预期结果。程序员必须考虑控制流、数据流和交互，以确保算法高效、正确并满足要求。

**4 平衡重放速度**

对于 MySQL 从库重放，平衡重放速度是指在正常条件下从库与主库速度匹配的点。如果主库的速度处于或低于此阈值，则几乎没有延迟。然而，如果主库的速度超过此阈值，从库在事务重放进度上开始滞后。

**5 因果关系**

因果关系是一种关系，其中一个事件或状态（原因）导致产生另一个事件或状态（结果）。原因对结果负有部分责任，而结果依赖于原因。

**6 崩溃恢复**

数据库中的事务可能会意外中断。如果在事务中的所有更改完成并提交之前发生故障，数据库可能变得不一致和不可用。崩溃恢复是将数据库恢复到一致和可用状态的过程。

**7 数据结构**

在计算机科学中，数据结构是一种数据组织和存储格式，通常选择它是为了有效访问数据。它由数据值、它们的关系以及可以对它们执行的操作组成，形成一个代数结构。

**8 双写**

双写缓冲区是一个存储区域，InnoDB 在将页面写入数据文件中的最终位置之前，先将缓冲池中的页面写入该区域。如果在页面写入期间发生意外故障，InnoDB 可以在崩溃恢复期间使用双写缓冲区中的副本来恢复页面。

**9 双一**

Binlog 是一个逻辑日志，记录事务修改，用于 MySQL 中的数据复制和灾难恢复。相比之下，redo log 是一个物理日志，捕获对数据页的修改，对于在崩溃后恢复已提交的数据至关重要。这两个日志对于 MySQL 崩溃恢复都至关重要。为了防止在意外故障期间数据丢失，"双一"保护是必不可少的，它涉及确保 binlog 和 redo log 及时刷新到磁盘。

**10 容错**

容错是系统在其组件出现故障或错误的情况下继续正确运行的能力。

**11 GTID**

全局事务标识符（GTID）通过唯一标识每个事务简化了复制，在设置新从库或处理故障转移时无需引用日志文件或位置。使用基于 GTID 的复制，直接跟踪事务可确保轻松验证一致性：如果主库上提交的所有事务都存在于从库上，则保证一致性。

GTID 在主库和从库之间维护，使您能够通过二进制日志追踪任何事务的来源。一旦在服务器上提交了具有特定 GTID 的事务，具有相同 GTID 的重复事务将被忽略，确保每个事务仅在从库上应用一次，从而保持一致性。

在本书中，MySQL Server 在运行前配置了以下设置：

```
gtid_mode=ON
enforce_gtid_consistency=ON
```

此配置确保每个 MySQL 节点使用 GTID，简化了跟踪和故障排除问题的过程。

**12 高可用性**

高可用性是网络弹性的一个属性，确保尽管存在故障和运营挑战，仍能保持可接受的服务水平。

**13 幂等性**

幂等性是计算机科学中某些操作的属性，其中多次应用操作与应用一次具有相同的效果。

**14 Latch 与 Lock：关键区别**

在计算机科学中，latch 或 mutex（互斥锁的缩写）是一种同步原语，可防止多个线程同时访问状态。锁定是一种用于防止并发访问数据库中数据的技术，确保结果一致。

Latch 类似于 mutex，而 MySQL 中的锁结构如下：

```c++
/** 锁结构；由 lock_sys latches 保护 */
struct lock_t {
  /** 拥有该锁的事务 */
  trx_t *trx;
  /** 事务的锁列表 */
  UT_LIST_NODE_T(lock_t) trx_locks;
  /** 记录锁的索引 */
  dict_index_t *index;
  /** 记录锁的哈希链节点。单链表中的链接节点，由哈希表使用。*/
  lock_t *hash;
  union {
    /** 表锁 */
    lock_table_t tab_lock;
    /** 记录锁 */
    lock_rec_t rec_lock;
  };
  ...
```

那么，这两个概念有什么区别呢？考虑这个比喻 [19]：

- **latch** 将门、大门或窗户固定在适当位置，但不能防止未经授权的访问。
- 然而，**lock** 限制没有钥匙的人进入，确保安全和控制。

在 MySQL 中，全局 latch 用于序列化特定的处理过程。例如，以下是 MySQL 对全局 latch 作用的描述。

```c++
除了上述所有步骤（除了步骤 2，因为我们通常已经知道页面）都是通过单行完成的：
    locksys::Shard_latches_guard guard{*block_a, *block_b};
要"停止世界"，只需使用以下方式对全局 latch 进行 x-latch：
    locksys::Global_exclusive_latch_guard guard{};
这个类不暴露太多公共函数，因为意图是使用友元守卫类，如演示的 Shard_latches_guard。
*/
class Latches {
 private:
  using Lock_mutex = ib_mutex_t;
 ...
```

在 MySQL 中，锁是事务模型的组成部分，常见类型包括行锁和表锁。MySQL 中的死锁检测与锁相关，而不是 latch。

理解锁对以下方面至关重要：

- 实现大规模、繁忙或高度可靠的数据库应用程序
- 调整 MySQL 性能

熟悉 InnoDB 锁定和 InnoDB 事务模型对于这些任务至关重要。

值得注意的是，MySQL 中的锁对象需要 latch 保护以确保正确性，如以下代码所示。

```c++
/** 为等待的锁请求授予锁并释放等待的事务。调用者必须持有包含该锁的分片的 lock_sys latch，
但不持有 lock->trx->mutex。
@param[in,out]    lock    等待的锁请求
 */
static void lock_grant(lock_t *lock) {
  ut_ad(locksys::owns_lock_shard(lock));
  ut_ad(!trx_mutex_own(lock->trx));
  trx_mutex_enter(lock->trx);
  if (lock_get_mode(lock) == LOCK_AUTO_INC) {
    dict_table_t *table = lock->tab_lock.table;
    if (table->autoinc_trx == lock->trx) {
      ib::error(ER_IB_MSG_637) << "事务已经拥有"
                               << " AUTO-INC 锁！";
    } else {
      ut_ad(table->autoinc_trx == nullptr);
      table->autoinc_trx = lock->trx;
      ib_vector_push(lock->trx->lock.autoinc_locks, &lock);
    }
  }
  ...
```

**15 使用 replica_preserve_commit_order 维护事务顺序**

在 MySQL 中，*replica_preserve_commit_order* 配置确保从库上的事务按照它们在中继日志中出现的相同顺序提交。此设置为维护事务之间的因果关系奠定了基础：如果事务 A 在主库上在事务 B 之前提交，事务 A 也将在从库上在事务 B 之前提交。这防止了在从库上以相反顺序读取事务的不一致性。

**16 MVCC**

多版本并发控制（MVCC）是数据库管理系统使用的一种非锁定并发控制方法，用于启用对数据库的并发访问。

**17 网络延迟**

包交换网络中的网络延迟通常测量为往返延迟时间，包括从源到目的地再返回的延迟。这种延迟显著影响 MySQL 的性能。

**18 网络分区**

网络分区将计算机网络划分为独立的子网，可以是有意为优化而为，也可以是由于设备故障。分布式软件必须具有分区容错性，这意味着即使网络被分区，它也应该继续正确运行。

**19 NUMA**

非均匀内存访问（NUMA）是一种在多处理中使用的计算机内存设计，其中内存访问时间取决于内存位置相对于处理器的位置。NUMA 扩展了对称多处理（SMP）架构的扩展性。SMP 在可扩展性方面存在困难，因为它一次只允许一个处理器访问内存，导致瓶颈。NUMA 现在是主流服务器架构，减轻了这些问题。然而，将 MySQL 实例绑定到单个 NUMA 节点本质上使其作为纯 SMP 架构运行。

**20 OLTP**

在线事务处理（OLTP）是指用于面向事务的应用程序的数据库系统，例如运营系统。这些系统设计用于实时处理和响应用户请求。这与在线分析处理（OLAP）相反，后者专注于数据分析而不是事务处理。

**21 Paxos 算法**

Paxos 是一系列用于在不可靠或易出错的处理器网络中解决共识问题的协议。

**22 流水线**

在计算中，流水线或流水线处理类似于制造装配线，其中过程的不同阶段并发执行，即使某些阶段依赖于其他阶段的完成。这种方法允许多个操作同时进行，提高了整体效率并减少了处理时间。

**23 PGO**

配置文件引导优化（PGO）是一种使用分析数据来改进程序运行时性能的编译器技术。作为一种动态优化方法，PGO 基于运行时信息改进代码。

**24 读已提交隔离级别**

简单来说，读已提交隔离级别确保事务期间读取的任何数据在读取时已提交。它防止读取者看到未提交或"脏"数据。但是，它不保证如果事务再次读取相同的数据，数据将是相同的；数据可以在读取后更改。

**25 复制**

复制依赖于主服务器跟踪其二进制日志中的所有数据库更改（更新、删除等）。此日志记录从服务器启动开始改变数据库结构或内容的所有事件。SELECT 语句不被记录，因为它们不修改数据库。复制通过读取主服务器上二进制日志中的事件并在从服务器上处理它们来工作。事件以各种格式记录在二进制日志中，具体取决于事件类型。

**26 响应时间**

在计算中，响应时间测量系统响应服务请求所需的时间，表示服务的响应性。

**27 基于行的复制**

使用基于行的日志记录时，主库将详细说明对单个表行的更改的事件写入二进制日志。复制到从库涉及将这些行更改事件复制到从库，这个过程称为基于行的复制。

**28 状态机复制**

在计算机科学中，状态机复制（SMR）是一种通过复制服务器和协调客户端与这些副本的交互来实现容错服务的方法。在 MySQL Group Replication 集群中，使用 Paxos 算法采用 SMR。

**29 线程池**

在计算机编程中，线程池是一种用于实现并发执行的设计模式。它维护一个准备好在任务可用时执行任务的线程池。这种方法通过减少频繁创建和销毁线程相关的开销来提高性能。池中的线程数量根据程序的计算资源进行调整，优化任务执行和资源利用。

**30 吞吐量**

吞吐量衡量系统在单位时间内处理的请求数。常见的统计指标包括：

1. **每秒事务数（TPS）**：每秒执行的数据库事务数。
2. **每秒查询数（QPS）**：每秒执行的数据库查询数。
3. **TPC-C 的 tpmC**：TPC-C 基准测试中每分钟执行的 New-Order 事务率。
4. **TPC-C 的 tpmTOTAL**：TPC-C 基准测试中每分钟执行的总事务率。

**31 惊群**

在计算机科学中，当许多进程或线程被事件唤醒，但只有一个可以处理它时，就会发生惊群问题。这导致对资源的过度竞争，可能会冻结系统。

**32 TPC-C**

TPC-C 是一个 OLTP 基准测试，它根据每分钟执行的 New-Order 事务率来衡量性能。该速率报告为 tpmC（每分钟事务数 C），作为基准测试的主要性能指标。

**33 事务**

在数据库管理系统中，事务是一个逻辑工作单元，通常包含多个操作。事务不仅简化了应用程序编码的复杂性，而且是数据库管理系统的核心功能之一。

**34 事务限流**

在 MySQL 中，事务限流通过限制一次进入事务系统的用户线程数来管理高并发。这种方法通过控制系统负载来帮助减少系统压力并增强稳定性。

**35 视图变更**

视图表示给定时间 MySQL Group Replication 配置的活动成员。当组配置更改时（例如成员加入或离开时）发生视图变更，并同时传达给所有成员。每个视图由视图标识符唯一标识，该标识符在每次视图变更时生成。

## **测试工具**

### 1 SysBench

SysBench 是一个广泛使用的开源基准测试工具，用于测试开源数据库管理系统（DBMS）。它提供了对系统性能的快速评估，无需复杂的基准测试设置。该工具支持各种测试，包括传统的读写和只写测试，以及基于 Pareto 分布的冲突类型测试。

SysBench 的主要优点是其简单性、易用性和用户友好性。然而，这种简单性也可能是一个重大缺点。测试过程的过度简化可能导致失真，可能会错过关键问题，无法准确表示在线处理能力。因此，虽然 SysBench 提供了一种直接的基准测试方法，但它可能无法完全捕获数据库性能的复杂性。

### 2 TPC-C 测试工具

由事务处理性能委员会定义的 TPC-C 基准测试是一个 OLTP 测试，涉及 9 个表和 10 个外键关系。除 Item 表外，所有表都根据初始数据库加载期间指定的仓库数（W）按基数缩放。

<img src="media/3b2c3f6b25df292e0ed14955df4a7328.png" style="zoom:50%;" />

此模式由五个不同的事务使用，每个事务创建不同的访问模式：

1. **Item**：只读。
2. **Warehouse、District、Customer、Stock**：读写。
3. **New-Order**：插入、读取和删除。
4. **Order 和 Order-Line**：具有时间延迟更新的插入，导致行变旧且不经常读取。
5. **History**：仅插入。

这个小模式具有有限数量的事务，其多样化的访问模式促成了 TPC-C 作为主要数据库基准测试的持续重要性。在本书中，主要使用 BenchmarkSQL [68] 来评估 MySQL 中的 TPC-C 性能。

## MySQL 如何处理 SQL？

在计算机科学中，请求-响应或请求-回复模型是网络中的基本通信方法。它涉及一台计算机发送数据请求，另一台计算机响应该请求。具体来说，这种模式涉及向回复系统发送请求或发送消息，该系统处理请求并返回响应。

MySQL 使用经典的请求-响应模型：客户端向 MySQL Server 发送 SQL 查询，MySQL Server 处理这些查询并将响应发送回客户端。下图说明了 MySQL Server 8.0 中的标准 SQL 查询处理流程。

![](media/f3b8490d4e41a8da5e954aabf0a4aacf.png)

以下是 MySQL Server 如何通过详细示例处理 SQL 请求的方法。假设用户从 MySQL 客户端向 MySQL Server 发送以下 SQL 语句：

```
select name from student where id=1;
```

在执行 SQL 查询之前，MySQL Server 首先使用"解析器"解析 SQL 语句，该解析器执行两个基本任务：

**1. 词法分析**

MySQL Server 扫描您输入的 SQL 字符串并将其转换为 token，识别关键字和其他元素。

![](media/b9bd3a0655d66dfdb5690e999db6ac32.png)

**2. 语法分析**

使用词法分析中的 token，语法解析器检查 SQL 语句是否符合 MySQL 语法规则。验证成功后，它构建一个 SQL 语法树。这个树结构帮助后续模块从 WHERE 子句中提取关键组件，如 SQL 类型（例如 SELECT、INSERT）、表名、字段名和条件。例如，提供的 SQL 语句将生成表示这些组件的语法树：

![](media/0e32e448841daf774e632cb0884cd7f4.png)

这捕获了解析过程的核心，MySQL Server 使用 Bison 解析器构建语法树。

解析后，MySQL Server 在实际执行开始之前执行几个步骤来优化查询性能：

**1. 预处理器**

预处理器执行初步任务，例如验证表或字段的存在，并扩展通配符，如 \`select \*\` 中的 \`\*\` 以包含所有表列。

**2. 查询优化器**

查询优化器确定 SQL 查询的执行计划。此阶段包括：

- **逻辑查询重写**：将查询转换为逻辑等效形式。
- **基于成本的连接优化**：评估不同的连接方法以最小化执行成本。
- **基于规则的访问路径选择**：根据预定义规则选择最佳数据访问路径。

查询优化器生成执行计划，然后由查询执行引擎使用。

完成执行计划后，查询执行引擎开始执行 SQL 语句，逐记录与存储引擎交互。

以下是两种类型的执行过程：全表扫描和索引查询。

**1. 全表扫描**

假设进行查询以检索所有年龄大于 20 岁的学生信息：

```
select * from student where age > 20;
```

由于此查询条件不使用索引，优化器通过将访问类型设置为 ALL 来选择全表扫描。

![](media/71732b73a7cb4fe508f504fca85b0db3.png)

执行器和存储引擎的执行过程如下：

1. Server 层调用存储引擎的全扫描接口开始从表中读取记录。
2. 执行器检查检索到的记录的年龄是否超过 20。如果有可用空间，满足此条件的记录将被分派到网络写入缓冲区。
3. 执行器循环从存储引擎请求下一条记录。根据查询条件评估每条记录，满足条件的记录将被发送到网络写入缓冲区，前提是缓冲区未满。
4. 一旦存储引擎从表中读取了所有记录，它将通知执行器读取完成。
5. 收到完成信号后，执行器退出循环并将查询结果刷新到客户端。

为了优化性能，MySQL 通过在将记录发送到客户端之前检查网络缓冲区是否已满来最小化频繁的写系统调用。仅当缓冲区已满或收到完成信号时才发送记录。

**2. 索引查询**

考虑以下 SQL 查询：

```
select name from student where id < 3 and name like 'wang%';
```

在执行此查询之前，在 student 表的 name 字段上添加辅助索引：

```
alter table student add index index_name(name);
```

可以使用 *'explain'* 语句查看此 SQL 查询的执行计划，该语句显示查询现在利用了新创建的索引。

![](media/f0839a5643114a83f16d647c8c8299d3.png)

使用索引的执行过程如下：

1. 执行器请求存储引擎定位与查询条件匹配的第一个索引记录（例如 name LIKE 'wang%'）。

2. 存储引擎检索并将匹配的索引记录返回给 Server 层。

3. 执行器检查记录是否满足额外的查询条件（例如 id \< 3）。

   如果满足条件，除非缓冲区已满，否则将相应的 name 添加到网络缓冲区。如果不满足条件，执行器跳过该记录并从存储引擎请求下一条记录。

4. 此循环继续，执行器反复请求并评估与查询条件匹配的下一个索引记录，直到处理完所有相关索引记录。

5. 一旦存储引擎指示所有相关索引记录已被处理，执行器退出循环并将收集的结果发送到客户端。

使用索引允许存储引擎快速定位必要的记录，绕过扫描整个表的需要。通常，这显著提高了执行效率并加快了查询速度。

## MySQL 架构

下图说明了客户端-服务器架构：

![](media/fd0d20c24fda6e6b4d654a468ed84d2c.png)

MySQL 遵循客户端-服务器架构，将系统分为两个主要组件：客户端和服务器。

### 1 客户端

1. 客户端是与 MySQL 数据库服务器交互的应用程序。
2. 它可以是独立应用程序、Web 应用程序或任何需要数据库的程序。
3. 客户端将 SQL 查询发送到 MySQL 服务器进行处理。

### 2 服务器

1. 服务器是负责存储、管理和处理数据的 MySQL 数据库管理系统。
2. 它接收 SQL 查询，处理它们并返回结果集。
3. 它管理多个客户端的数据存储、安全性和并发访问。

客户端使用 MySQL 协议通过网络与服务器通信，使多个客户端能够同时交互。应用程序使用 MySQL 连接器连接到数据库服务器。MySQL 还提供客户端工具，例如基于终端的 MySQL 客户端，用于与服务器直接交互。

MySQL 数据库服务器包括几个守护进程：

1. **SQL 接口**：为应用程序提供标准化接口，使用 SQL 查询与数据库交互。
2. **查询解析器**：分析 SQL 查询以了解其结构和语法，将它们分解为组件以进行进一步处理。
3. **查询优化器**：评估给定查询的各种执行计划，并选择最有效的计划以提高性能。

在 MySQL 中，存储引擎负责数据的存储、检索和管理。MySQL 的可插拔存储引擎架构允许选择不同的存储引擎，例如 InnoDB 和 MyISAM，以满足特定的性能和可扩展性要求，同时保持一致的 SQL 接口。

文件系统组织和存储各种文件类型，包括数据和索引文件。MySQL 使用日志文件，如二进制日志和 redo 日志，来维护事务一致性并支持恢复机制。

总的来说，MySQL 遵循客户端-服务器架构，其中客户端将 SQL 查询发送到服务器进行处理。MySQL 支持可插拔存储引擎架构，允许以各种特性和性能特征管理数据存储和检索。

## MySQL 集群

创建容错系统的最常见方法是使用冗余组件，允许系统在一个组件发生故障时继续运行。

MySQL 中的复制将数据从一台服务器（主库）复制到一台或多台服务器（从库），提供了几个优点：

1. **横向扩展解决方案**：在多个从库之间分散负载以提高性能。所有写入和更新都发生在主服务器上，而读取可以发生在从库上，从而提高读取速度。
2. **分析**：允许在从库上进行分析而不影响主库性能。
3. **长距离数据分发**：为远程站点创建本地数据副本，而无需持续访问主库。

原始同步类型是单向异步复制。异步复制的优点是用户响应时间不受从库影响。但是，如果主服务器发生故障且从库未完全同步，则存在数据丢失的重大风险。

半同步复制除了异步复制之外，还要求主服务器上的提交等待至少一个从库确认并记录事务事件。这确保了从库上的数据是最新的，但影响用户响应时间并引入高可用性复杂性。

复制引入了显著的复杂性，因为它需要管理多个服务器而不是一个。这涉及解决经典的分布式系统问题，如网络分区和脑裂场景。主要挑战是一致地协调这些服务器，确保它们就系统和数据状态达成一致，并随着每次更改保持一致。本质上，服务器必须作为分布式状态机运行，要么作为单个实体前进，要么最终收敛到相同的状态。

MySQL Group Replication 通过强大的服务器协调提供分布式状态机复制。组内的服务器通过内置的成员服务自动维护一致的视图，在服务器加入或离开时进行更新。如果服务器发生故障，故障检测机制会提醒组。

事务提交需要对全局事务序列达成多数共识，确保统一的提交或中止决策。在脑裂场景中，网络分区会停止进度，直到问题解决。组通信系统（GCS）协议通过故障检测、成员管理和有序消息传递确保一致的数据复制，所有这些都由 Paxos 算法作为核心通信引擎提供支持。

MySQL Group Replication 可以在单主模式下运行，具有自动主选举功能，一次只允许一台服务器接受更新。或者，高级用户可以在多主模式下部署组，其中所有服务器都可以接受并发更新。但是，这要求应用程序管理此类部署的限制。

通常，更简单的集群设置更容易配置，但在出现问题时可能会变得更具挑战性。相比之下，更复杂的集群最初更难配置，但在处理问题时提供了更优雅的解决方案。复制机制的选择应基于具体的用例。

## 测试相关材料

这里提供了详细的硬件配置、操作系统和各种测试脚本，以帮助读者复制本书中提供的一些测试结果。

### 1 硬件配置

Intel(R) Xeon(R) Gold 6238 CPU @ 2.10GHz

cache_alignment : 64

cpu MHz : 3700.000

cache size : 30976 KB

x86 架构上 NUMA 环境中的节点详细信息：

![](media/d5e1fe52debddd4c5b288e3b6e9c5262.png)

磁盘都是 NVMe SSD。

除非另有说明，否则大多数测试都在这些硬件条件下进行。

### 2 操作系统

操作系统内核版本是 Linux 5.19.8。

### 3 BenchmarkSQL 脚本

创建 TPC-C 表的脚本可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/tableCreates.sql_base_for_test

创建相关索引的脚本可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/indexCreates.sql_base_for_test

BenchmarkSQL 测试的主要配置脚本如下：

```
warehouses=1000
loadWorkers=100
terminals=300
warehouses-begin=1
warehouses-end=1000
//要运行每个终端的指定事务数 - runMins 必须等于零
runTxnsPerTerminal=0
//要运行指定的分钟数 - runTxnsPerTerminal 必须等于零
runMins=5
//每分钟的总事务数
limitTxnsPerMin=0
//设置为 true 以在 4.x 兼容模式下运行。设置为 false 以均匀使用
//整个配置的数据库。
terminalWarehouseFixed=true
//以下五个值必须加起来等于 100
//默认百分比 45、43、4、4 和 4 与 TPC-C 规范匹配
newOrderWeight=45
paymentWeight=43
orderStatusWeight=4
deliveryWeight=4
stockLevelWeight=4
```

从图中可以看出，通常有 1000 个仓库，每次测试持续 5 分钟。

### 4 SysBench 脚本

SysBench 测试非常简单；这里只是关键参数。

SysBench 测试参数：*-table_size=1000000 --tables=1*

对于低争用的测试，使用参数 *--rand-type=uniform*；对于高争用的测试，使用参数 *--rand-type=pareto*。

### 5 使用 tpcc-mysql 脚本进行基准测试

tpcc-mysql 改进工具的下载链接：http://www.anysql.net/

![](media/77a84346325aa692f5e8b92741c6efb9.png)

如何使用 tpcc-mysql 工具？

假设用户名是 xxx，密码是 yyy，步骤如下：

数据加载命令如下：

```
./tpcc_load -h127.0.0.1 -d tpcc200 -u xxx -p "yyy" -P 3306 -w 200
```

测试命令如下：

```
./tpcc_start -h127.0.0.1 -P 3306 -d tpcc200 -u xxx -p "yyy" -w 200 -c 100 -r 0 -l 60 -F 1
```

### 6 配置参数

由于测试众多，这里仅列出典型配置。特殊配置需要相应的参数修改。

有关独立 MySQL 实例的典型配置参数的具体详细信息，请参阅以下地址：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/my.cnf_base_for_test

需要注意的是，默认情况下，测试在 *Read Committed* 事务隔离级别下进行，启用了 *binary logging*、*'dual one'* 配置和 *doublewrite*。

对于 Group Replication，原生 MySQL 版本中的主服务器配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:63318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50
loose-group_replication_flow_control_mode=disabled
slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

需要注意的是，读者应修改 IP 地址以匹配其特定环境。

对于 Group Replication，原生 MySQL 版本中从服务器的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:53318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50
loose-group_replication_flow_control_mode=disabled
slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

关于改进的 Group Replication，由于 MySQL 8.0.32 和 MySQL 8.0.40 之间相似，我们在以下地址提供了一个可供在线使用的版本：https://github.com/enhancedformysql/mysql-8.0.40。

相应地，主服务器的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:63318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50

slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

对于改进的 Group Replication，从服务器的配置参数如下：

```
# for mgr
plugin_load_add='group_replication.so'
enforce-gtid-consistency
gtid-mode=on
loose-group_replication_member_expel_timeout=3
loose-group_replication_start_on_boot= OFF
loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-baaaaaaaaaab"
loose-group_replication_local_address=127.0.0.1:53318
loose-group_replication_group_seeds= "127.0.0.1:63318,127.0.0.1:53318,127.0.0.1:43318"
loose-group_replication_member_weight=50

slave_parallel_workers=256
slave_parallel_type=LOGICAL_CLOCK
slave_preserve_commit_order=on
```

请注意，我们不再提供基于 MySQL 8.0.32 的源代码，但我们提供了基于 MySQL 8.0.40 的源代码。

与半同步复制相关的详细信息可以在以下地址找到：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/semisynchronous.txt

### 7 源代码仓库

**"Percona Server for MySQL 8.0.27-18" 的补丁：**

补丁地址：

https://github.com/enhancedformysql/mysql_8.0.27/blob/main/book_8.0.27_single.patch

此补丁专门针对独立 MySQL 实例的优化，包括：

- **MVCC ReadView** 增强
- **Binlog 组提交** 改进
- **查询执行计划** 优化

**集群源代码：**

MySQL 集群版本的源代码可在此处获得：https://github.com/enhancedformysql/mysql-8.0.40

对于 MySQL 集群，补丁引入了对 **Group Replication** 和 **MySQL 从库重放** 的进一步优化。

## 关于作者

早年，王斌在一家专注于开发高性能计算和高并发系统的互联网公司工作。他还为 TCPCopy [65] 和 MySQL Proxy [66] 等开源项目做出了贡献，在问题解决方面获得了宝贵的经验，特别是在逻辑思维方面。

离开互联网公司后，他专注于 MySQL 相关的开发，成功为 Group Replication、从库重放、InnoDB 存储引擎和查询优化 [67] 等项目做出了贡献。他在 MySQL 领域的问题解决方面积累了丰富的经验。
