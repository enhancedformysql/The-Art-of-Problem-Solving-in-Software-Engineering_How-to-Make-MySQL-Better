# 4.2 数据结构

本节探讨MySQL中的基本数据结构,包括数组、链表、队列、堆、哈希表、红黑树、B+树和混合数据结构。这些数据结构本身并不具有优劣之分;它们的有效性取决于它们对特定系统架构和实际数据特征的应用。

## 4.2.1 数组

数组由按特定顺序排列的元素组成,通常是相同类型。通过整数索引访问元素以指定所需的项。数组通常使用连续内存分配实现,可以是固定长度或可调整大小的 [45]。在 MySQL 中,常用的数组包括动态向量和固定长度数组,具体选择取决于特定需求。向量可以动态调整大小,而固定长度数组具有预定的大小。

在MySQL InnoDB存储引擎中,MVCC ReadView使用类似于向量的数据结构来存储所有活动事务的事务ID。这个动态数组支持不同的长度,适应活动事务列表的变化,尽管大小会波动。对于读已提交事务隔离级别,每次读操作都使用自己的ReadView。

以下是关于ReadView对象的一些细节。

```c++
private:
  // Disable copying
  ReadView(const ReadView &);
  ReadView &operator=(const ReadView &);
 private:
  /** The read should not see any transaction with trx id >= this
  value. In other words, this is the "high water mark". */
  trx_id_t m_low_limit_id;
  /** The read should see all trx ids which are strictly
  smaller (<) than this value.  In other words, this is the
  low water mark". */
  trx_id_t m_up_limit_id;
  /** trx id of creating transaction, set to TRX_ID_MAX for free
  views. */
  trx_id_t m_creator_trx_id;
  /** Set of RW transactions that was active when this snapshot
  was taken */
  ids_t m_ids;
  /** The view does not need to see the undo logs for transactions
  whose transaction number is strictly smaller (<) than this value:
  they can be removed in purge if not needed by other views */
  trx_id_t m_low_limit_no;
```

变量 *m_ids* 是 *ids_t* 类型的数据结构,与 *std::vector* 非常相似。更多细节见下:

```c++
  /** This is similar to a std::vector but it is not a drop
  in replacement. It is specific to ReadView. */
  class ids_t {
    typedef trx_ids_t::value_type;
    /**
    Constructor */
    ids_t() : m_ptr(), m_size(), m_reserved() {}
    /**
    Destructor */
    ~ids_t() { ut::delete_arr(m_ptr); }
    /** Try and increase the size of the array. Old elements are copied across.
    It is a no-op if n is < current size.
    @param n            Make space for n elements */
    void reserve(ulint n);
```

固定长度数组有实际价值吗?在MySQL中,缓冲池块使用固定长度数组组织。详细信息如下:

```c++
/** @brief The buffer pool structure.
NOTE! The definition appears here only for other modules of this
directory (buf) to see it. Do not use from outside! */
struct buf_pool_t {
  ...
  /** Number of buffer pool chunks */
  volatile ulint n_chunks;
  /** New number of buffer pool chunks */
  volatile ulint n_chunks_new;
  /** buffer pool chunks */
  buf_chunk_t *chunks;
  /** old buffer pool chunks to be freed after resizing buffer pool */
  buf_chunk_t *chunks_old;
  /** Current pool size in pages */
  ulint curr_size;
  /** Previous pool size in pages */
  ulint old_size;
  /** Size in pages of the area which the read-ahead algorithms read
  if invoked */
  page_no_t read_ahead_area;
```

上面定义了数组名称和类型,下面根据数组的成员类型进行动态内存分配。

```c++
 buf_pool->chunks = reinterpret_cast<buf_chunk_t *>(ut::zalloc_withkey(
        UT_NEW_THIS_FILE_PSI_KEY, buf_pool->n_chunks * sizeof(*chunk)));
```

从实际角度来看,利用固定长度数组可以提供显著的性能优势。它们的稳定性防止了因内存重新分配而导致的性能波动,并且它们的缓存友好性进一步提高了效率。后续章节将包含几个示例,其中使用固定长度数组显著提高了性能或缓解了性能瓶颈。

## 4.2.2 链表

链表(或简称"列表")是数据元素的线性集合,称为节点,其中每个节点包含一个值和对序列中下一个节点的引用。与数组相比,链表的主要优势是它们在插入和删除元素时的效率,无需重新定位整个列表。然而,像随机访问特定元素这样的操作通常在链表中比数组慢 [45]。

