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

简而言之，cache（又名page cache）大部分用于读设备数据缓存，提高性能。联系reclaim，最终page cache会被归类为LRU (Least Recently Used) 中的2种数据结构，active_list 和 inactive_list。

通过原文描述可知，将page加入到page cache时一并会将其加入到 LRU 的 active_list 和 inactive_list 中。当内存不够时，reclaimation操作将在inactive_list中执行，从而腾挪出更多内存供其他进程使用。 由于active_list并不是被reclaimation直接回收的，那么为什么内存使用率统计会将所有的page cache计入可利用范围内？ 请继续3.1小节的分析。

以下内容摘录自：

- 参考文章[2]中 What is page cache小节
- 参考文章[3]中 10.2 Page cache, 10.3 LRU Lists小节

> What is page cache?
>
> When a user process reads or writes a file, it is actually modifying a copy of that file in main memory. The kernel creates that copy from the disk and writes back changes to it when necessary. The memory taken up by these copies is called cached memory, in other words we call it as page cache.
>
> 10.2 Page cache
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
> 10.3  LRU Lists
>
> As stated in Section 10.1, **the LRU lists consist of two lists called active_list and inactive_list.** They are declared in mm/page_alloc.c and are protected by the pagemap_lru_lock spinlock. They, broadly speaking, store the “hot” and “cold” pages respectively, or in other words, **the active_list contains all the working sets in the system and inactive_list contains reclaim canditates.**
>
> Page Cache API:
>
> void add_to_page_cache: Adds a page to the LRU with lru_cache_add() in addition to adding it to the inode queue and page hash tables

对于Buffer memory而言，有如下描述：

简而言之，buffer用于写设备数据缓冲。一般很小。但和page cache的区别在于buff是对设备的缓冲。page cache是对文件的缓存。reclaim会在系统内存紧缺时将buff回收。

以下内容摘录自：

- 参考文章[5]
- 参考文章[6]中，5.15 Improved Memory Reclamation Policy小节

> A buffer, also called buffer memory, is a portion of a computer's memory that is set aside as a temporary holding place for data that is being sent to or received from an external device, such as a hard disk drive (HDD), keyboard or printer.
> The original meaning of buffer is a cushion-like device that reduces the shock from the contact of two objects. A buffer in a computer system is usually located between two devices that have different speeds for handling data or used when there is a difference in the timing of events. It is analogous to a reservoir, which captures water upstream and then lets it out at controlled speeds intended to prevent the lower river from overflowing its banks. Likewise, a buffer in a computer ensures that data has somewhere to go, i.e., into the buffer temporarily until its ultimate destination becomes available.
>
> 5.15    Improved Memory Reclamation Policy
The memory reclamation policy was enhanced in Digital UNIX Version 4.0 to improve system performance. In previous releases of the operating system, once the system began running out of physical memory, global paging would begin to reclaim memory, attempting to select pages fairly between the Unified Buffer Cache (UBC) and VM.

### 3.1 Shrink all caches

kernel.org描述中带有很多函数说明，但这不妨碍我们分析shrink caches的原理。有以下几个要点：

- refilling 机制会将active_list中的page转移到inactive_list中去
- refilling 行为取决于系统需要的内存情况，会动态控制（priority）每次回收内存的程度。
- reclaimation最大程度会将 active 和 inactive 都回收
- Buffer pages也就是buffer memory

请参看[3]中章节： *10.3  LRU Lists*下对 *refilling， reclaim pages, shrink all caches*的全面描述

lwn.net在*reconsidering swapping* [4] 一文中阐述anonymous page也可以被reclaimed，只是代价较大。因此引入一个问题： 如果将anonymous page也计入可用内存的统计当中，那么岂不是更多的内存都可以使用。 但这里需要反思的是，可用内存的统计需要有一个承诺前提，那就是——“性能”，基于性能来表达使用率。而对于reclaiming anonymous page而言，会有性能代价。*reconsidering swapping*一文中解释了reclaiming anonymous page的代价：

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

通过redhat介绍，每个cgroup会有自己的usage统计。如下表内容[8]。

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

**Cgroup memory可用的最终公式：**

memory usage in cgroup = memory.usage_in_bytes - (active_file + inactive_file)

事实上当没有swap交换时，该公式可以为memory usage in cgroup = memory.stat.mapped_file + memory.stat.rss

但是rss中包含了swap cache大小。

### 3.4 Memory Pressure

当然在内存压力过大时，也会产生特别现象：

- anonymous page会被回收至swap cache（前面已经提到）
- 压力过大时，share memory将不会计入cgroup统计(摘自[7]2.3 Shared Page Accounting小节)
- memory pressure是有级别的，medium级别一般会回收page cache， low级别会保证一定的cache level， 到达critical时，需要采取立即行动来尽量避免OOM。

参考内容来自[7]中 11. Memory Pressure小节

## 4 Out Of Memory

当系统内存不足时，新发生的内存申请，会触发系统OOM机制。系统会选择合适的进程来决定是否需要将其kill。以下几个条件会作为oom的先决条件：

