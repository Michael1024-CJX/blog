-   Date: 2022-07-15
-   Author: Michael
-   Keyword: #redis 

# redis操作

## string

### set
`set key value [nx|xx]`
set会覆盖已存在的值。
#### 参数
##### nx
只在key不存在的时候成功设置。
##### xx
只在key存在的时候成功设置。
### get
`get key`

### getset
`getset key new_value`
设置新值，并返回旧值

### mset
`mset key value [key value]...` 
一次设置多个值

### mget
`mget key [key]...`
一次获取多个值

> mset和mget可以将n次网络通信降低至一次。 


### msetnx
`msetnx key value [key value]...` 
一次设置多个值，只有所有键都不存在的情况下才设置成功。

### strlen 
`strlen key`
获取值的字节长度。

### 字符索引
redis为字符串提供了一系列索引操作，同时支持正数索引和负数索引。
正数索引，从左到右，从0开始。
负数索引，从右到左，从-1开始。

### getrange
`getrange key start end`
获取字符串从start索引开始到end索引为止的内容。（包含start 和 end 位置）

### setrange
`setrange key index substitute`
将字符串的值从索引index开始的部分替换为指定的新内容。setrange会自动扩容，如果设置的新子串与原来的子串之间存在空字节，会已`\x00`填充

### append
`append key suffix`
将给定内容追加到字符串键已有值的末尾。如果key不存在，那么append会先将键的值初始化为空字符串，然后在追加。效果与set相似。


### 数字值
redis会对存入的值进行检测，如果是long型或者double型，redis会将其存储为数字值，分别为整数，和浮点数。
整数的范围在64位有符号整数中，即-2^63 - 2^63-1之间。
浮点数的取值范围在128位的有符号浮点数中，即

### incrby/decrby
`incrby key increment`
`decrby key increment`
当字符串值为`整数`时，可以通过incrby/decrby指令执行加减法。
如果键不存在，则会先初始化为0，在执行操作。

### incr/decr
`incr key`
`decr key`
对整数加减1.

### incrbyfloat
`incrbyfloat key increment`
将一个浮点数值加到一个数字值上。注意不存在`decrbyfloat`，只能通过传入负数值实现减法。
如果计算的结果是整数，会以整数存放。
最多保留小数点后17位数字。


## hash
将一个键与一个散列关联，散列中可以存在任意多的键值对。

### hset
`hset hash field value`
将 field:value 与hash关联，如果field对应的value不存在会新增，返回1，否则会覆盖，返回0。

### hsetnx
`hsetnx hash field value`
只在字段不存在的情况下为它设置值。

### hget
`hget hash field`

### hincrby
`hincrby hash field increment`
对字段存储的整数值执行加法或减法操作，不存在减法指令。

### hincrbyfloat
`fincrbyfloat hash field increment`

### hstrlen
`hstrlen hash field`
获取字节长度。

### hexists
`hexists hash field`
判断字段是否存在散列中。

### hdel
`hdel hash field`
删除散列中的字段。

### hlen
`hlen hash`
获取散列的字段数量。



## Stream
Redis用来实现消息队列的一种数据结构。

### XADD
`xadd stream id field value [field value]`
将带有指定Id的键值对元素追加到流的末尾，如果流不存在会新建。

流中的每个元素可以包含一个或任意多个键值对，并且同一个流中的不同元素可以包含不同数量的键值对。而且元素的键顺序会保持原序。

#### 流元素ID
流元素的ID由毫秒时间（millisecond）和顺序编号（sequcen number）两部分组成，顺序编号以0开始。

Redis还要求新元素的ID必须比流中所有已有元素的ID都要大。具体来说，Redis会记住每个流已有元素的最大ID，并在用户尝试向流里面添加新元素的时候，使用新元素的ID与流目前最大的ID进行对比。

可以通过 `*` 来指定ID由Redis自动生成，这也是官方推荐的方式。生成的方式是采用系统当前时间+顺序编号的方式实现的。如果当前已存在的ID比系统时间更大，则会采用该ID的时间戳，并对序号取+1值。

