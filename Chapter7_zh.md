# 第7章：MySQL 8.0相对于MySQL 5.7的关键改进

MySQL 8.0相对于MySQL 5.7引入了实质性的改进。它不仅增强了功能并在执行计划中添加了对hash join的支持，而且更重要的是，极大地提高了可扩展性。这些进步为未来的改进奠定了坚实的基础。

## 7.1 扩展：InnoDB改进

早期的开源DBMS代码通常为整个内核使用粗粒度锁存器。相比之下，InnoDB采用了更精细的方法，为不同的内核组件（如锁管理器和缓冲池）使用单独的锁存器[19]。

MySQL 8.0引入了额外的改进以增强InnoDB存储引擎的可扩展性。以下是相关的改进：

1. **Redo Log优化：** 对redo log的增强促进了后续的性能改进。
2. **Lock-sys锁存器分片：** lock-sys锁存器已被分片，类似于读写锁，以改进事务锁定。
3. **Trx-sys锁存器拆分和分片：** 虽然锁存器的争用仍然存在，但trx-sys的优化为未来MVCC ReadView的改进奠定了坚实的基础。

这些重大的可扩展性改进将在下面详细讨论。

### 7.1.1 Redo Log优化

预写日志（Write-ahead logging）是ARIES风格并发和恢复中的一个基础性、无处不在的组件，它代表了一个重大的潜在瓶颈，特别是在频繁对数据进行小修改的OLTP工作负载中。确定了两个与日志记录相关的数据库系统可扩展性障碍，每个障碍都挑战软件架构的不同层级[3]：

1. 大量小型I/O请求可能会使磁盘饱和。
2. 当事务序列化访问内存中的日志数据结构时会产生争用。

上述潜在瓶颈反映在MySQL 5.7中。有关redo log优化的详细信息可以在"*MySQL 8.0: New Lock-Free, Scalable WAL Design*"中找到，其中复杂性在于新设计中如何确保日志序列号（Log Sequence Numbers, LSN）的顺序。该文章还强调了以下改进[27]：

*我们为与redo log写入相关的特定任务引入了专用线程。用户线程不再自己写入redo文件。当它们需要将redo刷新到磁盘但尚未刷新时，它们只是等待。*

这一改进完全改变了以前的机制，为可扩展性奠定了坚实的基础。以下git日志详细说明了对redo log的具体优化。

```c++
commit 6be2fa0bdbbadc52cc8478b52b69db02b0eaff40
Author: Paweł Olchawa <pawel.olchawa@oracle.com>
Date:   Wed Feb 14 09:33:42 2018 +0100

    WL#10310 Redo log optimization: dedicated threads and concurrent log buffer.

    0. Log buffer became a ring buffer, data inside is no longer shifted.
    1. User threads are able to write concurrently to log buffer.
    2. Relaxed order of dirty pages in flush lists - no need to synchronize
       the order in which dirty pages are added to flush lists.
    3. Concurrent MTR commits can interleave on different stages of commits.
    4. Introduced dedicated log threads which keep writing log buffer:
        * log_writer: writes log buffer to system buffers,
        * log_flusher: flushes system buffers to disk.
       As soon as they finished writing (flushing) and there is new data to
       write (flush), they start next write (flush).
    5. User threads no longer write / flush log buffer to disk, they only
       wait by spinning or on event for notification. They do not have to
       compete for the responsibility of writing / flushing.
    6. Introduced a ring buffer of events (one per log-block) which are used
       by user threads to wait for written/flushed redo log to avoid:
        * contention on single event
        * false wake-ups of all waiting threads whenever some write/flush
          has finished (we can wake-up only those waiting in related blocks)
    7. Introduced dedicated notifier threads not to delay next writes/fsyncs:
        * log_write_notifier: notifies user threads about written redo,
        * log_flush_notifier: notifies user threads about flushed redo.
    8. Master thread no longer has to flush log buffer.
    ...
    30. Mysql test runner received a new feature (thanks to Marcin):
        --exec_in_background.
    Review: RB#15134
    Reviewers:
        - Marcin Babij <marcin.babij@oracle.com>,
        - Debarun Banerjee <debarun.banerjee@oracle.com>.
    Performance tests:
        - Dimitri Kravtchuk <dimitri.kravtchuk@oracle.com>,
        - Daniel Blanchard <daniel.blanchard@oracle.com>,
        - Amrendra Kumar <amrendra.x.kumar@oracle.com>.
    QA and MTR tests:
        - Vinay Fisrekar <vinay.fisrekar@oracle.com>.
```