MySQL 通常使用标准库中的列表,它通常实现双向链表以便于插入和删除,尽管它通常查询性能不佳。在主流 NUMA 架构中,由于非连续的内存访问模式,链表通常查询效率低下。因此,链表最适合作为辅助数据结构或用于涉及较小数据量的场景。

在大型项目中,为了避免链表中内存分配的不可预测性,可以使用内存池进行更好的管理。下面是 *undo* 使用的列表数据结构。随着 *undo* 列表的增长,MVCC 效率会显著降低。

```c++
  using Recs = std::list<rec_t, mem_heap_allocator<rec_t>>;
  ...
  /** Undo recs to purge */
  Recs *recs;
```

## 4.2.3 队列

在计算机科学中,队列是按序列组织的实体集合,其中实体可以在一端添加,并从另一端删除。将元素添加到后端的操作称为入队,从前端删除元素称为出队。这使得队列成为先进先出(FIFO)数据结构,意味着首先添加的元素将是第一个被删除的。换句话说,元素按照添加的顺序进行处理。

队列是线性数据结构或顺序集合,通常用于计算机程序。它们可以使用循环缓冲区或链表实现。在MySQL中,队列通常封装了额外的功能,例如同步队列和双端队列,用于FIFO处理需求。例如,下面显示的 *incoming* 成员使用同步队列来存储 Group Replication 的应用程序包,作为与 Paxos 网络交互和中继日志磁盘写入相关的数据的缓存。当中继日志写入滞后时,这种缓冲有助于管理数据。

```c++
  /* The incoming event queue */
  Synchronized_queue<Packet *> *incoming;
```

双端队列在各种应用程序中常用。例如,MySQL 利用 *std::deque* 来实现通用的 *mem_root_deque*。详细信息如下:

```c++
/**
  A (partial) implementation of std::deque allocating its blocks on a MEM_ROOT.
  This class works pretty much like an std::deque with a Mem_root_allocator,
  and used to be a forwarder to it. However, libstdc++ has a very complicated
  implementation of std::deque, leading to code blowup (e.g., operator[] is
  23 instructions on x86-64, including two branches), and we cannot easily use
  libc++ on all platforms. This version is instead:
   - Optimized for small, straight-through machine code (few and simple
     instructions, few branches).
   - Optimized for few elements; in particular, zero elements is an important
     special case, much more so than 10,000.
  ...
 */
template <class Element_type>
class mem_root_deque {
```

## 4.2.4 堆

在计算机科学中,堆是基于树的数据结构,它维护堆属性,通常使用数组实现 [45]。它作为称为优先队列的抽象数据类型的有效实现。无论其底层实现如何,优先队列通常被称为"堆"。在堆中,具有最高(或最低)优先级的元素总是在根部。然而,与排序结构不同,堆是部分有序的。

当需要重复访问和删除具有最高或最低优先级的元素,或者当根节点的插入和删除频繁发生时,堆特别有用。优先队列通常使用堆实现,也在 MySQL 中使用。例如,在 MySQL 8.0.34 中,数据结构 *purge_pg_t*(详细信息如下)利用标准库中的 *priority_queue* 来高效查找最旧的事务 ID。

```c++
typedef std::priority_queue<
    TrxUndoRsegs, std::vector<TrxUndoRsegs, ut::allocator<TrxUndoRsegs>>,
    TrxUndoRsegs>
purge_pq_t;
```

从数学角度来看,堆数据结构具有平衡的树结构,理论树级别最少。然而,在现代架构中,它们存在显著的缺点。堆具有非顺序访问模式,从根移动到叶子,这不是缓存友好的。这使得堆适用于相对较小的数据集,但随着数据规模的扩大效率会降低。缓存访问效率低下可能解释了为什么基于堆的算法(如 *堆排序*)在平均性能上不如 *快速排序*,尽管它们有理论优势。

## 4.2.5 哈希表

哈希表,也称为哈希映射,是一种数据结构,旨在基于键快速检索值。它使用哈希函数将键映射到数组中的索引,允许平均常数时间访问。哈希表通常用于字典、缓存和数据库索引。尽管它们高效,但哈希冲突可能会降低性能,诸如链接和开放寻址等技术用于管理它们。

