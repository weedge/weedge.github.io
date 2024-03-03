---
author: "weedge"
title: "译：利用 GPU 上的大规模并行hashmap最大限度地提高性能"
date: 2023-11-02T10:26:23+08:00
tags: [
	"oneday", "gpu", "hashmap"
]
categories: [
	"技术",
]

---



![img](https://github.com/weedge/mypic/raw/master/oneday/maximizing-performance-with-massively-parallel-hash-maps-on-gpus/1.png)

数十年的计算机科学历史一直致力于设计有效存储和检索信息的解决方案。hashmap（或hashtable）是一种流行的信息存储数据结构，因为它们可以保证元素插入和检索的恒定时间。 

然而，尽管hashmap很流行，但很少在 GPU 加速计算的背景下进行讨论。虽然 GPU 以其大量线程和计算能力而闻名，但其极高的内存带宽可以加速许多数据结构（例如hashmap）。 

这篇文章将介绍哈hashmap的基础知识以及它们的内存访问模式如何使其非常适合 GPU 加速。我们将介绍[cuCollections](https://github.com/NVIDIA/cuCollections)，这是一个用于并发数据结构（包括hashmap）的新开源 CUDA C++ 库。 

最后，如果有兴趣在应用程序中使用 GPU 加速的哈希表，我们提供了多列关系连接算法的示例实现case。RAPIDS cuDF 集成了 GPU 哈希表，这有助于为数据科学工作负载实现令人难以置信的加速。要了解更多信息，请参阅GitHub 上的[rapidsai/cudf](https://github.com/rapidsai/cudf); 以及使用示例case [使用 Dask 和 RAPIDS 加速 TF-IDF 进行自然语言处理](https://medium.com/rapids-ai/accelerating-tf-idf-for-natural-language-processing-with-dask-and-rapids-6f6e416429df)。 

还可以将 cuCollections 用于表格数据处理之外的许多用例，例如推荐系统、流压缩、图形算法、基因组学和稀疏线性代数运算。请参阅[Pinterest 通过切换推荐系统的 GPU 加速将主页订阅参与度提高 16%](https://blogs.nvidia.com/blog/2022/08/04/pinterest-gpu-acceleration-recommenders/)了解更多信息。

<!--more-->

## Hash map基础知识

[Hashmap](https://en.wikipedia.org/wiki/Hash_table)是关联(*associative*)容器，这意味着它们存储pair<key,val>对，其中keymap到关联val，从而可以通过查找key来检索val。例如，可以使用hashmap来实现电话簿，方法是使用个人姓名作为key，使用电话号码作为关联值。 

Hashmap与其他关联容器的不同之处在于，插入或检索等操作的平均成本是恒定的。[`std::map`](https://en.cppreference.com/w/cpp/container/map)C++ 标准模板库中的map不是hashtable，而是通常以二叉搜索树的形式实现。[`std::unordered_map`](https://en.cppreference.com/w/cpp/container/unordered_map)更类似于与此讨论相关的hashtable。就本文而言，hashtable和hashmap之间没有区别。这两个术语将在全文中互换使用。 

### 单值与多值比较

讨论哈希表时的一个重要区别是是否允许重复key。单值哈希表或hashmap要求key是唯一的（例如，`std::unordered_map`），而多值哈希表或哈希多重map允许重复的key（例如，`std::unordered_multimap`）。 

使用电话簿类比，后者指的是一个人可以拥有多个电话号码的情况。例如，电话簿可能具有(k=Alice, v=408-555-0148)和具有另一个值(k=Alice, v=408-555-3847) 的重复key。 

### 存储和检索

从概念上讲，哈希表由一组桶组成，其中每个桶可以保存一个或多个key值对。为了将新的对插入到map中，对key应用哈希函数以产生哈希值。然后使用该哈希值来选择其中一个存储桶。如果存储桶可用，则该对存储在该存储桶中。 

例如，要插入对(Alice, 408-555-0148)，可以对keyhash(Alice)=4 进行哈希处理，以获取其哈希值，并选择位置 4 处的存储桶来存储该对。稍后，要检索与Alice关联的值，可以使用相同的哈希函数hash(Alice)再次选择位置 4 处的存储桶并检索之前存储的值。 

### 哈希冲突

如果表中的桶的数量等于可能的key的数量，则可以采用散列桶和key之间的一对一关系，其中每个key恰好map到表中的一个桶。 

然而，这在大多数情况下是不切实际的，因为事先不知道潜在key的数量，或者为每个key保留存储桶所需的存储空间将超出可用内存容量。想象一下，如果的电话簿必须为宇宙中每个可能的名字保留一个条目！ 

因此，哈希函数通常不完善，可能会导致哈希冲突，即两个不同的keymap到相同的哈希值（图 1）。好的哈希函数会尽量减少冲突的可能性，但在大多数情况下它们是不可避免的。

![显示存储桶四中的哈希冲突的图表。 灰色插槽表示已被占用的插槽。 ](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/hash-collision-diagram.png)*图 1. 两个不同的key（Alice 和 Bob)具有相同的哈希值，导致存储桶 4 处发生哈希冲突*

### 开放寻址

在文献中可以找到许多解决哈希冲突的策略，但本文重点介绍一种称为线性探测(**linear probing**)的开放寻址策略。 

开放寻址哈希表使用内存中连续的存储桶数组。使用线性探测，如果在位置 i 遇到已占用的存储桶，则移动到下一个相邻位置i+1。如果这个存储桶也被占用，则移动到i+2，依此类推。当到达最后一个桶时，将回到起点。这种探测方案对于每个key都是确定性的（图 2）。

![该图显示了两个不同key的两个哈希值相同时的线性探测策略](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/linear-probing-strategy-diagram.png)*图 2. 开放寻址通过按确定性顺序遍历一系列替代存储桶的探测方案将冲突条目存储在不同位置*

这种方法的缓存效率很高，因为它访问内存中的连续位置。如果负载因子（已填充的存储桶与总存储桶的比率）较高，则可能会导致性能下降 ，因为这会导致额外的内存读取。

从hashmap中检索key Bob 的工作方式相同：从位置hash(Bob)=4开始遵循key的探测序列，直到在位置 6 处找到所需的存储桶。如果在给定key的探测序列中的任何点遇到空存储桶，则知道所查询的key不存在于hashmap中。 

## 随机存储器访问 

精心设计的散列函数通过最大化散列任意两个key产生不同散列值的可能性来最小化冲突次数。这意味着对于任何给定的两个key，它们对应的存储桶可能位于不同的内存位置。 

因此，大多数哈希表操作的内存访问模式实际上是随机的。要理解哈希表的性能，了解随机内存访问的性能非常重要。 

表 1 将理论峰值带宽与在现代 CPU 和 GPU 上 通过[GUPS 基准测试](https://icl.utk.edu/projectsfiles/hpcc/RandomAccess/)测量的随机 64 位读取所实现的带宽进行了比较。

| 芯片（内存）                                                 | 理论峰值带宽（GB/s） | **测量的随机 64 位读取带宽 (GB/s)** |
| ------------------------------------------------------------ | -------------------- | ----------------------------------- |
| [英特尔至强铂金 8360Y](https://www.intel.com/content/www/us/en/products/sku/212459/intel-xeon-platinum-8360y-processor-54m-cache-2-40-ghz/specifications.html)（DDR4-3200，8 通道） | 204                  | 15                                  |
| NVIDIA A100-80GB-SXM (HBM2e)                                 | 2039                 | 141                                 |
| NVIDIA H100-80GB-SXM (HBM3)                                  | 3352                 | 256                                 |

*表 1. 带宽的计算方式为访问大小乘以访问次数除以时间*

如果有兴趣在系统上运行 GUPS GPU 基准测试，请参阅[NVIDIA 开发人员博客代码示例](https://github.com/NVIDIA-developer-blog/code-samples/tree/master/posts/gups)GitHub 存储库。可以在[ParRes/Kernels](https://github.com/ParRes/Kernels) GitHub 存储库中访问 CPU 代码。

正如所看到的，随机内存访问比理论峰值带宽大约慢 10 倍。这是因为内存子系统针对顺序访问进行了优化。更重要的是，NVIDIA GPU 的随机访问吞吐量比现代 CPU 高出一个数量级。这些结果表明，性能最佳的 CPU 哈希表可能比性能最佳的 GPU 哈希表慢一个数量级。 

## GPU哈希表实现

随机内存访问在哈希表实现中是不可避免的，与 CPU 相比，GPU 在随机访问方面表现出色。这是有希望的，因为它暗示 GPU 应该擅长哈希表操作。为了测试这一理论，本节讨论 GPU 哈希表的实现和优化，并将性能与 CPU 实现进行比较。

我们的目标不是开发标准 C++ 容器直接替代品（例如 `std::unordered_map`)，而是专注于实现适合 GPU 加速应用程序中出现的大规模并行、高吞吐量问题的哈希表。 

此示例使用以下简化假设： 

- 表的容量是固定的——不能在初始容量之外添加额外的key值对
- 需要将其中一些key values留作哨兵值以表示空桶
- key value类型的大小之和必须小于或等于八个字节
- Key-value对一旦插入就无法删除

请注意，这些不是基本限制，可以通过 cuCollections 库中提供的更高级的实现来克服。 

首先，示例哈希表使用开放寻址并由存储桶数组组成。每个存储桶可以容纳一个key值对，并使用key/值标记进行初始化以表示它当前为空。对于碰撞解决， 使用线性探测。 

GPU 加速的哈希表需要支持来自多个线程的并发更新，并且有必要采取措施避免数据竞争，例如，如果两个线程尝试在同一位置插入。为了避免昂贵的锁定，示例哈希表使用原子操作，其中使用`libcu++` 中的 [`cuda::std::atomic`](https://nvidia.github.io/libcudacxx/extended_api/synchronization_primitives/atomic.html) 函数将每个存储桶定义为`cuda::std::atomic<pair<key, value>>`。 

要插入新key，该实现根据哈希值计算第一个存储桶，并执行原子比较和交换操作，期望存储桶中的key等于`empty_sentinel`。如果是，则槽为空，插入成功。否则，它会前进到下一个桶，直到最终找到一个空桶。 

下面的代码显示了哈希表插入函数的简化版本。

```c++
__device__ bool insert(Key k, Value v) {
// get initial probing position from the hash value of the key
auto i = hash(k) % capacity;
while (true) {
  // load the content of the bucket at the current probe position
  auto [old_k, old_v] = buckets[i].load(memory_order_relaxed);
  // if the bucket is empty we can attempt to insert the pair
  if (old_k == empty_sentinel) {
    // try to atomically replace the current content of the bucket with the input pair
    bool success = buckets[i].compare_exchange_strong(
                    {old_k, old_v}, {k,v}, memory_order_relaxed);
    if (success) {
      // store was successful
      return true;
    }
  } else if (old_k == k) {
    // input key is already present in the map
    return false;
  }
  // if the bucket was already occupied move to the next (linear) probing position
  // using the modulo operator to wrap back around to the beginning if we     
  // go beyond the capacity
  i = ++i % capacity;
}
}
```

以类似的方式查找hashmap中特定key的关联value。检查key探测序列中的每个位置，直到找到包含所需key的存储桶；或空存储桶表明该key没在hashmap中。

（**注**：「[SimpleGPUHashTable](https://nosferalatu.com/SimpleGPUHashTable.html)」一样的实现，但是cuCollections hashmap 做了进一步的优化，使用Cooperative groups 在负载系数高的情况下，线性探测的优化）

## Cooperative groups 协作组

乍一看，为每个输入元素分配一个工作线程似乎是一个合理的比例。但是，请考虑以下事项：

- 输入中的相邻key与其在内存中的相关探测位置之间没有关系。这意味着warp中的每个线程都可能访问hashmap的完全不同的区域。在最坏的情况下，每个探测步骤都需要从全局内存中的 32 个不同位置加载每个 warp。（回想一下随机存储器访问。）
- 通过线性探测，每个线程可以从其初始探测位置开始访问多个相邻的存储桶。这种本地访问模式允许使用单个合并负载预取多个探测位置，不幸的是，这无法通过单个线程实现。

我们可以做得更好吗？是的。CUDA[协作组](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#cooperative-groups)模型可以轻松地重新配置工作分配的粒度。每个输入元素不是使用单个 CUDA 线程，而是将元素分配给同一warp内的一组连续线程。 

对于给定的输入key，不是按顺序遍历其关联的探测序列，而是使用单个合并负载来预取多个相邻桶的窗口。然后，该组使用高效的 `ballot` 和 `shuffle` 内在函数合作确定窗口内的候选存储桶。

下图显示了关key Bob 的小组合作探测步骤及其中间步骤。 由四个线程组成的协作组用于将key Bob 插入哈希表中。 从由key的哈希值确定的初始探测索引开始，将桶的合并窗口加载到本地寄存器中，并使用“ballot”内在函数确定候选桶。

![](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/group-cooperative-probing-diagram.png)*图 3. key Bob 的群体合作探测步骤及其中间步骤*

以下代码扩展了之前引入的插入函数，以使用warp中的四个连续线程来协作插入单个key。`cg::thread_block_tile<4>`代表子warp中的四个线程。

```c++
enum class probing_state { SUCCESS, DUPLICATE, CONTINUE };

__device__ bool insert(cg::thread_block_tile<4> group, Key k, Value v) {
// get initial probing position from the hash value of the key
auto i = (hash(k) + group.thread_rank()) % capacity;
auto state = probing_state::CONTINUE;
while (true) {
  // load the contents of the bucket at the current probe position of each rank in a coalesced manner
  auto [old_k, old_v] = buckets[i].load(memory_order_relaxed);
  // input key is already present in the map
  if(group.any(old_k == k)) return false;
  // each rank checks if its current bucket is empty, i.e., a candidate bucket for insertion
  auto const empty_mask = group.ballot(old_k == empty_sentinel);
  // it there is an empty buckets in the group's current probing window
  if(empty_mask) {
    // elect a candidate rank (here: thread with lowest rank in mask)
    auto const candidate = __ffs(empty_mask) - 1;
    if(group.thread_rank() == candidate) {
      // attempt atomically swapping the input pair into the bucket
      bool const success = buckets[i].compare_exchange_strong(
                      {old_k, old_v}, {k, v}, memory_order_relaxed);
      if (success) {
        // insertion went successful
        state = probing_state::SUCCESS;
      } else if (old_k == k) {
        // else, re-check if a duplicate key has been inserted at the current probing position
        state = probing_state::DUPLICATE;
      }
    }
    // broadcast the insertion result from the candidate rank to all other ranks
    auto const candidate_state = group.shfl(state, candidate);
    if(candidate_state == probing_state::SUCCESS) return true;
    if(candidate_state == probing_state::DUPLICATE) return false;
  } else {
    // else, move to the next (linear) probing window
    i = (i + group.size()) % capacity;
  }
}
}
```

前面的哈希表插入函数的代码示例是 cuCollections 实际实现的简化版本`cuco::static_map`。 

图 4 显示了在[NVIDIA A100 80 GB](https://www.nvidia.com/en-us/data-center/a100/) GPU上测量的非合作和合作探测方法的性能，未具体化不同组大小和表占用率。

下图为不同协作组大小的探测吞吐量，以及不同哈希表负载因子下的最大可实现吞吐量（GUPS 结果）。

![](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/cooperative-probing-throughput-graph.png)*图 4. 对于协作探测，吞吐量以 GB/s 为单位（越高越好)。红色虚线显示峰值 GUPS 结果，它提供了该系统上可以实现的吞吐量的上限。*

如果负载系数较低，则非合作（非 CG）表现出接近最佳性能。然而，如果负载因子增加，由于冲突次数增加和探测序列更长，吞吐量会急剧下降。这是有问题的，因为较高的表加载因子对应于更好的内存利用率。 

协作探测可提高此类高负载系数场景的性能。当组大小为 4 时，当负载系数较高时，与非合作方法相比，可以观察到插入吞吐量高出 13%，查找吞吐量高出 40%。

长探测序列也会出现在具有高key重数的多值场景中，因为相同的key会遍历相同的桶序列。合作探测也有助于加快这些场景的速度。

有关组协作哈希表探测的更多信息，请参阅[多 GPU 节点上的并行哈希](https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2018-s8237/)和[WarpCore：GPU 上快速哈希表的库](https://www.nvidia.com/en-us/on-demand/session/gtcspring21-e31204/)。

## 现有CPU和GPU哈希表比较

多年来已经提出了各种 C++ hashmap实现。其中最受欢迎的是libstdc++/libc++ `std::unordered_map`和[Abseil](https://abseil.io/) `absl::flat_hash_map`。这些是顺序实现，从多个线程使用它们需要额外的同步。 

[**TBB**](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onetbb.html) `tbb::concurrent_hash_map` 和 [**Folly**](https://github.com/facebook/folly) `folly::AtomicHashMap`是在CPU中并发多线程场景下，常使用的hashmap库。GPU 上可用的少数实现之一来自[Kokkos ](https://kokkos.github.io/)`kokkos::UnorderedMap`库。

将上面提供的hashmap实现的性能与 cuCollection `cuco::static_map`进行比较。基准设置如下。 

首先，将 2^27 (1 GB) 个唯一的 4 字节key/4 字节值对插入到每个map中，然后查询同一组key以检索其关联值。每次运行的目标hashtable负载率为 50%。性能以内存吞吐量（GB/秒；越高越好）来衡量。

结果如图 5 所示。`cuco::static_map`在单个 NVIDIA H100-80GB-SXM 上实现了 87.5 GB/s 的插入吞吐量和 134.6 GB/s 的查找吞吐量，这意味着比最快的 CPU 单线程和多线程实现，有数量级的提升。此外，在本次测试中，cuCollections 的性能优于其他 GPU 实现，`kokkos::UnorderedMap`插入性能分别高出 3.8 倍，查找性能分别高出 2.6 倍。

请注意，在此基准测试设置中，每个操作的 I/O 向量驻留在 CPU 端实现的 CPU 内存中，以及 GPU 端实现的 GPU 内存中。如果 GPU 哈希表的数据向量需要驻留在 CPU 内存中，则需要首先将输入数据移至 GPU，然后将结果移回 CPU 内存。 

这可以通过显式（异步批量）复制或使用 CUDA[统一内存](https://developer.nvidia.com/blog/maximizing-unified-memory-performance-cuda/)(unified memory)概念的自动页面迁移来实现。结果表明，我们实现的吞吐量始终远高于 PCIe Gen4 的实际可用带宽，甚至高于 H100 上的 PCIe Gen5。这意味着这种方法能够使 CPU 和 GPU 之间的链路完全饱和。 

换句话说，cuCollections 能够以系统 PCIe 带宽的速度构建和查询哈希表，即使数据不在 GPU 内存中也是如此。此外，得益于 CPU 和 GPU 之间的快速 NVLink-C2C 互连，[NVIDIA Grace Hopper Superchip](https://developer.nvidia.com/blog/nvidia-grace-hopper-superchip-architecture-in-depth/)可以提供额外的加速，从而释放哈希表的全部吞吐量。相比之下，与 PCIe 相比，CPU hashmap的吞吐量通常要低得多。

![显示批量插入和批量查找操作的各种hashmap实现的吞吐量的条形图。](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/insert-find-throughput-graph.png)*图 5. 流行的 CPU 和 GPU hash map实现的性能比较*

## 多列关系连接(multicolumn relational join)示例

本节提供一个真实示例，说明如何使用 GPU 哈希表来实现复杂算法。

[cuDF](https://github.com/rapidsai/cudf)是一个用于数据分析的 GPU 加速库。它提供了数据操作的原语，例如加载、连接和聚合。通过利用 cuCollections 哈希表，它使用哈希联接算法来执行联接操作。

![该图显示了三个表，说明了 cuDF 连接实现如何用于内部连接。  ](https://github.com/weedge/mypic/raw/master/oneday//maximizing-performance-with-massively-parallel-hash-maps-on-gpus/inner-join-implementation-RAPIDS-cuDF.png)*图 6. RAPIDS cuDF 中内部联接实现的构建和探测阶段*

图 6 显示了 cuDF 连接实现如何用于内部连接。cuDF 提供内置哈希函数，将任意类型的行哈希为哈希值。不同的行可以具有相同的哈希值，因此需要进行行相等检查来确定两行是否真正相同。 

左侧的表用于填充 一个[`cuco::static_multimap`](https://github.com/NVIDIA/cuCollections/blob/dev/include/cuco/static_multimap.cuh)其中key是行的哈希值，有效负载是关联的行索引。第24行插入到第47个桶，第25行插入到第48个桶。在探测阶段，右表第200行的哈希值为47，与桶的哈希值相同（或相同的key） 47 来自哈希表。 

为了最终确定两行是否相等，需要右表中 {André-Marie, Ampère} 的行索引 200 和左表中 {Alessandro, Volta} 的行索引 24 ，传递给行相等函数*row_equal(200, 24)*。 

最后，这两行不相同，因此左侧表的第 24 行不匹配。最终，左表的第 25 行与右表的第 200 行匹配，因为哈希值相同，并且行相等性检查 ( row_equal *(200, 25)* ) 也通过了。 

考虑到大小、选择性等方面的许多选项，对连接操作进行基准测试是一个复杂的主题。有关更多详细信息，请参阅[如何充分利用 GPU 加速数据库运算符](https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2018-s8289/)和[有效、可扩展的多 GPU 连接](https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2019-s9557/)。

（**注**： 类似的场景很多，比如判断两个特征向量是否相似，合并两个特征向量等等，可以引入GPU hashmap对应实现库来加速，一般是模型训练算力加速；在web业务服务场景下，很少使用，主要是cpu服务场景已经满足，没必要进一步优化，而且gpu计算服务成本高）

## 如何在代码中使用 GPU 哈希表

GPU 非常适合hashmap等并发数据结构。这一切都始于高带宽内存架构，对于许多小型随机读取和原子更新来说，高带宽内存架构比 CPU 快一个数量级(order-of-magnitude)。这直接转化为 GPU 上高效的哈希表插入和探测性能。 

本文介绍了设计大规模并行hashmap时的一些重要注意事项：

1）具有开放寻址的哈希桶的平坦内存布局，以解决冲突；

2）线程在相邻哈希桶上进行协作以进行插入和探测，以提高高负载因子场景中的性能。

可以在 GitHub 上找到快速灵活的hashmap实现，作为[cuCollections](https://github.com/NVIDIA/cuCollections#data-structures)库的一部分。

如果高性能数据存储和检索对的应用程序很重要，那么 GPU 加速的哈希表可以成为的首选数据结构。尝试一下[cuCollections](https://github.com/NVIDIA/cuCollections)库，亲自体验 GPU 的强大功能。

(**注**：还有另外一个库[stdgpu](https://github.com/stotko/stdgpu)，提供在GPU场景下 类似c++ STL相关容器操作；两者代码结构都是标准规范的c++工程开发结构，都有example,test,benchmark，使用起来非常友好~。[NVIDIA](https://github.com/NVIDIA)官方库 [cuCollections ](https://github.com/NVIDIA/cuCollections)比较新, 优化支持更好，如果感兴趣可以贡献一波)

## Reference

1. https://developer.nvidia.com/blog/maximizing-performance-with-massively-parallel-hash-maps-on-gpus/
2. https://developer.nvidia.com/blog/maximizing-unified-memory-performance-cuda/
3. https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2018-s8289/
4. https://www.nvidia.com/en-us/on-demand/session/gtcsiliconvalley2019-s9557/
5. https://medium.com/rapids-ai/accelerating-tf-idf-for-natural-language-processing-with-dask-and-rapids-6f6e416429df
6. https://en.wikipedia.org/wiki/Hash_table
7. https://web.stanford.edu/class/ee380/Abstracts/070221_LockFreeHash.pdf
8. https://oneapi-src.github.io/oneTBB/main/tbb_userguide/concurrent_hash_map.html
9. https://github.com/facebook/folly/blob/main/folly/concurrency/ConcurrentHashMap.h
10. https://stotko.github.io/stdgpu/doxygen/classstdgpu_1_1unordered__map.html