新机制使用专用线程刷新redo log文件，支持对日志缓冲区的并发写入，删除了代码中的全局锁存器，并引入了无锁存器处理，显著增强了可扩展性。

进行了一项测试，比较优化前后在不同并发级别下的TPC-C吞吐量。具体细节如下图所示：

<img src="media/image-20240829094221268.png" alt="image-20240829094221268" style="zoom:150%;" />

图7-1. 不同并发级别下redo log优化的影响。

图中的结果显示，在并发级别为100时吞吐量有显著提高，但在高并发级别时有所下降。这种下降可以归因于两个潜在原因：

1. **未解决的基础缺陷：** 在转换过程中，基础问题可能没有得到充分解决。
2. **多个队列瓶颈的干扰：** 可能会出现类似于多队列瓶颈相互干扰的问题。虽然某些区域的性能有所提高，但在高并发下其他瓶颈已经恶化。

大量研究表明，理论上这种优化应该提高吞吐量。redo log优化使用类似于组提交的机制来减少I/O开销。用户线程不是立即刷新redo log内容，而是写入日志缓冲区并等待，而专用线程批量将日志刷新到磁盘，并在过程完成时通知用户线程。这种方法预计将在高并发下显著减少I/O操作。因此，性能问题最可能的原因是其他队列中的瓶颈加剧。

实现redo log优化极具挑战性，如果没有它，几乎不可能实现数百万tpmC的吞吐量水平。

广泛的测试表明，优化在低并发条件下表现良好，并显著加快了TPC-C数据加载过程。具体细节如下图所示：

<img src="media/image-20240829095452552.png" alt="image-20240829095452552" style="zoom:150%;" />

图7-2. redo log优化对TPC-C数据加载时间的影响。

TPC-C数据加载过程涉及高达100MB的大事务。以前，加载1000个仓库需要77分钟，但通过优化，现在只需16分钟。这表明redo log优化对处理大事务非常有效。

为了评估此优化的真正价值，对MySQL 5.7.36应用了可扩展性增强。此过程首先应用trx-sys补丁，然后应用lock-sys补丁，以评估吞吐量改进的程度。具体细节如下图所示：

<img src="media/image-20240829100209917.png" alt="image-20240829100209917" style="zoom:150%;" />

图7-3. redo log优化的间接影响。

从图中可以看出，在应用trx-sys和lock-sys可扩展性补丁后，MySQL 5.7.36的吞吐量有所提高。然而，它并没有从根本上解决可扩展性问题，特别是与改进的MySQL 8.0.27版本相比。差距仍然很大。要识别250并发时的瓶颈，可以检查以下*perf*工具的截图。

![](media/739df2baa4dbff3a74a4cdc6d1d651e7.png)

图7-4. 250并发时的*perf*工具截图。

从图中可以明显看出，瓶颈是**prepare_write**，这正好对应于MySQL 5.7.36版本中写入redo log缓冲区的瓶颈。

```c++
/** Prepare to write the mini-transaction log to the redo log buffer.
@return number of bytes to write in finish_write() */
ulint mtr_t::Command::prepare_write() {
  switch (m_impl->m_log_mode) {
    case MTR_LOG_SHORT_INSERTS:
      ut_d(ut_error);
      /* fall through (write no redo log) */
      [[fallthrough]];
    case MTR_LOG_NO_REDO:
    case MTR_LOG_NONE:
      ut_ad(m_impl->m_log.size() == 0);
      return 0;
    case MTR_LOG_ALL:
      break;
    default:
      ut_d(ut_error);
      ut_o(return 0);
  }
  ...
```

让我们通过检查其调用栈关系来分析这个函数。

![](media/f558d2899635882ec517e2b1da5a0a8f.png)

