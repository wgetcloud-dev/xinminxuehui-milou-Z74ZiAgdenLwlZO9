[合集 \- Redis(1\)](https://github.com)1\.Redis 内存突增时，如何定量分析其内存使用情况09\-23收起
# 背景


最近碰到一个 case，一个 Redis 实例的内存突增，`used_memory`最大时达到了 78\.9G，而该实例的`maxmemory`配置却只有 16G，最终导致实例中的数据被大量驱逐。


以下是问题发生时`INFO MEMORY`的部分输出内容。



```
# Memory
used_memory:84716542624
used_memory_human:78.90G
used_memory_rss:104497676288
used_memory_rss_human:97.32G
used_memory_peak:84716542624
used_memory_peak_human:78.90G
used_memory_peak_perc:100.00%
used_memory_overhead:75682545624
used_memory_startup:906952
used_memory_dataset:9033997000
used_memory_dataset_perc:10.66%
allocator_allocated:84715102264
allocator_active:101370822656
allocator_resident:102303637504
total_system_memory:810745470976
total_system_memory_human:755.07G
used_memory_lua:142336
used_memory_lua_human:139.00K
used_memory_scripts:6576
used_memory_scripts_human:6.42K
number_of_cached_scripts:13
maxmemory:17179869184
maxmemory_human:16.00G
maxmemory_policy:volatile-lru
allocator_frag_ratio:1.20
allocator_frag_bytes:16655720392

```

内存突增导致数据被驱逐，是 Redis 中一个较为常见的问题。很多童鞋在面对这类问题时往往缺乏清晰的分析思路，常常误以为是复制、RDB 持久化等操作引起的。接下来，我们看看如何系统地分析这类问题。


本文主要包括以下几部分：


1. INFO 中的`used_memory`是怎么来的？
2. 什么是`used_memory`？
3. `used_memory`内存通常会被用于哪些场景？
4. Redis 7 在内存统计方面的变化。
5. 数据驱逐的触发条件——当`used_memory`超过`maxmemory`后，是否一定会触发驱逐？
6. 最后，分享一个脚本，帮助实时分析`used_memory`增长时，具体是哪一部分的内存消耗导致的。


# INFO 中的 used\_memory 是怎么来的？


当我们执行`INFO`命令时，Redis 会调用`genRedisInfoString`函数来生成其输出。



```
// server.c
sds genRedisInfoString(const char *section) {
    ...
    /* Memory */
    if (allsections || defsections || !strcasecmp(section,"memory")) {
        ...
        size_t zmalloc_used = zmalloc_used_memory();
        ...
        if (sections++) info = sdscat(info,"\r\n");
        info = sdscatprintf(info,
            "# Memory\r\n"
            "used_memory:%zu\r\n"
            "used_memory_human:%s\r\n"
            "used_memory_rss:%zu\r\n"
            ...
            "lazyfreed_objects:%zu\r\n",
            zmalloc_used,
            hmem,
            server.cron_malloc_stats.process_rss,
            ...
            lazyfreeGetFreedObjectsCount()
        );
        freeMemoryOverheadData(mh);
    }
    ...
    return info;
}

```

可以看到，used\_memory 的值来自 zmalloc\_used，而 zmalloc\_used 又是通过`zmalloc_used_memory()`函数获取的。



```
// zmalloc.c
size_t zmalloc_used_memory(void) {
    size_t um;
    atomicGet(used_memory,um);
    return um;
}

```

zmalloc\_used\_memory() 的实现很简单，就是以原子方式读取 used\_memory 的值。


# 什么是 used\_memory


`used_memory`是一个静态变量，其类型为`redisAtomic size_t`，其中`redisAtomic`是`_Atomic`类型的别名。`_Atomic`是 C11 标准引入的关键字，用于声明原子类型，保证在多线程环境中对该类型的操作是原子的，避免数据竞争。



```
#define redisAtomic _Atomic
static redisAtomic size_t used_memory = 0;

```

`used_memory` 的更新主要通过两个宏定义实现：



```
#define update_zmalloc_stat_alloc(__n) atomicIncr(used_memory,(__n))
#define update_zmalloc_stat_free(__n) atomicDecr(used_memory,(__n))

```

其中，`update_zmalloc_stat_alloc(__n)`是在分配内存时调用，它通过原子操作让 used\_memory 加`__n。`


而`update_zmalloc_stat_free(__n)`则是在释放内存时调用，它通过原子操作让 used\_memory 减`__n`。


这两个宏确保了在内存分配和释放过程中`used_memory`的准确更新，并且避免了并发操作带来的数据竞争问题。


在通过内存分配器（常用的内存分配器有 glibc's malloc、jemalloc、tcmalloc，Redis 中一般使用 jemalloc）中的函数分配或释放内存时，会同步调用`update_zmalloc_stat_alloc`或`update_zmalloc_stat_free`来更新 used\_memory 的值。


在 Redis 中，内存管理主要通过以下两个函数来实现：



```
// zmalloc.c
void *ztrymalloc_usable(size_t size, size_t *usable) {
    ASSERT_NO_SIZE_OVERFLOW(size);
    void *ptr = malloc(MALLOC_MIN_SIZE(size)+PREFIX_SIZE);

    if (!ptr) return NULL;
#ifdef HAVE_MALLOC_SIZE
    size = zmalloc_size(ptr);
    update_zmalloc_stat_alloc(size);
    if (usable) *usable = size;
    return ptr;
#else
    ...
#endif
}

void zfree(void *ptr) {
    ...
    if (ptr == NULL) return;
#ifdef HAVE_MALLOC_SIZE
    update_zmalloc_stat_free(zmalloc_size(ptr));
    free(ptr);
#else
   ...
#endif
}

```

其中，


* `ztrymalloc_usable`函数用于分配内存。该函数首先会调用`malloc`分配内存。如果分配成功，则会通过 `update_zmalloc_stat_alloc`更新 used\_memory 的值。
* `zfree` 函数用于释放内存。在释放内存之前，先通过`update_zmalloc_stat_free`调整 used\_memory 的值，然后再调用`free`释放内存。


这种机制保证了 Redis 能够准确跟踪内存的分配和释放情况，从而有效地管理内存使用。


# used\_memory 内存通常会被用于哪些场景？


`used_memory`主要由两部分组成：


1. 数据本身：对应 INFO 中的`used_memory_dataset`。
2. 内部管理和维护数据结构的开销：对应 INFO 中的`used_memory_overhead`。


需要注意的是，used\_memory\_dataset 并不是根据 Key 的数量及 Key 使用的内存计算出来的，而是通过 used\_memory 减去 used\_memory\_overhead 得到的。


接下来，我们重点分析下`used_memory_overhead` 的来源。实际上，Redis 提供了一个单独的函数\-`getMemoryOverheadData`，专门用于计算这一部分的内存开销。



```
// object.c
struct redisMemOverhead *getMemoryOverheadData(void) {
    int j;
    // mem_total 用于累积总的内存开销，最后会赋值给 used_memory_overhead。
    size_t mem_total = 0;
    // mem 用于计算每一部分的内存使用量。
    size_t mem = 0;
    // 调用 zmalloc_used_memory() 获取 used_memory。
    size_t zmalloc_used = zmalloc_used_memory();
    // 使用 zcalloc 分配一个 redisMemOverhead 结构体的内存。
    struct redisMemOverhead *mh = zcalloc(sizeof(*mh));
    ...
    // 将 Redis 启动时的内存使用量 server.initial_memory_usage 加入到总内存开销中。
    mem_total += server.initial_memory_usage;

    mem = 0;
    // 将复制积压缓冲区的内存开销加入到总内存开销中。
    if (server.repl_backlog)
        mem += zmalloc_size(server.repl_backlog);
    mh->repl_backlog = mem;
    mem_total += mem;

    /* Computing the memory used by the clients would be O(N) if done
     * here online. We use our values computed incrementally by
     * clientsCronTrackClientsMemUsage(). */
    // 计算客户端内存开销
    mh->clients_slaves = server.stat_clients_type_memory[CLIENT_TYPE_SLAVE];
    mh->clients_normal = server.stat_clients_type_memory[CLIENT_TYPE_MASTER]+
                         server.stat_clients_type_memory[CLIENT_TYPE_PUBSUB]+
                         server.stat_clients_type_memory[CLIENT_TYPE_NORMAL];
    mem_total += mh->clients_slaves;
    mem_total += mh->clients_normal;
    // 计算 AOF 缓冲区和 AOF Rewrite Buffer 的内存开销
    mem = 0;
    if (server.aof_state != AOF_OFF) {
        mem += sdsZmallocSize(server.aof_buf);
        mem += aofRewriteBufferSize();
    }
    mh->aof_buffer = mem;
    mem_total+=mem;
    // 计算 Lua 脚本缓存的内存开销
    mem = server.lua_scripts_mem;
    mem += dictSize(server.lua_scripts) * sizeof(dictEntry) +
        dictSlots(server.lua_scripts) * sizeof(dictEntry*);
    mem += dictSize(server.repl_scriptcache_dict) * sizeof(dictEntry) +
        dictSlots(server.repl_scriptcache_dict) * sizeof(dictEntry*);
    if (listLength(server.repl_scriptcache_fifo) > 0) {
        mem += listLength(server.repl_scriptcache_fifo) * (sizeof(listNode) +
            sdsZmallocSize(listNodeValue(listFirst(server.repl_scriptcache_fifo))));
    }
    mh->lua_caches = mem;
    mem_total+=mem;
    // 计算数据库的内存开销：遍历所有数据库 (server.dbnum)。对于每个数据库，计算主字典 (db->dict) 和过期字典 (db->expires) 的内存开销。
    for (j = 0; j < server.dbnum; j++) {
        redisDb *db = server.db+j;
        long long keyscount = dictSize(db->dict);
        if (keyscount==0) continue;

        mh->total_keys += keyscount;
        mh->db = zrealloc(mh->db,sizeof(mh->db[0])*(mh->num_dbs+1));
        mh->db[mh->num_dbs].dbid = j;

        mem = dictSize(db->dict) * sizeof(dictEntry) +
              dictSlots(db->dict) * sizeof(dictEntry*) +
              dictSize(db->dict) * sizeof(robj);
        mh->db[mh->num_dbs].overhead_ht_main = mem;
        mem_total+=mem;

        mem = dictSize(db->expires) * sizeof(dictEntry) +
              dictSlots(db->expires) * sizeof(dictEntry*);
        mh->db[mh->num_dbs].overhead_ht_expires = mem;
        mem_total+=mem;

        mh->num_dbs++;
    }
    // 将计算的 mem_total 赋值给 mh->overhead_total。
    mh->overhead_total = mem_total;
    // 计算数据的内存开销 (zmalloc_used - mem_total) 并存储在 mh->dataset。
    mh->dataset = zmalloc_used - mem_total;
    mh->peak_perc = (float)zmalloc_used*100/mh->peak_allocated;

    /* Metrics computed after subtracting the startup memory from
     * the total memory. */
    size_t net_usage = 1;
    if (zmalloc_used > mh->startup_allocated)
        net_usage = zmalloc_used - mh->startup_allocated;
    mh->dataset_perc = (float)mh->dataset*100/net_usage;
    mh->bytes_per_key = mh->total_keys ? (net_usage / mh->total_keys) : 0;

    return mh;
}

```

基于上面代码的分析，可以知道 used\_memory\_overhead 由以下几部分组成：


* server.initial\_memory\_usage：Redis 启动时的内存使用量，对应 INFO 中`used_memory_startup`。
* mh\-\>repl\_backlog：复制积压缓冲区的内存开销，对应 INFO 中的`mem_replication_backlog`。
* mh\-\>clients\_slaves：从库的内存开销。对应 INFO 中的`mem_clients_slaves`。
* mh\-\>clients\_normal：其它客户端的内存开销，对应 INFO 中的`mem_clients_normal`。
* mh\-\>aof\_buffer：AOF 缓冲区和 AOF 重写缓冲区的内存开销，对应 INFO 中的`mem_aof_buffer`。AOF 缓冲区是数据写入 AOF 之前使用的缓冲区。AOF 重写缓冲区是 AOF 重写期间，用于存放新增数据的缓冲区。
* mh\-\>lua\_caches：Lua 脚本缓存的内存开销，对应 INFO 中的`used_memory_scripts`。Redis 5\.0 新增的。
* 字典的内存开销，这部分内存在 INFO 中没有显示，需要通过`MEMORY STATS`查看。



```
17) "db.0"
18) 1) "overhead.hashtable.main"
    2) (integer) 2536870912
    3) "overhead.hashtable.expires"
    4) (integer) 0

```


在这些内存开销中，used\_memory\_startup 基本不变，mem\_replication\_backlog 受 repl\-backlog\-size 的限制，used\_memory\_scripts 开销一般不大，而字典的内存开销则与数据量的大小成正比。


所以，重点需要注意的主要有三项：`mem_clients_slaves`，`mem_clients_normal` 和`mem_aof_buffer`。


* mem\_aof\_buffer：重点关注 AOF 重写期间缓冲区的大小。
* mem\_clients\_slaves 和 mem\_clients\_normal：都是客户端，内存分配方式相同。客户端的内存开销主要包括以下三部分：
1. 输入缓冲区：用于暂存客户端命令，大小由 `client-query-buffer-limit` 限制。
2. 输出缓冲区：用于缓存发送给客户端的数据，大小受 `client-output-buffer-limit` 控制。如果数据超过软硬限制并持续一段时间，客户端会被关闭。
3. 客户端对象本身占用的内存。


# Redis 7 在内存统计方面的变化


在 Redis 7 中，还会统计以下项的内存开销：


* mh\-\>cluster\_links：集群链接的内存开销，对应 INFO 中的`mem_cluster_links`。
* mh\-\>functions\_caches：Function 缓存的内存开销，对应 INFO 中的`used_memory_functions`。
* 集群模式下键到槽映射的内存开销，对应 MEMORY STATS 中的`overhead.hashtable.slot-to-keys`。


此外，Redis 7 引入了 Multi\-Part AOF，这个特性移除了 AOF 重写缓冲区。


需要注意的是，mh\-\>repl\_backlog 和 mh\-\>clients\_slaves 的内存计算方式也发生了变化。


在 Redis 7 之前，mh\-\>repl\_backlog 统计的是复制积压缓冲区的大小，mh\-\>clients\_slaves 统计的是所有从节点客户端的内存使用量。



```
if (server.repl_backlog)
    mem += zmalloc_size(server.repl_backlog);
mh->repl_backlog = mem;
mem_total += mem;

mem = 0;
// 遍历所有从节点客户端，累加它们的输出缓冲区、输入缓冲区的内存使用量以及客户端对象本身的内存占用。
if (listLength(server.slaves)) {
    listIter li;
    listNode *ln;

    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        mem += getClientOutputBufferMemoryUsage(c);
        mem += sdsAllocSize(c->querybuf);
        mem += sizeof(client);
    }
}
mh->clients_slaves = mem;

```

因为每个从节点都会分配一个独立的复制缓冲区（即从节点对应客户端的输出缓冲区），所以当从节点的数量增加时，这种实现方式会造成内存的浪费。不仅如此，当`client-output-buffer-limit`设置过大且从节点数量过多时，还容易导致主节点 OOM。


针对这个问题，Redis 7 引入了一个全局复制缓冲区。无论是复制积压缓冲区（repl\-backlog），还是从节点的复制缓冲区都是共享这个缓冲区。


`replBufBlock`结构体用于存储全局复制缓冲区的一个块。



```
typedef struct replBufBlock {
    int refcount;           /* Number of replicas or repl backlog using. */
    long long id;           /* The unique incremental number. */
    long long repl_offset;  /* Start replication offset of the block. */
    size_t size, used;
    char buf[];
} replBufBlock;

```

每个`replBufBlock`包含一个`refcount`字段，用于记录该块被多少个复制实例（包括主节点的复制积压缓冲区和从节点）所引用。


当新的从节点添加时，Redis 不会为其分配新的复制缓冲区块，而是增加现有`replBufBlock`的`refcount`。


相应地，在 Redis 7 中，mh\-\>repl\_backlog 和 mh\-\>clients\_slaves 的内存计算方式也发生了变化。



```
if (listLength(server.slaves) &&
    (long long)server.repl_buffer_mem > server.repl_backlog_size)
{
    mh->clients_slaves = server.repl_buffer_mem - server.repl_backlog_size;
    mh->repl_backlog = server.repl_backlog_size;
} else {
    mh->clients_slaves = 0;
    mh->repl_backlog = server.repl_buffer_mem;
}
if (server.repl_backlog) {
    /* The approximate memory of rax tree for indexed blocks. */
    mh->repl_backlog +=
        server.repl_backlog->blocks_index->numnodes * sizeof(raxNode) +
        raxSize(server.repl_backlog->blocks_index) * sizeof(void*);
}
mem_total += mh->repl_backlog;
mem_total += mh->clients_slaves;

```

具体而言，如果全局复制缓冲区的大小大于`repl-backlog-size`，则复制积压缓冲区（`mh->repl_backlog`）的大小取 `repl-backlog-size`，剩余部分视为从库使用的内存（`mh->clients_slaves`）。如果全局复制缓冲区的大小小于等于 `repl-backlog-size`，则直接取全局复制缓冲区的大小。


此外，由于引入了一个 Rax 树来索引全局复制缓冲区中的部分节点，复制积压缓冲区还需要计算 Rax 树的内存开销。


# 数据驱逐的触发条件


很多人有个误区，认为只要 used\_memory 大于 maxmemory ，就会触发数据的驱逐。但实际上不是。


数据被驱逐需满足以下条件：


1. maxmemory 必须大于 0。
2. maxmemory\-policy 不能是 noeviction。
3. 内存使用需满足一定的条件。不是 used\_memory 大于 maxmemory，而是 used\_memory 减去 mem\_not\_counted\_for\_evict 后的值大于 maxmemory。


其中，`mem_not_counted_for_evict`的值可以通过 INFO 命令获取，它的大小是在`freeMemoryGetNotCountedMemory`函数中计算的。



```
size_t freeMemoryGetNotCountedMemory(void) {
    size_t overhead = 0;
    int slaves = listLength(server.slaves);

    if (slaves) {
        listIter li;
        listNode *ln;

        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            client *slave = listNodeValue(ln);
            overhead += getClientOutputBufferMemoryUsage(slave);
        }
    }
    if (server.aof_state != AOF_OFF) {
        overhead += sdsalloc(server.aof_buf)+aofRewriteBufferSize();
    }
    return overhead;
}

```

`freeMemoryGetNotCountedMemory`函数统计了所有从节点的复制缓存区、AOF 缓存区和 AOF 重写缓冲区的总大小。


所以，在 Redis 判断是否需要驱逐数据时，会从`used_memory`中剔除从节点复制缓存区、AOF 缓存区以及 AOF 重写缓冲区的内存占用。


# Redis 内存分析脚本


最后，分享一个脚本。


这个脚本能够帮助我们快速分析 Redis 的内存使用情况。通过输出结果，我们可以直观地查看 Redis 各个部分的内存消耗情况并识别当 `used_memory` 增加时，具体是哪一部分的内存消耗导致的。


脚本地址：https://github.com/slowtech/dba\-toolkit/blob/master/redis/redis\_mem\_usage\_analyzer.py



```
# python3 redis_mem_usage_analyzer.py -host 10.0.1.182 -p 6379
Metric(2024-09-12 04:52:42)    Old Value            New Value(+3s)       Change per second   
==========================================================================================
Summary
---------------------------------------------
used_memory                    16.43G               16.44G               1.1M                
used_memory_dataset            11.93G               11.93G               22.66K              
used_memory_overhead           4.51G                4.51G                1.08M               

Overhead(Total)                4.51G                4.51G                1.08M               
---------------------------------------------
mem_clients_normal             440.57K              440.52K              -18.67B             
mem_clients_slaves             458.41M              461.63M              1.08M               
mem_replication_backlog        160M                 160M                 0B                  
mem_aof_buffer                 0B                   0B                   0B                  
used_memory_startup            793.17K              793.17K              0B                  
used_memory_scripts            0B                   0B                   0B                  
mem_hashtable                  3.9G                 3.9G                 0B                  

Evict & Fragmentation
---------------------------------------------
maxmemory                      20G                  20G                  0B                  
mem_not_counted_for_evict      458.45M              461.73M              1.1M                
mem_counted_for_evict          15.99G               15.99G               2.62K               
maxmemory_policy               volatile-lru         volatile-lru                             
used_memory_peak               16.43G               16.44G               1.1M                
used_memory_rss                16.77G               16.77G               1.32M               
mem_fragmentation_bytes        345.07M              345.75M              232.88K             

Others
---------------------------------------------
keys                           77860000             77860000             0.0                 
instantaneous_ops_per_sec      8339                 8435                                     
lazyfree_pending_objects       0                    0                    0.0       

```

该脚本每隔一段时间（由 `-i` 参数决定，默认是 3 秒）采集一次 Redis 的内存数据。然后，它会将当前采集到的数据（New Value）与上一次的数据（Old Value）进行对比，计算出每秒的增量（Change per second）。


输出主要分为四大部分：


* Summary：汇总部分，used\_memory \= used\_memory\_dataset \+ used\_memory\_overhead。
* Overhead(Total)：展示 used\_memory\_overhead 中各个细项的内存消耗情况。Overhead(Total) 等于所有细项之和，理论上应与 used\_memory\_overhead 相等。
* Evict \& Fragmentation：显示驱逐和内存碎片的一些关键指标。其中，mem\_counted\_for\_evict \= used\_memory \- mem\_not\_counted\_for\_evict，当 `mem_counted_for_evict` 超过 `maxmemory` 时，才会触发数据驱逐。
* Others：其他一些重要指标，包括 keys（键的总数量）、instantaneous\_ops\_per\_sec（每秒操作数）以及 lazyfree\_pending\_objects（通过异步删除等待释放的对象数）。


如果发现`mem_clients_normal`或`mem_clients_slaves`比较大，可指定 \-\-client 查看客户端的内存使用情况。



```
# python3 redis_mem_usage_analyzer.py -host 10.0.1.182 -p 6379 --client
ID    Address            Name  Age    Command         User     Qbuf       Omem       Total Memory   
----------------------------------------------------------------------------------------------------
216   10.0.1.75:37811          721    psync           default  0B         232.83M    232.85M        
217   10.0.1.22:35057          715    psync           default  0B         232.11M    232.13M        
453   10.0.0.198:51172         0      client          default  26B        0B         60.03K         
...  

```

其中，


* Qbuf：输入缓冲区的大小。
* Omem：输出缓冲区的大小。
* Total Memory：该连接占用的总内存。


结果按 Total Memory 从大到小的顺序输出。


 本博客参考[樱花宇宙官网](https://yzygzn.com)。转载请注明出处！
