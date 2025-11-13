# 4.3 算法

本节涵盖了 MySQL 中各种问题解决算法,包括搜索算法、排序算法、贪心算法、动态规划、摊销分析和 Paxos 系列算法。

## 4.3.1 搜索算法

在计算机科学中,搜索算法旨在在特定数据结构或搜索空间内定位信息 [45]。

MySQL 通常采用几种搜索算法,包括二分查找、红黑树搜索、B+ 树搜索和哈希搜索。二分查找对于搜索没有特殊特征的数据集或处理较小数据集时很有效。例如,二分查找用于确定记录在活动事务列表中是否可见,详见下文:

```c++
/** Check whether the changes by id are visible.
  @param[in]    id      transaction id to check against the view
  @param[in]    name    table name
  @return whether the view sees the modifications of id. */
  [[nodiscard]] bool changes_visible(trx_id_t id,
                                     const table_name_t &name) const {
    ut_ad(id > 0);
    if (id < m_up_limit_id || id == m_creator_trx_id) {
      return (true);
    }
    check_trx_id_sanity(id, name);
    if (id >= m_low_limit_id) {
      return (false);
    } else if (m_ids.empty()) {
      return (true);
    }
    const ids_t::value_type *p = m_ids.data();
    return (!std::binary_search(p, p + m_ids.size(), id));
  }
```

B+ 树搜索在 *btr/btr0btr.cc* 中实现,是一种成熟的方法,非常适合索引场景。哈希查找是 MySQL 中另一种普遍的搜索方法,经常被使用,例如在 Group Replication 中用于认证数据库,详见下文:

```c++
bool Certifier::add_item(const char *item, Gtid_set_ref *snapshot_version,
                         int64 *item_previous_sequence_number) {
  DBUG_TRACE;
  mysql_mutex_assert_owner(&LOCK_certification_info);
  bool error = true;
  std::string key(item);
  Certification_info::iterator it = certification_info.find(key);
  snapshot_version->link();
  if (it == certification_info.end()) {
    std::pair<Certification_info::iterator, bool> ret =
        certification_info.insert(
            std::pair<std::string, Gtid_set_ref *>(key, snapshot_version));
    error = !ret.second;
  } else {
    *item_previous_sequence_number =
        it->second->get_parallel_applier_sequence_number();
    if (it->second->unlink() == 0) delete it->second;
    it->second = snapshot_version;
    error = false;
  }
  ...
  return error;
}
```

搜索算法的选择是灵活的,通常根据性能瓶颈进行调整。例如,前面提到的基于哈希的搜索算法成为 Group Replication 应用程序线程操作的主要瓶颈,如下面的具体性能火焰图所示:

![](media/48058e6b17b3f83cdc5479c38ad16648.png)

图 4-11. 基于哈希的搜索算法中瓶颈的示例。

图中显示,*Certify::add_item* 中的哈希搜索占总时间的一半,突出了一个显著的瓶颈。这表明需要探索替代的搜索算法。关于潜在解决方案的更多细节将在后续章节中讨论。

## 4.3.2 排序算法

在计算机科学中,排序算法将列表的元素排列成特定顺序,最常用的顺序是数字和词典序,升序或降序。高效的排序对于优化其他算法(如搜索和合并算法)的性能至关重要,这些算法需要排序的输入数据 [45]。

常用的排序算法本身并没有优劣之分;每种都有其自己的用途。例如,考虑 C++ 标准库中的 *std::sort*。它根据数据的特征采用不同的排序算法。当数据大部分有序时,它使用 *插入排序*。详见下文:

```c++
// sort
template <typename _RandomAccessIterator, typename _Compare>
_GLIBCXX20_CONSTEXPR inline void __sort(_RandomAccessIterator __first,
                                        _RandomAccessIterator __last,
                                        _Compare __comp) {
  if (__first != __last) {
    std::__introsort_loop(__first, __last, std::__lg(__last - __first) * 2,
                          __comp);
    std::__final_insertion_sort(__first, __last, __comp);
  }
}
```

*插入排序* 用于数据大部分有序的情况,因为它在这种情况下的时间复杂度接近线性。更重要的是,*插入排序* 顺序访问数据,这显著提高了缓存效率。这种缓存友好的特性使插入排序非常适合对少量数据进行排序。

例如,在 *std::sort* 函数中,调用 *__introsort_loop* 函数时,如果元素数量小于或等于 16,则跳过排序,并将控制权返回给 *sort* 函数。然后 *sort* 函数利用 *插入排序* 进行排序。

```c++
/**
   *  @doctodo
   *  This controls some aspect of the sort routines.
  */
  enum { _S_threshold = 16 };
/// This is a helper function for the sort routine.
template <typename _RandomAccessIterator, typename _Size, typename _Compare>
_GLIBCXX20_CONSTEXPR void __introsort_loop(_RandomAccessIterator __first,
                                           _RandomAccessIterator __last,
                                           _Size __depth_limit,
                                           _Compare __comp) {
  while (__last - __first > int(_S_threshold)) {
    if (__depth_limit == 0) {
      std::__partial_sort(__first, __last, __last, __comp);
      return;
    }
    --__depth_limit;
    _RandomAccessIterator __cut =
        std::__unguarded_partition_pivot(__first, __last, __comp);
    std::__introsort_loop(__cut, __last, __depth_limit, __comp);
    __last = __cut;
  }
}
```

