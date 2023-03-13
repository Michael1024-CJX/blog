# MongoDB学习记录

## 什么是MongoDB

​        `MongoDB` 是一个 `No-SQL` ，文档型数据库，以 `BSON` 格式存储数据格式，不受关系型数据库的表格式的约束，拓展性更高，适合存储数据格式会变化数据。

​        开源，但也有企业版。

## MongoDB 特性

###  数据格式

​		以 `BSON` 格式存储数据可以说是 `MongoDB` 最大亮点之一，这种格式的数据`No-Schema`，`BSON` 格式是 `JSON` 的二进制格式，使用起来和 `JSON` 一样，不过`MongoDB` 可以检索 `BSON` 中的数据内容。`BSON`采用`name:value`的格式存储数据，字段的值可以包括其他文档、数组和文档数组。当你想要存储如下数据时：

| name | price | kind | desc                 |
| ---- | ----- | ---- | -------------------- |
| 篮球 | 85 ￥ | 球类 | 周长 75cm，重量 650g |
| 跳绳 | 20 $  | 绳类 | 长度 2.5m            |

​		在上述的数据中，似乎我们使用传统的关系型数据库也能够进行保存，对于`描述`这一列，可以使用`String`，或者外表存储，使用`String`进行存储时，我们没有办法直接检索出描述中 “重量” 的数据，而使用外表存储，则显得麻烦。但是在 `BSON` 这种数据格式中，可以直接检索出这种数据。以下是以`BSON` 格式进行存储的数据：

```json
{
    "name":"篮球",
    "price":{"num":85,"unit":"￥"},
    "kind":"球类",
    "desc":[{"周长":"75","unit":"cm"},{"重量":"650","unit":"g"}]
},{
    "name":"跳绳",
    "price":{"num":20,"unit":"$"},
    "kind":"绳类",
    "desc":[{"长度":"2.5","unit":"m"}]
}
```

Key的最大长度是1024 byte，document的最大长度是16M。

###  GridFS

​		GridFS是用于存储和检索超过bson文档大小限制(16mb)的文件的规范。

> Note:
>
> ​		GridFS不支持多文档之间的事务关系。

​		GridFS将文件分块存储，默认情况下，每块的大小是255KB（除了最后一块），GridFS使用两个`collection`来保存文件，一个存储文件元数据，一个存储文件分块。

### 支持多种存储引擎

- [WiredTiger Storage Engine](https://docs.mongodb.com/manual/core/wiredtiger/)
  - Document Level Concurrency：不同的文档的写操作不上锁，相同文档的读写操作，只有在发生冲突时，在内部使用cas并发控制（乐观锁）。
  - 一些全局操作会给数据库上锁。
  - 使用多版本并发控制（MVCC）保证并发情况下的数据安全。
  - 占用内存：内存减1GB的50% 或者 256MB，取二者之间大的一项。

- [In-Memory Storage Engine](https://docs.mongodb.com/manual/core/inmemory/)

### 高性能

- 支持嵌入式的文档类型，减少对磁盘的I/O。
- 索引支持更快的查询，可以包含来自嵌入文档和数组的键。

### 高可用

- 自动故障修复
- 数据冗余灾备

## MongoDB 使用场景



## MongoDB缺点

- 