- Is there enough swap space left (nr_swap_pages > 0) ? If yes, not OOM
- Has it been more than 5 seconds since the last failure? If yes, not OOM
- Have we failed within the last second? If no, not OOM
- If there hasn't been 10 failures at least in the last 5 seconds, we're not OOM
- Has a process been killed within the last 5 seconds? If yes, not OOM

当满足以上条件后，kernel会决定一个合适进程然后kill掉。决定因素是计分制。算法如下：

badness_for_task = total_vm_for_task / (sqrt(cpu_time_in_seconds) * sqrt(sqrt(cpu_time_in_minutes)))

目的就是选择一个短时间运行的进程，却在大量申请内存。如原文所述[9]. 另外，对于根进程，拥有CAP_SYS_ADMIN，或CAP_SYS_RAWIO的进程，这个分数会除以4。

> This has been chosen to select a process that is using a large amount of memory but is not that long lived. Processes which have been running a long time are unlikely to be the cause of memory shortage so this calculation is likely to select a process that uses a lot of memory but has not been running long. If the process is a root process or has CAP_SYS_ADMIN capabilities, the points are divided by four as it is assumed that root privilege processes are well behaved. Similarly, if it has CAP_SYS_RAWIO capabilities (access to raw devices) privileges, the points are further divided by 4 as it is undesirable to kill a process that has direct access to hardware.

为了避免进程被kill, 我们的内存统计算法也需要增加一个先决条件： 内存监控能够帮助预警OOM的事情发生。那么在OOM之前，还有哪些内存可以被回收，而不必触发OOM ？

> When checking available memory, the number of required pages is passed as a parameter to vm_enough_memory(). Unless the system administrator has specified that the system should overcommit memory, the mount of available memory will be checked. To determine how many pages are potentially available, Linux sums up the following bits of data:
>
> - Total page cache as page cache is easily reclaimed
> - Total free pages because they are already available
> - Total free swap pages as userspace pages may be paged out
> - Total pages managed by swapper_space although this double-counts the free swap pages. This is balanced by the fact that slots are sometimes reserved but not used
> - Total pages used by the dentry cache as they are easily reclaimed
> - Total pages used by the inode cache as they are easily reclaimed

但需要注意前面触发oom_Killer的先决条件，会有申请不到内存的时候。也就是即便有reclaim机制，也可能回收不及时导致最后oom。所以回想内存统计的公认算法。并没有将anonymous page（reclaim时会交换到swap cache中）作为可用内存一部分。因此，结论会比较明确：**基于性能和业务可靠性下的内存统计和预警，一方面可用内存必须有cache buff free。另外一方面必须有阈值告警，在内存未达到最高峰前做必要扩容或分摊压力处理。**

## 5 Conclusion

对于内存统计而言：

- page cache(active_list和inactive_list)， buffer都会被回收。这取决于kernel内部的算法。
- 这部分被回收的内存算作可利用内存的一部分。
- 假设上面提到的内存统计是“公认的内存可用统计”。意味着在性能一定的前提下，anonymous page不能被回收，应该算作已使用内存。
- cgroup中cache 的部分组成（active_file和inactive_file）说明了这部分是可以回收的。但基于“保证性能”的前提下，share memory应该排除在外（share memory reclaim时会被swap cache接收）
- 鉴于OOM的出现，我们需要做必要的预警来保证业务平稳运行（如果回收内存不及时会导致oom）

## 6 Reference

1. [Interpreting /proc/meminfo and free output for Red Hat Enterprise Linux 5, 6 and 7](https://access.redhat.com/solutions/406773)
2. [RHEL 6 – Controlling Cache Memory / Page Cache Size](http://unixadminschool.com/blog/2013/10/rhel-6-controlling-cache-memory-page-cache-size/)
3. [Chapter 10  Page Frame Reclamation](https://www.kernel.org/doc/gorman/html/understand/understand013.html)
4. [Reconsidering swapping](https://lwn.net/Articles/690079/)
5. [Buffer Definition](http://www.linfo.org/buffer.html)
6. [5 Virtual Memory](https://www3.physnet.uni-hamburg.de/physnet/Tru64-Unix/HTML/AQTLLATE/DOCU_012.HTM)
7. [Memory Resource Controller](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)
8. [Cgroup subsystem -- 3.7. MEMORY](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/sec-memory)
9. [Chapter 13  Out Of Memory Management](https://www.kernel.org/doc/gorman/html/understand/understand016.html)

其他可以加强对memory使用率统计的理解文章：

- [Low On Memory](https://linux-mm.org/Low_On_Memory)
- [What page replacement algorithms are used in Linux kernel for OS file cache?](https://unix.stackexchange.com/questions/281974/what-page-replacement-algorithms-are-used-in-linux-kernel-for-os-file-cache/282008)
- [What do the “buff/cache” and “avail mem” fields in top mean?](https://unix.stackexchange.com/questions/390518/what-do-the-buff-cache-and-avail-mem-fields-in-top-mean)
- [How can I get the amount of available memory portably across distributions?](https://unix.stackexchange.com/questions/261247/how-can-i-get-the-amount-of-available-memory-portably-across-distributions) 本文介绍了available memory的计算方法，事实上再一次说明内存使用率是无法精确计算的
- [cgroup memory usage statistic](https://hustcat.github.io/cgroup-memory-statistic-again/)