图7-5. 揭示redo log写入瓶颈的调用栈关系。

该图清楚地显示了瓶颈在于redo log写入。如果没有redo log优化补丁，MySQL 5.7.36中的可扩展性问题无法从根本上解决，这突显了redo log优化的重大影响。

redo log优化目前有任何副作用吗？测试数据表明，在低并发条件下，刷新操作的数量显著增加。使用SysBench读写测试，统计分析了每个事务的平均I/O刷新次数与并发性的关系。具体细节如下图所示：

<img src="media/image-20240829100245272.png" alt="image-20240829100245272" style="zoom:150%;" />

图7-6. 低并发时redo log优化的副作用：更多的I/O刷新。

从图中可以观察到，在3个并发读写操作下，每个事务平均超过9次刷新，而在200并发时，每个事务减少到约1次刷新。这些平均刷新次数可以进一步优化，但需要找到平衡：及时刷新可以更快地激活用户线程，但会产生更高的I/O开销，而延迟刷新可以降低I/O成本，但可能会增加用户响应时间。

需要注意的是，Redo log改进主要集中在提高高并发环境下的整体性能，但在少于50个并发连接的场景中表现不佳。许多用户抱怨MySQL 8.0的性能未达到预期，这是其中一个根本原因。

### 7.1.2 通过锁存器分片优化Lock-Sys

在MySQL 5.7中，锁系统经历了严重的锁存器争用问题，这在高并发下严重影响了吞吐量。在事务执行期间，频繁的加锁和解锁操作需要获取全局锁存器。当许多用户线程竞争这个全局锁存器时，MySQL的可扩展性成为一个主要问题。

Lock-sys优化是MySQL 8.0中的第二个主要改进。以下git日志描述了lock-sys优化的具体细节。

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
    This scaled poorly, and the managment of queues become a bottleneck.
    In this WL, we introduce a more granular approach to latching.

    Reviewed-by: Pawel Olchawa <pawel.olchawa@oracle.com>
    Reviewed-by: Debarun Banerjee <debarun.banerjee@oracle.com>
      RB:23836
```

理论上，分片全局锁存器可以显著提高高并发情况下的可扩展性。基于使用lock-sys优化前后的程序，使用BenchmarkSQL比较TPC-C吞吐量与并发性，具体结果如下图所示：

<img src="media/image-20240829100432417.png" alt="image-20240829100432417" style="zoom:150%;" />

图7-7. 不同并发级别下lock-sys优化的影响。

从图中可以看出，优化lock-sys在高并发条件下显著提高了吞吐量，而在低并发下由于冲突较少，效果不太明显。

### 7.1.3 trx-sys中的锁存器拆分

MySQL中的trx-sys子系统与MVCC密切相关，主要涉及读操作。对redo log和lock-sys的改进主要与写操作相关。

MySQL 5.7使用全局锁存器来同步trx-sys内的各种操作。为了增强读取能力，解决这个锁存器瓶颈至关重要。然而，交织的逻辑使修改具有挑战性。

在MySQL 8.0中，全局锁存器最初被拆分。为*serialization_list*引入了一个新的锁存器，允许绕过全局锁存器并减少对它的争用压力。以下git日志描述了这些优化的具体细节。

```c++
commit e66d48b0c73d5fec278f81784bd5697502990263
Author: Paweł Olchawa <pawel.olchawa@oracle.com>
Date:   Mon Mar 1 15:52:30 2021 +0100

    BUG#27933068 USE DIFFERENT MUTEX TO PROTECT TRX_SYS->SERIALISATION_LIST

    This is an optimization patch, which reduces contention on the trx_sys_t::mutex
    by introducing a new mutex - the trx_sys_t::serialisation_mutex.

    The new mutex protects the trx_sys_t::serialisation_list and replaces the
    trx_sys_t::mutex when trx->no is being assigned.

    This is a modified version of the contribution patch which was created by Zhai Weixiang.
    Modifications:
    1. Periodical write of max_trx_id to the transaction system header page is modified.
    2. The trx_get_serial_no() is called when we do not hold trx_sys_t::mutex.
    3. Members in trx_sys_t are rearranged, so they are grouped by mutex that protects them.
    4. The new mutex received its own latch_id.
    5. InnoDB relies on rw_trx_max_id instead of max_trx_id in few places.
    6. The min_active_id is updated only when it really changes.

    RB: 19712
    Reviewed-by: Debarun Banerjee debarun.banerjee@oracle.com