#### 限制流长度
`xadd stream [maxlen len] id field value [field value]`
指定maxlen len，可以限制流的长度，如果加入流元素后导致长度超过指定的限制，则会移除流头部的节点。

### XTRIM
`xtrim stream maxlen len`
将指定的流，裁剪成指定的大小，移除流头部的元素。（插入元素是插入流的末尾）

### XDEL
`XDEL stream id [id..]`
删除流中指定的元素。

### XLEN
`XLEN stream`
返回流的长度

### XRANGE、XREVRANGE
`xrange stream start-id end-id [count num]`
根据范围获取流中的元素。 支持特殊符号 `-` 和 `+` ，分别表示流中最小的元素和最大的元素。
count指定范围获取的数量。

XREVRANGE是XRANGE的逆序指令。

### XREAD
`XREAD [BLOCK ms] [COUNT n] STREAMS stream1 stream2 stream3 ... id1 id2 id3 ...`

XREAD支持`阻塞`的方式，获取流元素，但只能从指定的id向后获取，获取数量由count决定。xread支持同时获取多个流中的元素。如果设置了阻塞，至少其中一个流中返回元素时才会返回。

支持特殊符号`$`，该符号表示只获取指令执行后，新增的元素

### XGROUP
管理消费者组的指令
#### CREATE
`XGROUP CREATE stream group start_id`
对指定的`stream`创建组group，组内成员将从`start_id`开始读取流元素。start_id可以是具体ID，也可以是`$`，表示从最新的消息开始，`>`，表示从0开始。

流元素对同组的消费者是唯一的，对不同组的消费者是共享的。

一个流的组名唯一，不同流的组名可以相同，也就是有`流名` + `组名`的唯一键存在。

每个组都会有一个消费者名单，记录组下的消费者。
每个组都有一个待处理列表，表示被读取，但还未确认的情况。
每个组都会维护一个该组最后投递消息ID的值

#### SETID
`XGROUP SETID stream gourp start_id`
设置组的起始ID，可以使用`$`符号。

#### DELCONSUMER
`XGROUP DESTROY stream group`
删除组内的消费者，返回该消费者仍在处理的消息数量（在pending中的消息）。
当消费者被删除时，与他关联的待处理中的消息也会被删除，会照成消息丢失，因此需要保证删除消费者之前，需要保证消费者下的消息的正确处理。

#### DESTROY
`XGROUP DESTROY stream group`
删除消费者组，删除后组内的消息记录也会被删除。


### XREADGROUP
`XREADGROUP GROUP group consumer [COUNT n] [BLOCK ms] [noack] STREAMS stream [stream ...] id [id ...]`

通过组内成员来读取流，consumer表示成员名，不存在会自动创建，加入到组的消费者名单中。

被读取的消息，会被加入到组的待处理列表中，并与消费者关联。
`noack`表示读到的流元素不需要加入pending队列，即会自动ack，不需要手动ack。
id可以设置为特殊符号`>`，表示获取尚未向消费者组投递的第一条消息。

一条消费者组消息从出现到处理完毕，需要经历以下阶段：不存在；未递送；待处理；已确认。

### XPENDING 
`XPENDING stream group [start stop count] [consumer]`
查看组的待处理消息列表，可以指定消费者。
支持特殊符号`-`, `+`.

### XACK
 `xack stream group id [id]`
 对待处理列表中的流元素进行消息确认，确认后的流元素会从待处理列表中移除。
 
 ==注意该指令并不需要指定消费者，因此要注意可能被错误的消费者ack的情况。==
 
### XCLAIM
`XCLAIM stream group new_consumer max_pending_time id [id id ...] [JUSTID]`
该指令能将一组流元素，从一个消费者的pending列表移动到指定消费者的pending列表下面。并返回移动的元素信息，可以指定`JUSTID`表示只返回元素ID。

移动后流元素的pending时间会刷新，可以用来避免一条消息被并发移动的情况。

### XINFO
查看流和消费者组的相关信息.

#### CONSUMERS
`XINFO CONSUMERS stream group-name`
打印消费者信息

#### GROUPS
`XINFO GROUPS stream`
打印消费者组信息

#### STREAM
`XINFO STREAM stream`
打印流消息