在 *__introsort_loop* 函数中,如果递归深度超过阈值,则使用基于 *堆* 数据结构并提供稳定性能的 *partial_sort* 函数。总体而言,*__introsort_loop* 的主要部分采用了改进版本的 *快速排序*,消除了左递归并减少了函数调用的开销。

讨论排序算法不仅是因为 MySQL 广泛使用这些标准库排序算法,还要从它们的实现中获得优化策略的见解。这涉及根据具体情况选择不同的算法,以利用它们各自的优势,从而提高算法的整体性能。

在 MySQL 代码中,应用了类似的优化原则。例如,在 *sort_buffer* 函数中,当 *key_len* 值很小时,它使用适合短键的 *Mem_compare* 函数。当 *prefilter_nth_element* \> 0 时,它采用 *nth_element*(类似于 *快速排序* 的分区思想),选择所需的元素进行后续排序。

```c++
size_t Filesort_buffer::sort_buffer(Sort_param *param, size_t num_input_rows,
                                    size_t max_output_rows) {
  ...
  if (num_input_rows <= 100) {
    if (key_len < 10) {
      param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_SORT;
      if (prefilter_nth_element) {
        nth_element(it_begin, it_begin + max_output_rows - 1, it_end,
                    Mem_compare(key_len));
        it_end = it_begin + max_output_rows;
      }
      sort(it_begin, it_end, Mem_compare(key_len));
      ...
      return std::min(num_input_rows, max_output_rows);
    }
    param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_SORT;
    if (prefilter_nth_element) {
      nth_element(it_begin, it_begin + max_output_rows - 1, it_end,
                  Mem_compare_longkey(key_len));
      it_end = it_begin + max_output_rows;
    }
    sort(it_begin, it_end, Mem_compare_longkey(key_len));
    ...
    return std::min(num_input_rows, max_output_rows);
  }
  param->m_sort_algorithm = Sort_param::FILESORT_ALG_STD_STABLE;
  // Heuristics here: avoid function overhead call for short keys.
  if (key_len < 10) {
    if (prefilter_nth_element) {
      nth_element(it_begin, it_begin + max_output_rows - 1, it_end,
                  Mem_compare(key_len));
      it_end = it_begin + max_output_rows;
    }
    stable_sort(it_begin, it_end, Mem_compare(key_len));
    ...
  } else {
    ...
  }
  return std::min(num_input_rows, max_output_rows);
}
```

总的来说,在使用排序算法时,关键原则是应用灵活性和缓存友好性。通过考虑算法的复杂度,可以找到最合适的排序算法。

## 4.3.3 贪心算法

贪心算法遵循在每个阶段做出局部最优选择的启发式方法。尽管它通常不会为许多问题产生最优解,但它可以在合理的时间内产生接近全局最优解的局部最优解。

基于 SQL 生成执行计划中最具挑战性的问题之一是选择连接顺序。目前,MySQL 的执行计划使用简单的贪心算法进行连接顺序,在某些场景中表现良好。详见下文:

```c++
/**
  Find a good, possibly optimal, query execution plan (QEP) by a greedy search.
  ...
  @note
    The following pseudocode describes the algorithm of 'greedy_search':
    @code
    procedure greedy_search
    input: remaining_tables
    output: pplan;
    {
      pplan = <>;
      do {
        (t, a) = best_extension(pplan, remaining_tables);
        pplan = concat(pplan, (t, a));
        remaining_tables = remaining_tables - t;
      } while (remaining_tables != {})
      return pplan;
    }
  ...
*/
bool Optimize_table_order::greedy_search(table_map remaining_tables) {
```

确定最优连接顺序是一个 NP 难问题,对于复杂的连接来说,在计算资源方面成本过高。因此,MySQL 使用贪心算法。尽管这种方法可能并不总是产生绝对最佳的连接顺序,但它在计算效率和良好的整体性能之间取得了平衡。

在某些连接操作中,贪心算法的连接顺序选择可能导致性能不佳,这可能解释了为什么用户偶尔会批评 MySQL 的性能。

## 4.3.4 动态规划

动态规划通过递归地将复杂问题分解为更简单的子问题来简化问题。如果可以通过解决其子问题并组合它们的解决方案来最优地解决问题,则该问题表现出最优子结构。此外,如果子问题嵌套在更大的问题中并且动态规划方法适用,则更大问题的值与子问题的值之间存在关系。应用动态规划的两个关键属性是最优子结构和重叠子问题 [45]。

在执行计划优化的上下文中,MySQL 8.0 已经探索使用动态规划算法来确定最优连接顺序。这种方法可以极大地改进复杂连接的性能,尽管它在当前实现中仍然是实验性的。