```

基于此优化前后，使用BenchmarkSQL比较TPC-C吞吐量与并发性，具体结果如下图所示：

<img src="media/image-20240829100937477.png" alt="image-20240829100937477" style="zoom:150%;" />

图7-8. 不同并发级别下trx-sys中锁存器拆分的影响。

从图中可以看出，该优化在150并发时有效。然而，超过200并发后，吞吐量不仅没有增加反而下降。这种在高并发级别下的下降主要是由于其他队列瓶颈的干扰。

### 7.1.4 trx-sys的锁存器分片

在MySQL 8.0中，对trx-sys子系统进行了进一步的可扩展性改进。*rw_trx_set*已被划分为分片，每个分片都有自己的锁存器。这显著减少了读操作的全局锁存器争用。以下git日志描述了这些优化的具体细节。

```c++
commit bc95476c0156070fd5cedcfd354fa68ce3c95bdb
Author: Paweł Olchawa <pawel.olchawa@oracle.com>
Date:   Tue May 25 18:12:20 2021 +0200

    BUG#32832196 SINGLE RW_TRX_SET LEADS TO CONTENTION ON TRX_SYS MUTEX

    1. Introduced shards, each with rw_trx_set and dedicated mutex.
    2. Extracted modifications to rw_trx_set outside its original critical sections
       (removal had to be extracted outside trx_erase_lists).
    3. Eliminated allocation on heap inside TrxUndoRsegs.
    4. [BUG-FIX] The trx->state and trx->start_time became converted to std::atomic<>
       fields to avoid risk of torn reads on egzotic platforms.
    5. Added assertions which ensure that thread operating on transaction has rights
       to do so (to show there is no possible race condition).

    RB: 26314
    Reviewed-by: Jakub Łopuszański jakub.lopuszanski@oracle.com
