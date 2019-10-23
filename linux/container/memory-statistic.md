# Memory Statistic in Container

## 1 Background

容器的内存使用统计情况需要基于cgroup提供的系统接口memory.stat来实现。但memory.stat中呈现的内存参数表达的过于复杂，且和常规free指令或/proc/meminfo呈现的指标出入很大。因此需要一个统一的度量依据来确定内存实际使用情况。

## 2 Investigation of Memory usage

通常来讲，可用内存（available）会包含free+cache+buffer。正如redhat对free，meminfo的如下定义。包括AWS的内存采集也是如此。[1]

在Red Hat Enterprise Linux 5, 6 and 7.0版本下，free输出和meminfo对应关系：

free output | coresponding /proc/meminfo fields
-----------|--------------------------------
Mem: total | MemTotal
Mem: used | MemTotal - MemFree
Mem: free | MemFree
Mem: shared (can be ignored nowadays. It has no meaning.) | N/A
Mem: buffers | Buffers
Mem: cached | Cached
-/+ buffers/cache: used | MemTotal - (MemFree + Buffers + Cached)
-/+ buffers/cache: free | MemFree + Buffers + Cached
Swap: total | SwapTotal
Swap: used | SwapTotal - SwapFree
Swap: free | SwapFree

业界常用算法，real_used = total - (free + buff + cache)，-/+buffers/cache 值对应 Red Hat Enterprise Linux 7.1 or later是mem available。如下表：

free output | coresponding /proc/meminfo fields
-----------|---------------------------------
Mem: total | MemTotal
Mem: used | MemTotal - MemFree - Buffers - Cached - Slab
Mem: free | MemFree
Mem: shared | Shmem
Mem: buff/cache | Buffers + Cached + Slab
Mem:available | MemAvailable
Swap: total | SwapTotal
Swap: used | SwapTotal - SwapFree
Swap: free | SwapFree

大部分memory usage计算方式都会将cache buff计入可用内存。这部分内存系统会通过page frame reclaimation机制来回收page，从而给进程使用。分析aws metric采集算法发现：在上报EC2和ECS 内存监控时，都采用同样的依据, 虽然公式使用的内容数据来源不一致，但是计算思路是一致的。

```text
# Reference: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/mon-scripts.html#monitoring-scripts-intro 下载脚本CloudWatchMonitoringScripts
EC2: memory.usage = 100 * (total - (free + cache + buff)) / total

# Reference: https://github.com/aws/amazon-ecs-agent/pull/582
ECS: memory.usage = 100 * (memory.usage_in_bytes - memory.stat.cache) / memory.limit_in_bytes
```

本人认为包括AWS在内，上面的公式是对内存使用率统计的一种公认算法。而并非最准确的算法。在确定是否准确之前，我们先了解cache, buff基本概念，以及基本的回收机制。

## 3 What is Page Cache, Buffer and How was them reclaimed

简而言之，cache（又名page cache）大部分用于读设备数据缓存，提高性能。联系reclaim，最终page cache会被归类为LRU中的2种数据结构，active_list 和 inactive_list。

通过原文描述可知，将page加入到page cache时一并会将其加入到 LRU 的 active_list 和 inactive_list 中。而reclaimation操作将在inactive_list中执行。 那么为什么内存使用率统计会将所有的page cache计入可利用范围内？

原文摘录：

> What is page cache?[2]
>
> When a user process reads or writes a file, it is actually modifying a copy of that file in main memory. The kernel creates that copy from the disk and writes back changes to it when necessary. The memory taken up by these copies is called cached memory, in other words we call it as page cache.
>
> 10.2 Page cache[3]
>
> Pages read from a file or block device are generally added to the page cache to avoid further disk IO.
>
> The page cache is a set of data structures which contain pages that are backed by regular files, block devices or swap. There are basically four types of pages that exist in the cache:
>
> - Pages that were faulted in as a result of reading a memory mapped file;
> - Blocks read from a block device or filesystem are packed into special pages called buffer pages. The number of blocks that may fit depends on the size of the block and the page size of the architecture;
> - Anonymous pages exist in a special aspect of the page cache called the swap cache when slots are allocated in the backing storage for page-out, discussed further in Chapter 11;
> - **Pages belonging to shared memory regions are treated in a similar fashion to anonymous pages**. The only difference is that shared pages are added to the swap cache and space reserved in backing storage immediately after the first write to the page.
>
> 10.3  LRU Lists[3]
>
> As stated in Section 10.1, **the LRU lists consist of two lists called active_list and inactive_list.** They are declared in mm/page_alloc.c and are protected by the pagemap_lru_lock spinlock. They, broadly speaking, store the “hot” and “cold” pages respectively, or in other words, **the active_list contains all the working sets in the system and inactive_list contains reclaim canditates.**
>
> Page Cache API:
>
> void add_to_page_cache: Adds a page to the LRU with lru_cache_add() in addition to adding it to the inode queue and page hash tables