传统哈希表的主要优势在于其快速的查询速度。然而,由于存储在哈希槽中的内存指针分散,它们可能不是非常缓存友好的。这种分散可能导致频繁访问操作期间效率降低。为了解决这个问题,出现了像 Google 的 Swiss Table 这样的解决方案,它采用缓存友好且更高效的方法来使比较和插入更快。然而,这样的解决方案并不是通用的,并且不特别适合数据库使用。

在 MySQL 中,哈希表被广泛使用,利用 STL 类型如 *unordered_set* 和 *unordered_map*,以及针对特定用例量身定制的自定义哈希表。例如,用于缓冲池页面管理的 *hash_table_t* 数据类型就是这种专门实现的示例。

```c++
  /** Hash table of buf_page_t or buf_block_t file pages, buf_page_in_file() ==
  true, indexed by (space_id, offset).  page_hash is protected by an array of
  mutexes. */
  hash_table_t *page_hash;
```

*hash_table_t* 的数据成员如下:

```c++
/* The hash table structure */
class hash_table_t {
 public:
  hash_table_t(size_t n) {
    const auto prime = ut::find_prime(n);
    cells = ut::make_unique<hash_cell_t[]>(prime);
    set_n_cells(prime);

    /* Initialize the cell array */
    hash_table_clear(this);
  }
  ~hash_table_t() { ut_ad(magic_n == HASH_TABLE_MAGIC_N); }
  /** Returns number of cells in cells[] array.
   If type==HASH_TABLE_SYNC_RW_LOCK it can be used:
  - without any latches to peek a value, before hash_lock_[sx]_confirm
  - when holding S-latch for at least one n_sync_obj to get the "real" value
  @return value of n_cells
  */
  size_t get_n_cells() { return n_cells.load(std::memory_order_relaxed); }
  /** Returns a helper class for calculating fast modulo n_cells.
   If type==HASH_TABLE_SYNC_RW_LOCK it can be used:
  - without any latches to peek a value, before hash_lock_[sx]_confirm
  - when holding S-latch for at least one n_sync_obj to get the "real" value */
  const ut::fast_modulo_t get_n_cells_fast_modulo() {
    return n_cells_fast_modulo.load();
  }
  ...
```

请注意,基于页面的缓冲池具有较低的缓存效率,并且页面转换表是可扩展性瓶颈 [64]。

Group Replication 的认证数据库使用 *std::unordered_map* 哈希表来处理大量的认证信息。

```c++
  typedef std::unordered_map<
      std::string, Gtid_set_ref *, std::hash<std::string>,
      std::equal_to<std::string>,
      Malloc_allocator<std::pair<const std::string, Gtid_set_ref *>>>
      Certification_info;
  ...
  /**
    Certification database.
  */
  Certification_info certification_info;
```

尽管利用了像哈希表这样的高效数据结构,认证数据库由于频繁访问元素而面临性能挑战。这个问题的出现是因为认证数据库中的内存访问模式是非连续的,导致内存访问效率低下。因此,虽然哈希表提供了优势,但在这种情况下它们并不总是性能的最佳选择。

## 4.2.6 红黑树

在计算机科学中,红黑树是一种自平衡二叉搜索树,以高效存储和检索有序数据而闻名。红黑树中的每个节点都有一个额外的"颜色"位,通常是红色或黑色,这有助于维护树的平衡结构。MySQL 经常使用 STL(标准模板库)中的 *map*,它实现了红黑树以保持顺序。相比之下,STL 中的 *unordered_map* 是一个哈希表,不维护顺序,这就是为什么它被称为"无序映射"。

下面的页面展示了使用 map 数据结构进行高效查找、修改和顺序遍历。

```c++
 /* Assuming a page size, read the space_id from each page and store it
  in a map. Find out which space_id is agreed on by majority of the
  pages.  Choose that space_id. */
  for (uint32_t page_size = UNIV_ZIP_SIZE_MIN; page_size <= UNIV_PAGE_SIZE_MAX;
       page_size <<= 1) {
    /* map[space_id] = count of pages */
    typedef std::map<space_id_t, ulint, std::less<space_id_t>,
                     ut::allocator<std::pair<const space_id_t, ulint>>>
        Pages;
    Pages verify;
    ulint page_count = 64;
    ulint valid_pages = 0;
    /* Adjust the number of pages to analyze based on file size */
    while ((page_count * page_size) > file_size) {
      --page_count;
}
```

MySQL 还实现了一个针对其特定需求量身定制的红黑树。例如,下面的代码片段显示了 *SEL_ROOT*,一个用于存储键范围的红黑树。

