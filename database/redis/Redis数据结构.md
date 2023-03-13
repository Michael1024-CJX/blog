-   Date: 2022-07-15
-   Author: Michael
-   Keyword: #redis 

# Redis数据结构
Redis中的数据结构实现大多都比较简单，且同时考虑到了空间复杂度和时间复杂度，是比较好的参考对象。

## 简单动态字符串(SDS)
`SDS`用于存储**字符串**和**整型**数据，兼容了C语言的字符串处理函数，并在此之上保证了[二进制安全](字符串.md#二进制安全)。并使用了**柔性数组**来存储字符串数据，内存查找更快。

>柔性数组：也叫伸缩性数组成员，只能被放在结构体的末尾，通过`malloc`函数为柔性数组动态分配内存。
>柔性数组的内存地址和结构体是连续的，内存查找更快。

### SDS 与 char[] 的比较
Redis采用SDS(Simple Dynamic String，简单动态字符串)存储字符串，而不是直接使用C的char\[\]。
使用sds的理由：
- 因为char\*功能单一，不能高效的支持获取长度，追加等操作。SDS支持O(1)的获取长度。
- 通过对 `buf` 分配一些额外的空间，并通过free记录未使用的空间，可以有效减少字符追加时的内存分配问题。
- sds是二进制安全的。但Redis依然采取了`\0`结尾字符串，为了方便复用 `C` 语言字符串原生的一些API。

### 实现

在Redis 3.2之后，SDS又被细分为了`sdshdr5`、`sdshdr8`、`sdshdr16`、`sdshdr32`、`sdshdr64`5种实现。后面的数组表示该类型能够存储的最大长度的字符串幂数。

Redis通过`flags`属性来区分SDS的类型，flags占1byte大小。在**sdshdr5**中，其中前3bit表示类型，后5bit表示字符串大小，因此sdshdr5只能存储0到2^5-1长度的字符串，并且不允许扩容（扩容时强制转成sdshdr8）。而剩下4种SDS类型中，flags只用到了前3bit表示类型，后5bit不使用，数组已分配长度和已使用长度分别用1,2,4,8字节存储。
![](Pasted%20image%2020220715172203.png)

#### \_attribute\_ ((\_packed\_))
一般情况下，结构体会按其所有变量的**最小公倍数**做字节对齐，而用了`_attribute_ ((packed))`后，结构体则按**1字节**对齐。使用这个好处有两个：
1. 节省内存。虽然sdshdr8不会省空间，但sdshdr16, sdshdr32, sdshdr64 分别能够省下1, 3, 7个字节大小的空间。
2. 统一flags的获取方式。因为sds是返回buf指针给上层，因此需要考虑buf指针的获取方式，可以通过`char* sh + sizeof(sds)`获取到buf的指针，但如果想要获取flags，又需要减去字节对齐大小，而不同类型的字节对齐数量不一样，又需要单独的处理。而使用了`pached`后，只需要buf\[-1\]即可获取到flags。

#### 扩容
sdshdr5不支持扩容，扩容时会转成sdshdr8。
SDS在创建时并不会分配额外的数组，只有在执行`APPEND`的时候，才会额外分配一倍大小的内存空间，如果字符串大小已大于1M则分配1M。

--- 

## 跳跃表(skipList)
跳表是一种**有序**集合的实现，是一个分层有序链表结构，采用空间换时间的思想（**O(n)**），实现了O(logn)的查找和插入，且支持范围查找。

### 实现
节点的代码实现：
```c
typedef struct zskiplistNode {
	sds ele;
	double score;
	struct zskiplistNode *backward;
	struct zskiplisLevel {
		struct zskiplistNode *forward;
		unsigned int span;
	} level[];
} zskiplistNode;
```
- ele: 存储string类型的数据。
- score：用于存储排序的分值，即排序字段。
- backward：后退指针，用于从后向前遍历跳跃表。
- level：柔性数组，表示节点的层。数量为64个。新增节点时，随机生成1-64中的一个数，数越大概率越小，将节点加入数表示的节点的链表中。
- forward：指向本层下一个节点。
- span：forward指向的节点与本节点之间的元素个数，span值越大，跳过的节点个数越多。

跳表的代码实现：
```c
typedef struct zskipList {
	struct zskiplistNode *header, *tail;
	unsigned long length;
	int level;
} zskipList;
```
- header：头节点，头结点比较特殊，level数组元素为64个，且forward都指向null，score为0，ele为null。不计入总长。
- tail：尾节点。
- length：节点链表总长度。不计入头结点。
- level：跳表高度。1-64。

---

## 压缩列表(zipList)
压缩列表本质就是一个字节数组，是为了节约内存而设计的线性数据结构。
结构如下：
![](Pasted%20image%2020220717200804.png)

- zlbytes: 压缩列表的字节长度，占4个字节，因此压缩列表最多有**2^32-1**个字节。
- zltail: 压缩列表尾元素相对于压缩列表起始位置的偏移量，占4字节。
- zllen: 压缩列表的元素个数，占2个字节，因此压缩列表的元素个数无法超过2^16-1。
- entryX: 存储的元素。
- zlend: 压缩列表的结尾，占1个字节，恒为`0xFF`。

entry的结构：
![](Pasted%20image%2020220717201523.png)
- prev_entry_len: 前一个元素的字节长度，占1或5字节，如果前元素长度小于254字节，则用1字节，否则用5字节。用5字节的时候，第一个字节是固定的`0xFE`，后面4字节才是真正的元素长度。通过prev_entry_len就可以实现从尾到头的遍历。
- encoding：表示当前元素的编码，即value的数据类型。根据不同类型的数据，可能占1,2,5字节。通过encoding的前两位可以判断数据的类型，剩下的位表示value的长度。
- value：数据。

> prev_entry_len只能记录0-253字节的长度，因为0xFF（255）表示zlend，0xFE（254）开头的第一个字节表示prev_entry_len占5字节。

而经过解码后的entry(取出的entry)，其结构体`zlentry`共定义的7个字段，其他的四个字段是用于缓存前面的结果的，因为那些结果需要复杂的运算。
```c
typedef struct zlentry {
	unsigned int prevrawlensize; // 表示prev_entry_len占用的长度
	unsigned int prevrawlen;     // 表示prev_entry_len的内容
	unsigned int lensize;        // encoding占用的长度
	unsigned int len;            // 元数据内容的长度
	unsigned char encoding;      // 数据类型
	unsigned int headersize;     // 当前元素的首部长度，即prev_entry_len+encoding
	unsigned char *p;            // 当前元素首地址
} zlentry;
```

---

## 字典(dict)
字典是redis最重要的数据结构，底层是通过数组加列表实现的。初始大小为4，扩容为当前大小的一倍，掩码是size-1，由于size总是2的幂次方，所以mask的值二进制各个位都是1，用来和key的hash值进行`&`运算的散列效果更好，Java中HashMap的实现[Hash](HashMap.md#Hash)也是类似的。Hash冲突采用拉链头插法解决。

字典源码：
```c
typedef struct dict {
	dictType *type;          // 不不同场景下的字典，抽象出来的操作函数
	void *privdata;          // 字典依赖的数据，配合type使用
	dictht ht[2];            // 存储Hash表，以及扩容时会使用到另一个
	long rehashidx;          // -1表示没有在rehash，否则表示rehash到了哪个元素

	unsigned long iterators; //当前运行的迭代器数
}
```

hash表源码实现：
```c
typedef struct dictht {
	dictEntry **table;        // 数组指针,即hash桶的指针
	unsigned long size;       // table的大小，桶的大小
	unsigned long sizemask;   // 掩码，size - 1,用于取余计算
	unsigned long used;       // 已存的元素个数。	
} dictht;
```

### dictEntry
redis中，每一个键值对都是被包装成`dictEntry`对象镜像存储，dictEntry拥有两个属性分别代表`key`和`value`，其中value又会被封装成[redisObject](#redisObject)。

Hash表节点源码：
```c
typedef struct dictEntry {
	void *key;           // 键指针
	union {
		void *val;       // 值
		uint64_t u64;    
		int64_t s64;     // 过期时间
		double d;     
	} v;
	struct dictEntry *next;  // hash冲突时的链节点
} dictEntry;
```


![](Pasted%20image%2020220715135617.png)

### redisObject
redisObject是redis用来封装value的对象。redis是通过`type`和`encoding`这两个变量来确定值的类型。
type表示值的数据类型，有`REDIS_STRING`,`REDIS_LIST`,`REDIS_HASH`,`REDIS_SET`,`REDIS_ZSET`等 。
encoding针对不同的数据类型又有不同的值，因为对于同一种数据类型，Redis允许其底层是多种数据结构，比如string类型就有3种编码int, embstr, raw。通过encoding来表示底层的具体数据结构。整个redisObject占16字节。
redis通过`refcount`实现对象的回收，创建对象时refcount初始化为1，当被引用时+1，不被引用是-1，当为0时，表示可回收。
```c
typedef struct redisObject {
	unsigned type:4; //对象类型（4位=0.5字节） 
	unsigned encoding:4; //编码（4位=0.5字节） 
	unsigned lru:LRU_BITS; //记录对象最后一次被应用程序访问的时间（24位=3字节） 
	int refcount; //引用计数。等于0时表示可以被垃圾回收（32位=4字节） 
	void *ptr; //指向底层实际的数据存储结构，如：sds的柔性数组、list等(8字节) 
} robj;
```

### hash函数
客户端的hash函数使用的是`times33`，核心算法是`hash(i) = hash(i-1)*33+str[i]`，是针对字符串已知的最好的散列函数之一。而服务端的hash函数是`siphash`，其优点是针对有规律的键计算出来的hash值也具有强随机分布性。

### hash冲突
redis通过链地址法解决hash冲突。

### 扩缩容
**扩容**：当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：
- 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1;
- 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5；

**收缩**： 当负载因子小于等于0.1时，会执行收缩操作。

扩容是一倍大小，而缩容是当使用量不到总空间的10%时触发的，缩容至恰好大于used的2的幂次方大小。

redis采用的是**渐进式rehash**，在操作字典时，如果发现字典在rehash中，则每次只rehash一个节点，空闲时则批量rehash 100个节点。避免一次rehash大量数据，造成进程阻塞(redis是单线程的)。

---

## 整数集合（intset）
当集合的元素类型都是整数并且在64位有符号整数范围内时，redis会使用intset来存储，并且保证元素有序且不会重复。
如果向其中插入字符串，或者大于64位整型，则会升级为hashtable。

```c
typedef struct intset {
	uint32_t encoding;    // 编码类型，决定每个元素的长度
	uint32_t length;      // 元素个数
	int8_t contents[];    // 柔性数组
} intset;
```

由encoding决定每个元素占用的字节数。如果插入的值大于当前元素能表达的长度，则该集合内的所有元素都会发送扩容。

## 快速列表（quickList）

quicklist是redis中List的底层实现，是一种将[ziplist](#压缩列表(zipList))和双向列表结合起来的一种数据结构，即由ziplist充当节点的双向链表。
```c
typedef struct quicklist {
	quicklistNode *head;
	quicklistNode *tail;
	unsigned long count;    // 总zipEntry数量
	unsigned long len;      // quicklistNode数量
	int fill : 16;          // 表示每个ziplist最多含有的数据项
	unsigned int compress : 16;  // 压缩参数
}
```

```c
typedef struct quicklistNode {
	struct quicklistNode *prev;
	struct quicklistNode *next;
	unsigned char *zl;                // 指向ziplist的指针
	unsigned int sz;                  // 表示整个ziplist的大小
	unsigned int count;               // 表示ziplist元素的个数，压缩格式的ziplist不保存自己的长度
	unsigned int encoding;            // 编码，1表示原生ziplist，2表示压缩格式
	unsigned int container;           // 
	unsigned int recompress;
	unsigned int attempted_compress;
	unsigned int extra;
}
```