对于Buffer memory而言，有如下描述：

简而言之，buffer用于写设备数据缓冲。一般很小。reclaim时会将dirty page写入设备。kernel会等待IO结束后，将page回收。

原文摘录：

> A buffer, also called buffer memory, is a portion of a computer's memory that is set aside as a temporary holding place for data that is being sent to or received from an external device, such as a hard disk drive (HDD), keyboard or printer.
> The original meaning of buffer is a cushion-like device that reduces the shock from the contact of two objects. A buffer in a computer system is usually located between two devices that have different speeds for handling data or used when there is a difference in the timing of events. It is analogous to a reservoir, which captures water upstream and then lets it out at controlled speeds intended to prevent the lower river from overflowing its banks. Likewise, a buffer in a computer ensures that data has somewhere to go, i.e., into the buffer temporarily until its ultimate destination becomes available.[5]

> reclaim buffer的原理来自文章[3]中小节： *10.3.2  Reclaiming Pages from the LRU Lists*

### 3.1 Shrink all caches

kernel.org描述中带有很多函数说明，但这不妨碍我们分析shrink caches的原理。有以下几个要点：

- refilling 机制会将active_list中的page转移到inactive_list中去
- refilling 行为取决于系统需要的内存情况，会动态控制（priority）每次回收内存的程度。
- reclaimation最大程度会将 active 和 inactive 都回收
- Buffer pages也就是buffer memory

请参看[3]中章节： *10.3  LRU Lists*下对 *refilling， reclaim pages, shrink all caches*的全面描述

lwn.net在*reconsidering swapping* [4] 一文中阐述anonymous page也可以被reclaimed，只是代价较大。因此引入一个问题： 如果将anonymous page也计入可用内存的统计当中，那么岂不是更多的内存都可以使用。 但这里需要反思的是，可用内存的统计需要有一个承诺前提，那就是——性能，基于性能来表达使用率。而对于reclaiming anonymous page而言，会有性能代价。*reconsidering swapping*一文中解释了reclaiming anonymous page的代价：

- 需要将anonymous page写入swap cache，才算reclaim。
- 这一行为和page cache相比，性能会差很多。因为page cache大部分都是整个文件，有连续的地址空间，对于磁盘而言顺序写和读是更快的。而anonymous page则是分散的，写入磁盘是随机的
- 因此很多情况下，操作系统可能就不提供swap或者将swappiness调整到最低。

### 3.2 Cgroup memory stat

首先，在cgroup memory子系统下，有默认的统计使用率接口。可以执行

```bash
cat /sys/fs/cgroup/memory/<cgroup_name>/memory.usage_in_bytes
```

但是在kernel.org中有所阐述，这个统计是不合理的（基于充分利用资源和性能角度而言）.因此我们需要根据memory.stat进一步分析

原文摘录：

