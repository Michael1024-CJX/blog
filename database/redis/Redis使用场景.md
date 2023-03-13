-   Date: 2022-08-11
-   Author: Michael
-   Keyword: #redis 

# Redis使用场景

## 缓存
Redis中比较常见的使用场景，可以使用[String](Redis介绍#String)，[Hash](Redis介绍.md#Hash)用来缓存数据库中的数据，避免有太多请求直接访问数据库。

## 排行榜
可以实现如歌曲排行榜，购买排行榜，收藏排行榜等。
使用[Zset](Redis介绍#Zset)来实现，将用于排序的元素作为`score`，来存储数据。通过`ZRANGE`或`ZREVRANGE`来范围获取排行榜中的元素，`ZINCRBY`来增加元素的score，`ZRANK`或`ZREVRANK`来获取元素的排名。

排行榜中的元素更新比较频繁，尤其是排名靠前的元素。因此并不适合存储在数据库中。

## 投票
使用[Zset](Redis介绍#Zset)来实现投票功能，可以避免重复投票，并具有排序功能。

## 时间线
使用[Zset](Redis介绍#Zset)来实现时间线，将时间戳作为`score`，就可以通过`ZRANGBYSCORE`来获取指定时间内的对象。

## 唯一ID
可以使用`INCY`来实现全局递增的唯一ID。

## 计数器
- 使用`INCY`和`INCRBY`来实现计数器，
- 使用[Set](Redis介绍.md#Set)来实现唯一计数器，并且可以获取被计数的对象。但是数据量大的时候比较占用内存。
- 使用HypeLogLog实现唯一计数器，占用内存小，但无法获取被计数的对象。

## 滑动窗口限流


## 分布式会话
可以使用[Hash](Redis介绍.md#Hash)存储会话信息。

## 分布式锁


## 最新列表
如果需要考虑时间元素，则可以使用Zset，如果不需要，可以使用List

## 消息系统


## 布隆过滤器
使用BitMap来实现布隆过滤器。

