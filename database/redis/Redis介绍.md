-   Date: 2022-07-14
-   Author: Michael
-   Keyword: #redis #db

# Redis
## 介绍
**Remote Dictionary Server**（远程字典服务）
redis是一款高性能的非关系型内存数据库，存储key-value键值对形式的数据。
操作是原子性的，数据存储在内存中，单线程，不涉及线程切换。
redis具有高性能的主要原因：
1) Redis是基于内存的存储数据库，绝大部分命令处理只是纯粹的内存操作。（注意避免swap分区）
2) Redis是单进程线程的服务（指只有一个线程来处理网络请求），避免了线程上下文切换，也不存在线程竞争问题。
3) Redis使用**多路复用I/O模型**(select, poll, epoll)，可以高效处理大量并发连接。
4) Redis的[数据结构](Redis数据结构.md)经过专门设计，经过大量的优化。

## 数据类型

### 基本数据类型
Redis主要有5种基本数据类型，包括： String、List、Set、ZSet、Hash。
#### String
`string`是redis的一种数据类型，可以存储字符串、整型、浮点型，值最大存储为512M。

string 存储整型的时候，其编码是int，直接使用8字节的变量存储值，因此整型的范围是正负2^63-1。
当存储的整型大于8字节或者存储字符串时，根据字符串长度，又有embstr和raw两种编码，分别使用字符数组和SDS进行存储。

embstr编码支持存储小于**44**字节的字符串，大于44字节的会转为raw编码，底层数据结构是**SDS**，SDS有多种实现，用于适配不同长度的字符串。

> embstr为44字节的原因是：
> 1. 当string是embstr编码时，redis会一次性申请64字节的连续内存空间存放redisObject和sdshdr。
> 2. redisObject = 16byte = type 4bit + encoding 4bit + lru 3byte + refcount 4byte + ptr 8byte。
> 3. 而redis对于embstr编码的sds采用的是`sdshdr8`存储。len, alloc, flag 各一字节，

```c
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

#### List
存储列表，列表允许重复。
底层的数据结构可以是quicklist。

#### Set
存储无序集合，集合不允许重复。
底层数据结构可以是intset 或者 hashtable
#### Zset
有序集合。插入元素时需要指定`score`，score是一个64位的双精度浮点数。
底层数据结构可以说ziplist或者skiplist+dict

#### Hash
存储一组键值对，类似JSON，适合存储对象。
底层的数据结构可以是ziplist或者hashtable。

### 特殊数据类型
-   Geospatial：存储地理坐标。
-   Hyperloglog：用于计数的工具，使用少量的内存。
-   Bitmap：字节数组。
-   Stream：流，用于实现消息中间件。


## 持久化机制
### RDB（默认）
以全量的方式将数据库进行持久化，每次创建新的`dump.rdb`，替换旧的文件。
有两种方式触发RDB持久化，一种是执行`save`指令，该指令会阻塞服务，将当前数据进行持久化。第二种是执行`bgsave`，该指令会创建子进程来执行`save`命令，子进程会异步执行。也可以通过配置文件中的save参数来定义快照的周期，配置文件本质是bgsave指令。

```conf
save 60 1000 # 60s内有1000个key发生变化则进行RDB持久化
```

#### 优点
- 只有一个文件 `dump.rdb`，方便备份。
- 容灾性好，一个文件可以保存到安全的磁盘。
- 性能最大化，fork 子进程来完成写操作，让主进程继续处理命令，所以是 IO 最大化。使用单 独子进程来进行持久化，主进程不会进行任何 IO 操作，保证了 redis 的高性能。
- 相对于数据集大时，比 AOF 的启动效率更高。
#### 缺点
- 数据安全性低。RDB 是间隔一段时间进行持久化，如果持久化之间 redis 发生故障，会发生数 据丢失。所以这种方式更适合数据要求不严谨的时候)

### AOF
即Append Only File。以增量的方式保存Redis执行的每一个写命令。当Redis重启是会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当两种方式同时开启时，数据恢复Redis会优先选择AOF恢复。

```conf
appendonly yes
appendfsync everysec #每秒执行一次fsync操作,可选值有always,everysec,no
```

#### 优点
- 数据安全，aof 持久化可以配置 appendfsync 属性，有 always，每进行一次命令操作就记录到 `appendonly.aof` 文件中一次。
- 通过 append 模式写文件，即使中途服务器宕机，可以通过 redis-check-aof 工具解决数据一 致性问题。
- AOF 机制的 rewrite 模式。AOF 文件没被 rewrite 之前（文件过大时会对命令 进行合并重写），可以删除其中的某些命令（比如误操作的 flflushall)
#### 缺点
1.  AOF 文件比 RDB 文件大，且恢复速度慢。
2.  数据集大的时候，比 rdb 启动效率低。

#### AOF重写
当AOF文件越来越大时，可以配置AOF重写。AOF重写不会读取或修改原有文件，而是fork一个子进程，对所有数据库中所有键值生成一条执行指令，生成新的AOF文件，重写过程中父进程执行的指令会追加到文件中。

```conf
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64m
```
以上配置表示当AOF文件大于64MB时，并且AOF文件比基数增大一倍时，触发重写，基数是启动时AOF的大小，或每次重写后AOF文件的大小。


### 混合持久化
混合持久化是指AOF重写时，子进程保存当前快照为RDB文件，而父进程累积的指令保存为AOF格式，因此一个文件中会有两部分。
如果AOF文件
```conf
aof-use-rdb-preamble yes  # 开启混合持久化
```

---

## 过期策略
定期删除+惰性删除。redis并不会一直监听每个key是否过期，因为这样比较占cpu资源，而是采用定期删除+惰性删除的机制。

**定期删除**是redis将每个设置了过期时间的key放到一个单独的字典中，每隔100ms随机抽查20个key(代码写死)，判断key是否过期，如果过期就删掉，如果过期key的比率超过1/4（5个），那就再次循环抽查。redis为了保证定时删除不会出现循环过度，导致线程卡死现象，为此增加了定时删除循环流程的时间上限，默认不会超过 25ms。

**惰性删除**是指对于设置了过期时间的key，在下次获取的时候就会进行判断是否过期，如果过期了就删掉。


### 存在的问题
存在一种情况是一直没抽到过期的key，也没访问它，这样就不会被释放而一直占用内存，内存会越来越高，当redis的运行内存超过了maxmemory，就会触发[Redis内存淘汰](Redis内存淘汰.md)机制解决。