> 5.5 usage_in_bytes[7]
>
> For efficiency, as other kernel components, memory cgroup uses some optimization to avoid unnecessary cacheline false sharing. usage_in_bytes is affected by the method and doesn't show 'exact' value of memory (and swap) usage, it's a fuzz value for efficient access. (Of course, when necessary, it's synchronized.) If you want to know more exact memory usage, you should use RSS+CACHE(+SWAP) value in memory.stat(see 5.2).

通过redhat介绍，每个cgroup会有自己的usage统计。如下表内容。

Statistic | Description
----------|--------------
cache | page cache, including tmpfs (shmem), in bytes
rss | anonymous and swap cache, not including tmpfs (shmem), in bytes
mapped_file | size of memory-mapped mapped files, including tmpfs (shmem), in bytes
pgpgin | number of pages paged into memory
pgpgout | number of pages paged out of memory
swap | swap usage, in bytes
active_anon | anonymous and swap cache on active least-recently-used (LRU) list, including tmpfs (shmem), in bytes
inactive_anon | anonymous and swap cache on inactive LRU list, including tmpfs (shmem), in bytes
active_file | file-backed memory on active LRU list, in bytes
inactive_file | file-backed memory on inactive LRU list, in bytes
unevictable | memory that cannot be reclaimed, in bytes
hierarchical_memory_limit | memory limit for the hierarchy that contains the memory cgroup, in bytes
hierarchical_memsw_limit | memory plus swap limit for the hierarchy that contains the memory cgroup, in bytes

而且额外声明：

- active_anon + inactive_anon = anonymous memory + file cache for tmpfs + swap cache

Therefore, active_anon + inactive_anon ≠ rss, because rss does not include tmpfs.

- active_file + inactive_file = cache - size of tmpfs

所以我们可以知道：cgroup memory.stat.cache = ac + tive_file + inactive_file + size of tmpfs. 而表中强调的cache 中tmpfs仅仅是share memory。所以回过头去看AWS ECS 对cgroup memory统计方式是否正确？暂且还看不出来，**至少我们无法判断share memory是否可以在基于性能考虑的情况下做内存回收操作。**

### 3.3 Can Share memory be reclaimed

事实上在前面的小节中已经阐明了share memory 在统计使用率上应该如何界定，我们再来回顾一下：

> **Pages belonging to shared memory regions are treated in a similar fashion to anonymous pages**. The only difference is that shared pages are added to the swap cache and space reserved in backing storage immediately after the first write to the page.

而anonymous page又在3.1中指出，anonymous page的reclaimation代价很大，而且需要移入swap cache。**所以对于cgroup memory.stat而言，统计内存使用率时，我们不应该将share memory计入内存可用统计**。

stackoverflow也有相关回答[How is memory corruption handled by Linux when the process terminates?](https://unix.stackexchange.com/questions/370338/how-is-memory-corruption-handled-by-linux-when-the-process-terminates)

Cgroup memory可用的最终公式：

memory usage in cgroup = memory.usage_in_bytes - (active_file + inactive_file)

## 4 The Approach of Dockerd



## 5 Conclusion

- page cache(active_list和inactive_list)， buffer都会被回收。这取决于kernel内部的算法。
- 这部分被回收的内存算作可利用内存的一部分。
- 假设上面提到的内存统计是“公认的内存可用统计”。意味着在性能一定的前提下，anonymous page不能被回收，应该算作已使用内存
- cgroup中cache 的组成（active_file和inactive_file）说明了这部分是可以回收的。但基于“保证性能”的前提下，share memory应该排除在外

## 5 Reference

1. [Interpreting /proc/meminfo and free output for Red Hat Enterprise Linux 5, 6 and 7](https://access.redhat.com/solutions/406773)
2. [RHEL 6 – Controlling Cache Memory / Page Cache Size](http://unixadminschool.com/blog/2013/10/rhel-6-controlling-cache-memory-page-cache-size/)
3. [Chapter 10  Page Frame Reclamation - 10.2.3  Adding Pages to the Page Cache](https://www.kernel.org/doc/gorman/html/understand/understand013.html)
4. [Reconsidering swapping](https://lwn.net/Articles/690079/)
5. [Buffer Definition](http://www.linfo.org/buffer.html)
6. [Cgroup subsystem -- 3.7. MEMORY](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory)
7. [Memory Resource Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)

其他可以加强对memory使用率统计的理解文章：

- [Low On Memory](https://linux-mm.org/Low_On_Memory)
- [What page replacement algorithms are used in Linux kernel for OS file cache?](https://unix.stackexchange.com/questions/281974/what-page-replacement-algorithms-are-used-in-linux-kernel-for-os-file-cache/282008)
- [What do the “buff/cache” and “avail mem” fields in top mean?](https://unix.stackexchange.com/questions/390518/what-do-the-buff-cache-and-avail-mem-fields-in-top-mean)
- [How can I get the amount of available memory portably across distributions?](https://unix.stackexchange.com/questions/261247/how-can-i-get-the-amount-of-available-memory-portably-across-distributions) 本文介绍了available memory的计算方法，事实上再一次说明内存使用率是无法精确计算的
