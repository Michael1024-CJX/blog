-   Date: 2022-08-14
-   Author: Michael
-   Keyword: #redis 

# Redis数据类型

## String
`string`是redis中最常用的一种数据类型。可以存储字符串、整型、浮点型，值最大存储为512M。
当值为string时，type为`REDIS_STRING`，encoding根据值的类型，有`OBJ_ENCODING_INT`, `OBJ_ENCODING_EMBSTR`, `OBJ_ENCODING_RAW`三种。


> redis默认对`0-9999`的数值型字符串进行了缓存。如果新建的键值对中的值是这之间的，则会直接引用这些缓存中的数据。

### encoding 
#### OBJ_ENCODING_INT
如果一个字符串对象保存的是整数，并且这个整数值可以用long类型来标识（不超过long的范围），这时候字符串对象redisobject的指针将直接保存long数值（将void \* ptr转换成long）。
- 没有sdshdr对象。
- Long刚好跟指针的字节数一样, 存在ptr上，8字节。

#### OBJ_ENCODING_EMBSTR
当字符串对象中存储的是字符串，且长度小于 `44` （`Redis 3.2` 版本之前是 `39`）时，`Redis` 会选择使用 `embstr` 编码来存储。当string是embstr编码时，redis会一次性申请64字节的连续内存空间存放redisObject和sdshdr。
而其中redisObject占16字节，len，alloc，flag个占一个字节，加上字符结尾的 `\0`，共使用4字节。 因此 buf真正能存储数据的空间只有44字节。
![](Pasted%20image%2020220715163704.png)

#### OBJ_ENCODING_RAW
采用[简单动态字符串 SDS](Redis数据结构.md#简单动态字符串%20SDS)来存储字符串。

![](Pasted%20image%2020220715163620.png)


## Zset

### 底层数据结构
有序集合会根据一些条件判断是选择跳表还是压缩列表作为底层数据结构，有关的两个配置是：
- zet-max-ziplist-entries: 默认值128，表示列表中元素的最大个数。
- zet-max-ziplist-value: 默认值64，表示元素的字符串长度最大字节。

创建时，如果`zet-max-ziplist-entries`\=\=0 或者插入的字符大于`zet-max-ziplist-value`的值，才会选择使用[跳跃表(skipList)](Redis数据结构.md#跳跃表(skipList))作为底层数据结构，否则会默认使用[压缩列表(zipList)](Redis数据结构.md#压缩列表(zipList))。

在向zset插入元素时，zset中的元素个数大于`zet-max-ziplist-entries`的值，或者插入的元素长度大于`zet-max-ziplist-value`时，会将压缩列表转换为跳表。

### encoding
有序集合的编码可以是 `ziplist` 或者 `skiplist`。

`ziplist` 编码的有序集合对象使用压缩列表作为底层实现， 每个集合元素使用两个紧挨在一起的压缩列表节点来保存， 第一个节点保存元素的成员（member）， 而第二个元素则保存元素的分值（score）。
压缩列表内的集合元素按分值从小到大进行排序， 分值较小的元素被放置在靠近表头的方向， 而分值较大的元素则被放置在靠近表尾的方向。

`skiplist` 编码的有序集合对象使用 `zset` 结构作为底层实现， 一个 `zset` 结构同时包含一个[字典(dict)](Redis数据结构.md#字典(dict))和一个[跳跃表(skipList)](Redis数据结构.md#跳跃表(skipList))。
查找会通过字典表直接找到节点。插入，删除，更新则是通过跳表实现，并且会同步更新字典表。

## List
底层数据结构使用[快速列表（quickList）](Redis数据结构.md#快速列表（quickList）)实现。