重要的是要注意,由于成本估计可能不准确,动态规划算法确定的连接顺序可能并不总是真正的最优解。动态规划算法通常提供最佳计划,但可能具有高计算开销,并且由于不正确的成本估计而可能导致大成本 [55]。要更深入地了解所涉及的复杂机制,读者可以参考论文"Dynamic Programming Strikes Back" [35]。

## 4.3.5 摊销分析

在计算机科学中,摊销分析是一种分析算法复杂度的方法,特别是执行算法需要多少资源(如时间或内存)。摊销分析的动机是考虑最坏情况运行时间可能过于悲观。相反,摊销分析对一系列操作的运行时间进行平均 [45]。

本节讨论应用摊销分析来解决 MySQL 问题。虽然它与传统的摊销分析不同,但基本原理相似,并且在解决 MySQL 性能问题方面有许多实际应用。例如,在 Group Replication 的重构过程中,在多主场景中清理过时的认证数据库信息时观察到显著的抖动。下图显示了使用 Group Replication 在 MySQL 多主模式下的读写测试,使用 SysBench 在 100 并发级别下进行 300 秒的测试。

<img src="media/image-20240829083549751.png" alt="image-20240829083549751" style="zoom:150%;" />

图 4-12. MySQL Group Replication 中的性能波动。

为了解决这个问题,采用了摊销策略来清理过时的认证数据库信息。具体细节见下图:

<img src="media/image-20240829083610439.png" alt="image-20240829083610439" style="zoom:150%;" />

图 4-13. 增强的 MySQL Group Replication 中消除的性能波动。

MySQL 经历严重的性能波动主要是由于每 60 秒清理一次过时的认证数据库信息。在重新设计策略为每 0.2 秒清理一次后,导致毫秒级(ms)的性能波动,这些波动在测试期间变得不可感知。改进版本的 MySQL 消除了突然的性能下降,主要是通过应用摊销方法来减少显著的波动。

重要的是要注意,每 0.2 秒清理一次需要每个 MySQL 节点以约 0.2 秒的间隔及时发送其 GTID 信息。使用传统的 Multi-Paxos 算法很难满足这种高频发送,因为这些算法通常需要领导者在一段时间内保持稳定。因此,基于单领导者的 Multi-Paxos 算法难以有效处理突然的性能下降,因为底层算法缺乏对这种频繁操作的支持。

## 4.3.6 Paxos

在面对任意故障时维护副本之间的一致性一直是分布式系统几十年来的一个主要焦点。虽然天真的解决方案可能适用于简单的情况,但它们通常无法提供通用解决方案。Paxos 是一系列旨在在不可靠或易出错的处理器之间实现共识的协议。共识涉及在多个参与者之间就单个结果达成一致,当出现故障或通信问题时,这项任务变得具有挑战性。Paxos 家族被广泛认为是在三个或更多副本之间实现共识的唯一经过验证的解决方案,因为它解决了在 2F + 1 个副本之间达成一致同时容忍最多 F 个故障的一般问题。这使得 Paxos 成为状态机复制(SMR)的基本组成部分,也是确保集群环境中高可用性的最简单算法之一 [26]。

Paxos 的数学基础植根于集合论,特别是大多数集合的交集必须是非空的原则。这个原则是在复杂场景中解决高可用性问题的关键。

然而,纯粹的 Paxos 协议在满足实际业务需求方面往往不足,特别是在具有显著网络延迟的环境中。为了解决这个问题,已经开发了各种策略和增强功能,导致了几种 Paxos 变体,例如 Multi-Paxos、Raft 和 Group Replication 中的 Mencius。下图展示了 Multi-Paxos 的理想化序列,其中稳定的领导者允许提议消息在单个往返时间(RTT)内实现共识并将结果返回给用户 [45]。

```
Client      Servers
   X-------->|  |  |  Request
   |         X->|->|  Accept!(N,I+1,W)
   |         X<>X<>X  Accepted(N,I+1)
   |<--------X  |  |  Response
   |         |  |  |
```

从 MySQL 8.0.27 开始,Group Replication 整合了两种 Paxos 变体算法:Mencius 和传统的 Multi-Paxos。Mencius 旨在解决 Paxos 中多个领导者的挑战,最初是为广域网应用程序开发的。它允许每个节点直接发送提议消息,使任何 MySQL 从节点都可以随时与集群通信,而不会给单个 Paxos 领导者带来负担。这种方法支持读写的一致性,并改进了 Group Replication 的多主功能。相比之下,Multi-Paxos 算法采用单领导者方法。非领导者节点必须在向集群发送消息之前从领导者请求序列号。频繁地向领导者请求可能会造成瓶颈,可能限制性能。

要使用 Mencius 或 Multi-Paxos 实现高吞吐量,利用批处理和流水线技术是必不可少的。这些优化在状态机复制中常用 [49],可以显著提高性能。例如,下图展示了批处理如何在局域网场景中提高 Group Replication 吞吐量。

<img src="media/image-20240829083656994.png" alt="image-20240829083656994" style="zoom:150%;" />

图 4-14. 批处理对局域网环境中 Paxos 算法性能的影响。

[下一页](Chapter4_4_zh.md)