```c++
/**
  A graph of (possible multiple) key ranges, represented as a red-black
  binary tree. There are three types (see the Type enum); if KEY_RANGE,
  we have zero or more SEL_ARGs, described in the documentation on SEL_ARG.
  As a special case, a nullptr SEL_ROOT means a range that is always true.
  This is true both for keys[] and next_key_part.
*/
class SEL_ROOT {
 ...
  /**
    Insert the given node into the tree, and update the root.
    @param key The node to insert.
  */
  void insert(SEL_ARG *key);
  /**
    Delete the given node from the tree, and update the root.
    @param key The node to delete. Must exist in the tree.
  */
  void tree_delete(SEL_ARG *key);
  /**
    Find best key with min <= given key.
    Because of the call context, this should never return nullptr to get_range.
    @param key The key to search for.
  */
  SEL_ARG *find_range(const SEL_ARG *key) const;
  ...
```

通常,红黑树提供了诸如低插入和更新成本以及支持顺序遍历等优势。然而,它们的非顺序内存访问可能会降低缓存效率,使它们在高性能、计算密集型任务中不太理想。

## 4.2.7 B+ 树

B+ 树是一种 m 叉树,其特征是每个节点具有大量子节点,包括根、内部节点和叶子。根可以是叶子,也可以是具有两个或更多子节点的节点 [45]。

B+ 树在面向块的存储上下文(如文件系统)中表现出色,因为它们的高扇出(通常每个节点约 100 个或更多指针)。这种高扇出减少了定位元素所需的 I/O 操作次数,使 B+ 树在数据无法放入内存并且必须从磁盘读取时特别高效。

然而,B 树中操作的并发控制被认为是一个困难的主题,有许多微妙之处和特殊情况 [70]。有关此主题的详细信息,请参阅论文"A Survey of B-Tree Locking Techniques"。

InnoDB 采用 B+ 树进行索引,利用它们能够根据树的深度确保固定最大读取次数的能力,从而高效扩展。有关 MySQL 中 B+ 树实现的具体详细信息,请参阅文件 *btr/btr0btr.cc*。

## 4.2.8 混合数据结构

在各种应用场景中,依赖单一数据结构可能并不总能获得最佳性能。结合不同的数据结构通常可以带来显著的改进。例如,在 MySQL 中,MVCC ReadView 最初使用动态数组(*vector*)来维护活动事务列表,利用二分查找进行查询。然而,在高并发环境中,这个列表可能会变得过长,在 NUMA 环境中效率较低。为了缓解这个问题,采用了混合方法:最近的事务存储在静态数组中以便快速访问,而长时间运行的事务则放置在动态数组中。这种由多个变量管理的双数组策略提高了访问速度和效率。更多详细信息见下:

![](media/a54faa33502b8c17066b1e2af09bdbb0.png)

图 4-8. 适用于 MVCC ReadView 中活动事务列表的新混合数据结构。

为了更好地说明混合数据结构的概念,请考虑以下示例:

![](media/6ce64b147aa9f2c6635b18c08715437d.png)

图 4-9. 活动事务列表新混合数据结构的详细示例。

活动事务列表长度为 17,每个事务 ID 需要 8 个字节。使用动态数组(向量)存储这将至少需要 17 \* 8 = 136 个字节。通过切换到混合数据结构,大多数事务 ID 使用 3 字节位表示存储在静态数组中,而动态数组保存两个事务 ID(1 和 3),占用 16 个字节。此外,两个辅助变量消耗 16 个字节。因此,混合数据结构总共 3 + 16 + 16 = 35 个字节,比原始方法少 101 个字节。

关于查询效率,混合数据结构提供了显著的改进。例如,要检查事务 ID=24 是否在活动事务列表中:

- 在原始方法中,需要二分查找,时间复杂度为 O(log n)。
- 使用混合结构,使用最小短事务 ID 作为基线允许通过静态数据直接查询,实现 O(1) 的时间复杂度。

在 NUMA 环境中,如下图所示,可以看到简单地更改数据结构可以显著提高高并发条件下的 TPC-C 吞吐量,极大地缓解了与 MVCC ReadView 相关的可扩展性问题。

<img src="media/image-20240829083417393.png" alt="image-20240829083417393" style="zoom:150%;" />

图 4-10. NUMA 中新混合数据结构的性能改进。

[下一页](Chapter4_3_zh.md)
