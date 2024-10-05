关键词：mempool、rte_ring、mbuf、object cache

DPDK 使用大页作为其内存分配总资源池，是为了减少 TLB miss，以加快系统运行速度。除此之外，DPDK 还使用了 mempool 机制，对收发数据包时使用的内存进行更细致和更有效率的管理。
DPDK 的 `mempool`（内存池）机制是其内存管理的核心组件之一。`mempool` 是一个预分配的固定大小的内存对象池，用于高效分配和回收内存块，避免了频繁的内存分配和释放操作，减少了内存碎片和内存分配的开销。这种机制特别适用于网络数据包处理场景下，网络数据包的分配和释放频繁，传统的内存分配机制会带来较大的性能开销。
### 1. **mempool 的核心概念**
`mempool` 是 DPDK 提供的一种通用内存管理机制，主要用于管理对象（例如 `rte_mbuf`）的分配和释放。`mempool` 内存池是静态分配的，即在应用程序启动时，内存池的大小已经被确定，分配和释放都在这个池内进行。
#### 特点：
- **固定大小**：内存池由固定大小的对象组成，每个对象的大小是相同的。
- **静态分配**：所有内存块在启动时分配好，避免了运行时动态分配的开销。
- **高效回收**：通过锁自由或基于 CPU 核心的局部缓存机制，提供了高效的分配与回收。
- **多核友好**：支持多核并行分配和回收，减少锁竞争。
### 2. **mempool 的基本工作原理**
对于需要频繁分配/释放的数据结构，最典型的就是管理和保存数据包的数据结构，可以采用内存池的方式预先动态分配一整块内存区域，然后统一进行管理并提供更快速的分配和回收，从而免除频繁地动态分配/释放过程，既提高了性能，也减少了内存碎片的产生。这就是 DPDK 中 mempool 机制出现的原因。DPDK 中，数据包的内存操作对象被抽象为 mbuf，其对应的 struct rte_mbuf 数据结构对象存储在内存池中。DPDK 以环形队列（ring）的形式组织内存池中空闲或被占用的内存。此外还考虑了地址和 Cache Line 对齐等因素，以提升性能。
#######################################
图8-4
#######################################
#### 2.1 **mempool 的创建**
当创建一个 `mempool` 时，会从操作系统中分配一块连续的大内存（通常使用巨页内存）。然后，将这块内存划分为若干固定大小的对象（称为 `element`），这些对象存储在内存池中，等待被使用。
创建 `mempool` 的典型代码如下：
```c
struct rte_mempool *mbuf_pool;
mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL", NUM_MBUFS, MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());
```
- `"MBUF_POOL"`：内存池的名称，用于标识该内存池。
- `NUM_MBUFS`：内存池中元素的总数（即对象的个数）。
- `MBUF_CACHE_SIZE`：每个 CPU 核心上可以缓存的对象数（用于加速分配和回收）。
- `0`：额外的私有数据空间大小。
- `RTE_MBUF_DEFAULT_BUF_SIZE`：每个 `mbuf` 的大小。
- `rte_socket_id()`：用于为该内存池分配内存的 NUMA 节点。
#### 2.2 **分配和回收对象**
`mempool` 的分配和回收是通过 `rte_mempool_get` 和 `rte_mempool_put` 来实现的。
- **分配对象**：当需要使用内存时，调用 `rte_mempool_get` 从内存池中取出一个对象。
  ```c
  struct rte_mbuf *m;
  if (rte_mempool_get(mbuf_pool, (void **)&m) < 0) {
      printf("Failed to get an mbuf from the pool\n");
      return;
  }
  ```
- **回收对象**：当对象不再使用时，调用 `rte_mempool_put` 将对象归还到内存池。
  ```c
  rte_mempool_put(mbuf_pool, m);
  ```

DPDK 使用内存池来管理 `rte_mbuf`（数据包缓冲区），即每次从 `rte_mempool` 中分配 `rte_mbuf` 来处理数据包，数据包处理完成后，将 `rte_mbuf` 归还到 `rte_mempool`。

### 3. **mempool 的结构**

`mempool` 的主要结构体是 `struct rte_mempool`，它定义了内存池的属性和管理方法。`mempool` 的元素分布在多个页上，这些页从系统内存中分配而来。
#### `rte_mempool` 结构体：
```c
struct rte_mempool {
    char name[RTE_MEMPOOL_NAMESIZE];    // 内存池的名称
    unsigned size;                      // 内存池中的对象总数
    unsigned cache_size;                // 每个 CPU 核心的缓存大小
    unsigned elt_size;                  // 每个对象的大小
    struct rte_mempool_cache *local_cache;  // 每个 CPU 核的本地缓存
    struct rte_mempool_list *pool_data;     // 实际存放对象的内存池列表
    // ...
};
```
#### 关键字段说明：
- **name**：内存池的名称，用于标识不同的 `mempool`。
- **size**：内存池中的对象总数，即内存池中可以存放的对象的数量。
- **cache_size**：CPU 核心本地缓存的大小，用于减少锁竞争。
- **elt_size**：每个对象的大小，即每个元素的内存大小。
- **local_cache**：每个 CPU 核的本地缓存，避免频繁的全局加锁，减少分配和回收时的锁争用。
- **pool_data**：存放内存池中对象的实际内存区域。
### 4. **mempool 缓存（Local Cache）机制**
为了进一步提高内存分配和释放的效率，`mempool` 引入了本地缓存（local cache）机制。每个 CPU 核心有一个私有的对象缓存，用于减少全局内存池访问的频率，避免频繁的锁竞争。缓存机制的工作原理如下：
1. **对象分配时**：首先从本地缓存中获取对象。如果本地缓存有可用的对象，则直接返回；如果没有对象，则从全局的内存池获取一批对象填充本地缓存。
2. **对象释放时**：对象先归还到本地缓存中。如果本地缓存满了，则将多余的对象归还到全局内存池。
缓存的好处：
- **减少锁竞争**：由于每个 CPU 核心可以从自己的本地缓存中分配和回收对象，减少了多个核心同时访问全局 `mempool` 带来的锁争用。
- **提高分配效率**：通过批量分配和回收，可以提高每次分配和回收的效率。
### 5. **巨页内存（Hugepages）支持**
DPDK 强烈建议在 `mempool` 中使用巨页内存（Hugepages），因为它可以显著提升内存访问性能。巨页内存通过减少页表项的数量，减少 TLB（Translation Lookaside Buffer）失效的频率，从而提高内存访问速度。巨页的优势如下：
- **减少内存碎片**：巨页内存的页大小通常为 2MB 或 1GB，比标准的 4KB 页大得多，可以减少内存碎片。
- **更高的缓存命中率**：巨页内存减少了 TLB 表的项数，降低了 TLB 缺失的概率，从而提高了内存的访问速度。
#### 配置巨页内存的方法：
巨页内存通常在 DPDK 初始化时配置，使用如下命令为系统分配巨页内存：

```bash
sudo sysctl -w vm.nr_hugepages=1024
```