```

基于这些优化前后，使用BenchmarkSQL比较TPC-C吞吐量与并发性，具体结果如下图所示：

<img src="media/image-20240829101111288.png" alt="image-20240829101111288" style="zoom:150%;" />

图7-9. 不同并发级别下trx-sys中锁存器分片的影响。

从图中可以看出，这一改进显著提高了TPC-C吞吐量，在200并发时达到峰值。值得注意的是，在300并发时影响减弱，主要是由于trx-sys子系统中与MVCC ReadView相关的持续可扩展性问题。这个问题将在下一章中进一步讨论。

### 7.1.5 总结

上述一系列可扩展性改进为MySQL实现高吞吐量奠定了坚实的基础。如果没有这些变化，后续的改进将失去其意义。因此，MySQL 8.0在可扩展性方面取得了重大进展。

## 7.2 评估MySQL锁调度算法的性能提升

调度在计算机系统设计中至关重要。正确的策略可以在不需要更快机器的情况下显著减少平均响应时间，有效地免费提高性能。调度还优化其他指标，如用户公平性和差异化服务级别，确保某些作业类别比其他类别具有更低的平均延迟[24]。

MySQL 8.0使用争用感知事务调度（Contention-Aware Transaction Scheduling, CATS）算法来优先处理等待锁的事务。当多个事务竞争相同的锁时，CATS根据调度权重确定优先级，该权重由给定事务阻塞的事务数量计算得出。阻塞最多其他事务的事务获得更高的优先级；如果权重相等，则等待时间最长的事务优先。

当多个事务无法继续进行时会发生死锁，因为每个事务都持有另一个事务需要的锁，导致所有相关事务无限期等待而不释放其锁。

在了解MySQL锁调度算法后，让我们检查这个算法如何影响吞吐量。在测试之前，有必要了解以前的FIFO算法以及如何恢复它。有关详细信息，请参考下面提供的git日志解释。

```c++
This WL improves the implementation of CATS to the point where the FCFS will be redundant (as often slower, and easy to "emulate" by setting equal schedule weights in CATS), so it removes FCFS from the code, further simplifying the lock_sys's logic.
```

根据上述提示，在MySQL中恢复FIFO锁调度算法很简单。随后，在改进的MySQL 8.0.32中使用SysBench Pareto分布场景在不同并发级别下测试了吞吐量。详细信息在下图中提供。

<img src="media/image-20240829101222447.png" alt="image-20240829101222447" style="zoom:150%;" />

图7-10. CATS在不同并发级别下对吞吐量的影响。

从图中可以看出，CATS算法的吞吐量显著超过FIFO算法。要比较这两种算法在用户响应时间方面的表现，请参考下图。

<img src="media/image-20240829101254601.png" alt="image-20240829101254601" style="zoom:150%;" />

图7-11. CATS在不同并发级别下对响应时间的影响。

从图中可以看出，CATS算法提供了明显更好的用户响应时间。

此外，比较Pareto分布测试过程中的死锁错误统计，详细信息可以在下图中找到。

<img src="media/image-20240829101332034.png" alt="image-20240829101332034" style="zoom:150%;" />

图7-12. CATS在不同并发级别下对忽略错误的影响。

对比分析表明，CATS算法显著减少了死锁。这种死锁的减少可能在提高性能方面发挥关键作用。这种相关性的理论基础如下[8]：

*在高竞争设置下，目标系统的吞吐量将由目标系统的并发控制机制决定：能够更早释放锁或减少中止次数的系统在这种设置下将具有优势。*

上述测试结果与MySQL官方的发现非常一致。以下两个图基于官方测试[57]，展示了CATS算法的显著有效性。

![](media/853d21533f748c1c56a4151869a82a27.gif)

图7-13. CATS和FIFO在TPS和平均延迟方面的比较：来自MySQL博客的见解。

此外，MySQL官方对实现CATS算法的要求很严格。具体细节在下图中提供：

![](media/4f0ea97ad117848a71148849705e311e.png)

图7-14. CATS官方工作日志的要求。

因此，采用CATS算法后，在所有场景中都不应出现性能下降。似乎事情在这里结束，但CATS算法论文[24]中的总结引起了一些疑问。详细信息在下图中提供：

![](media/4cc389ee95fbae485f1e014aad393aa8.gif)

图7-15. 对CATS论文的疑问。

从以上信息可以推断，要么业界忽略了FIFO的潜在缺陷，要么论文的评估有缺陷，FIFO并没有建议的严重问题。这种矛盾突显了一个关键问题：这些结论中的一个必须有缺陷；两者不能都是正确的。

矛盾通常为深入的问题分析和解决提供了宝贵的机会。它们突出了现有理解可能受到挑战或可以获得新见解的领域。

这次，在改进的MySQL 8.0.27上进行测试时，在MySQL错误日志文件中发现了大量错误日志。以下是部分截图：

![](media/4ca52ffeebc49306e76c74ed9062257d.png)

图7-16. 大量错误日志的部分截图。

继续分析相应的代码，具体如下：

```c++
void Deadlock_notifier::notify(const ut::vector<const trx_t *> &trxs_on_cycle,
                               const trx_t *victim_trx) {
  ut_ad(locksys::owns_exclusive_global_latch());
  start_print();
  const auto n = trxs_on_cycle.size();
  for (size_t i = 0; i < n; ++i) {
    const trx_t *trx = trxs_on_cycle[i];
    const trx_t *blocked_trx = trxs_on_cycle[0 < i ? i - 1 : n - 1];
    const lock_t *blocking_lock =
        lock_has_to_wait_in_queue(blocked_trx->lock.wait_lock, trx);
    ut_a(blocking_lock);
    print_title(i, "TRANSACTION");
    print(trx, 3000);
    print_title(i, "HOLDS THE LOCK(S)");
    print(blocking_lock);
    print_title(i, "WAITING FOR THIS LOCK TO BE GRANTED");
    print(trx->lock.wait_lock);
  }
  const auto victim_it =
      std::find(trxs_on_cycle.begin(), trxs_on_cycle.end(), victim_trx);
  ut_ad(victim_it != trxs_on_cycle.end());
  const auto victim_pos = std::distance(trxs_on_cycle.begin(), victim_it);
  ut::ostringstream buff;
  buff << "*** WE ROLL BACK TRANSACTION (" << (victim_pos + 1) << ")\n";
  print(buff.str().c_str());
  DBUG_PRINT("ib_lock", ("deadlock detected"));
  ...
  lock_deadlock_found = true;
}
```

从代码分析来看，很明显死锁会导致大量的日志输出。测试期间观察到的忽略错误与这些死锁有关。CATS算法有助于减少忽略错误的数量，从而减少日志输出。这个问题可以一致地复现。

鉴于这种情况，出现了几个考虑因素：

1. **对性能测试的影响：** 大量的错误日志和由此产生的中断可能会扭曲性能评估，导致对系统能力的不准确评估。
2. **CATS算法的有效性：** CATS算法的性能改进可能需要重新评估。如果错误日志的大量输出显著影响性能，其实际有效性可能不如最初认为的那么高。

设置`innodb_print_all_deadlocks=OFF`或从`Deadlock_notifier::notify`函数中删除所有日志记录，重新编译MySQL，并使用Pareto分布运行SysBench读写测试。详细信息在下图中提供：

<img src="media/image-20240829101534550.png" alt="image-20240829101534550" style="zoom:150%;" />

图7-17. 消除干扰后CATS在改进的MySQL 8.0.27不同并发级别下对吞吐量的影响。

从图中可以明显看出，吞吐量比较发生了重大变化。在冲突严重的场景中，CATS算法略优于FIFO算法，但差异很小，远不如以前的测试明显。请注意，这些测试是在改进的MySQL 8.0.27上进行的。

让我们在改进的MySQL 8.0.32上进行性能比较测试，消除死锁日志干扰，使用Pareto分布。

<img src="media/image-20240829101612063.png" alt="image-20240829101612063" style="zoom:150%;" />

图7-18. 消除干扰后CATS在改进的MySQL 8.0.32不同并发级别下对吞吐量的影响。

从图中可以明显看出，消除干扰后性能差异很小。这种小的差异使得理解为什么FIFO调度问题的严重性可能难以察觉。CATS作者和MySQL官方的感知偏见很可能是由于大量死锁日志输出的干扰。

使用与CATS算法论文中相同的32个仓库，在不同并发级别下进行了TPC-C测试。MySQL基于改进的MySQL 8.0.27，BenchmarkSQL被修改以支持每个仓库100个并发事务。

<img src="media/image-20240829101632142.png" alt="image-20240829101632142" style="zoom:150%;" />

图7-19. 根据CATS论文，消除干扰后CATS在NUMA下不同并发级别对吞吐量的影响。

从图中可以明显看出，CATS算法的性能比FIFO算法差。为了避免NUMA相关的干扰，MySQL被绑定到NUMA节点0进行新一轮的吞吐量与并发性测试。

<img src="media/image-20240829101650730.png" alt="image-20240829101650730" style="zoom:150%;" />

图7-20. 根据CATS论文，消除干扰后CATS在SMP下不同并发级别对吞吐量的影响。

在这一轮测试中，FIFO算法继续优于CATS算法。CATS算法在BenchmarkSQL TPC-C测试中性能下降，相比SysBench Pareto测试中的改进，可以归因于以下原因：

1. **额外开销**：CATS算法本身引入了一些额外开销。
2. **NUMA环境问题**：CATS算法在NUMA环境中可能表现不佳。
3. **冲突严重程度**：TPC-C测试中的冲突严重程度不如SysBench Pareto测试明显。
4. **不同的并发场景**：SysBench创建的并发场景与BenchmarkSQL中的场景显著不同。

最后，再次使用1000个仓库在不同并发级别下进行标准TPC-C测试。具体细节如下图所示：

<img src="media/image-20240829101712694.png" alt="image-20240829101712694" style="zoom:150%;" />

图7-21. 消除干扰后CATS对BenchmarkSQL吞吐量的影响。

从图中可以明显看出，在低冲突场景中，两种算法之间几乎没有差异。换句话说，在冲突较少的情况下，CATS算法不提供显著的好处。

总的来说，虽然CATS在Pareto测试中显示了一些改进，但不如预期明显。CATS算法显著减少了事务死锁，可能导致比FIFO算法更少的性能下降。当抑制死锁日志时，这些算法之间的差异很小，这澄清了围绕CATS算法性能的困惑。

数据库性能测试本质上是复杂且容易出错的[9]。它不能仅凭数据来判断，需要彻底调查以确保逻辑一致性。

## 7.3 MySQL执行计划的增强

### 7.3.1 MySQL中的Hash Join实现

顾名思义，散列是hash join算法的核心。它从一个输入表构建哈希表，然后逐行处理另一个表，使用哈希表进行查找。

Hash join通常更快，优于早期MySQL版本中使用的块嵌套循环算法。其好处是巨大的，如下面的实际案例所示。

在缺乏hash join支持的MySQL 5.7中，SQL查询依赖于传统的连接方法，导致执行时间较长，为3.82秒。

![](media/47b077e25be1755b8c698aea97f51f7f.png)

图7-22. MySQL 5.7中的非hash join性能。

MySQL 8.0引入了hash join。对于相同的SQL查询，使用带有提示的hash join将执行时间减少到1.22秒，比使用传统方法的3.82秒有显著改进。

![](media/12c296b6f5d91c71b24391af9c293363.png)

图7-23. MySQL 8.0中的hash join性能。

值得注意的是，MySQL 8.0中的hash join在以下条件下增强了连接性能[13]：

1. 没有可用的索引
2. 查询是I/O绑定的
3. 访问表的大部分
4. 跨多个表有选择性条件
5. 增加join_buffer_size可以进一步提高性能

hash join的引入是MySQL 8.0中的一个重要特性，为减少响应时间提供了一个有前途的解决方案。

### 7.3.2 MySQL中Hypergraph算法的引入

超图（hypergraph）算法在MySQL 8.0中引入，但目前仅在调试模式下可用。以下git日志提供了超图算法的具体实现细节。

```c++
commit b9be77784bf690173522d8db015acf0e72f28f84
Author: Steinar H. Gunderson <steinar.gunderson@oracle.com>
Date:   Wed May 6 16:32:13 2020 +0200

    WL #14070: Hypergraph partitioning algorithm

    Implement DPhyp for hypergraph partitioning, a central component of the join
    optimizer. The algorithm enumerates all possible connected sub-hypergraphs
    of the larger join graph, in a bottom-up fashion. (That is, for a given graph G
    with a subgraphs A and B than can further be partitioned respectively into A1/A2
    and B1/B2, A1 and A2 will both be seen before A, which will in turn be seen
    before S. However, there is no guarantee that B1 or B2 is seen before A.)

    The algorithm is described in the paper "Dynamic Programming Strikes Back" by
    Neumann and Moerkotte. There is a somewhat extended version of the paper
    (that also contains a few corrections) in Moerkotte's treatise "Building Query
    Compilers". Some critical details are still missing, which we've had to fill in
    ourselves. We don't currently implement the extension to generalized
    hypergraphs, but it should be fairly straightforward to do later.

    Since our graphs can never have more than 61 tables, node sets and edge lists
    are implemented using 64-bit bit sets. This allows for a compact representation
    and very fast set manipulation; the algorithm does a fair amount of
    intersections and unions. If we should need extensions to larger graphs later
    (this will require additional heuristics for reducing the search space), we can
    use dynamic bit sets, although at a performance cost.

    This is implemented entirely independently of the server; there are no MySQL
    dependencies, short of some shared header files for bit manipulations. It is
    tested using unit tests and microbenchmarks.

    Change-Id: I7912c09ab69a17e607ee3b8fb2af2bd7602e54ec
