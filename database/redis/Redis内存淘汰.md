-   Date: 2022-08-15
-   Author: Michael
-   Keyword: #redis 

# Redis内存淘汰


## 基本配置
配置项：maxmemory-policy

- no-enviction（驱逐）：禁止驱逐数据，新写入操作会报错
- allkeys-lru：从数据集（server.db[i].dict）中挑选最久未使用的数据淘汰
- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最久未使用的数据淘汰
- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- allkeys-lfu：从所有键中驱逐使用频率最少的键
- volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键

### LRU
Redis3.0引入了缓冲池（堆）作为LRU算法的实现。当每一轮移除key的时候，随机拿取几个key（默认5个，maxmemory-samples）。将这几个key中，idel time最大的key与池中idel time最大的key比较，如果更大则放入池中，当池满后，直接取池中key的idel time最大的，将其删除。

开始淘汰前会计算出需要释放的内存大小，redis会循环上述操作，直到释放到指定的内存大小。

### LFU
LFU算法是Redis4.0里面新加的一种淘汰策略。它的全称是Least Frequently Used，它的核心思想是根据key的最近被访问的频率进行淘汰，很少被访问的优先被淘汰，被访问的多的则被留下来。

使用LFU算法时，对象头的lru字段会被拆成两段使用，高16位ldt记录key的访问时间戳，低8位logc记录key的访问频次，注意频次并不是次数，频次是次数与时间的比值。