```

从以上可以看出，超图算法实现的理论基础详细描述在论文"Dynamic Programming Strikes Back" [35]中。这突显了其实现所涉及的高度复杂性。

基于成本的查询优化器对数据库管理系统的整体性能至关重要，特别是在寻找最佳连接顺序方面。在高效的*DPccp*算法的基础上，该算法使用动态规划，引入了一种新算法*DPhyp*来有效处理复杂的连接谓词。通过将查询图建模为超图并分析其连接的子图，*DPhyp*改进了非内连接的优化，与以前的方法相比提供了显著的性能提升。

随着硬件的进步，高复杂度算法正变得实用。即使一些算法可能不在多项式时间内运行，现代计算机也可以高效地处理大型NP完全问题。动态规划技术虽然仍然是指数级的，但对于中等实例大小越来越可行，通常实现O(2^n)的时间复杂度。

然而，在使用超图算法时，参与连接的表数应保持在合理的范围内，以避免潜在的性能问题。对TPC-C中的复杂连接操作进行了性能比较，启用和禁用超图优化。详细结果如下图所示：

![](media/363e8bb8f055a7611cb80e0c62b2fa2a.png)

图7-24. 超图算法对典型TPC-C SQL工作负载的影响。

从图中可以明显看出，启用超图算法会导致执行时间为0.88秒，而禁用它会将时间减少到0.03秒。这表明使用超图算法的显著性能影响。在许多情况下，超图的开销可能是巨大的。如果MySQL的默认执行计划导致性能缓慢，超图算法可能会提供有价值的改进。

让我们通过检查下图中的*perf*火焰图来进一步分析超图算法的性能。

![](media/6b03a6feb6dee8e562c0970cb5417d6b.png)

图7-25. 超图算法的典型火焰图。

从图中可以明显看出，超图算法（hypergraph\*）消耗了大量的计算。目前在单线程模式下运行，优化超图算法有很大的潜力。

由于MySQL缺乏查询计划缓存，使用超图算法构建最佳执行计划非常耗时，对其在生产环境中的有效使用构成了挑战。

值得注意的是，AI也可以用于优化执行计划，如第5.20.2节所讨论。

## 7.4 使用Binlog压缩节省成本

从MySQL 8.0.20开始，支持binlog压缩，但默认情况下禁用。可以使用*binlog_transaction_compression*参数启用它，并且可以使用*binlog_transaction_compression_level_zstd*参数调整*zstd*压缩级别，默认级别为3。

使用同一数据中心内的Group Replication集群，使用BenchmarkSQL检查binlog压缩对TPC-C吞吐量和并发性的影响。主节点和辅助节点都配置了*binlog_transaction_compression*参数。具体测试结果如下图所示：

<img src="media/image-20240829101915930.png" alt="image-20240829101915930" style="zoom:150%;" />

图7-26. binlog压缩对BenchmarkSQL性能的影响。

从图中可以明显看出，启用binlog压缩显著影响吞吐量，出现明显的波动。

下一步是比较压缩前后的binlog大小。具体细节如下图所示：

<img src="media/image-20240829101936734.png" alt="image-20240829101936734" style="zoom:150%;" />

图7-27. BenchmarkSQL测试后binlog压缩的效果。

从图中可以明显看出，binlog压缩对TPC-C测试有显著的积极影响。值得注意的是，设置*binlog_row_image=minimal*可以显著减小binlog大小，但对性能的影响较小。具体细节如下图所示：

<img src="media/image-20240829101956608.png" alt="image-20240829101956608" style="zoom:150%;" />

图7-28. *binlog_row_image=minimal*对BenchmarkSQL性能的影响。

最后，让我们检查*binlog_row_image=minimal*和*binlog_row_image=full*之间binlog大小的比较。具体细节如下图所示：

<img src="media/image-20240829102020166.png" alt="image-20240829102020166" style="zoom:150%;" />

图7-29. BenchmarkSQL测试后*binlog_row_image=minimal*的效果。

从图中可以看出，设置*binlog_row_image=minimal*也可以显著减小binlog的大小。

总的来说，MySQL 8.0提供了有效的解决方案来解决binlog消耗大量I/O空间的问题。用户可以利用binlog压缩，并在可行的情况下，通过使用*binlog_row_image=minimal*进一步减小binlog大小以节省存储成本。重要的是要注意，压缩比可能因不同应用程序而异。

[下一页](Chapter8_zh.md)
