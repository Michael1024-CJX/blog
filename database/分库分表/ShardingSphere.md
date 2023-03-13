-   Date: 2022-08-24
-   Author: Michael
-   Keyword: #db 

# ShardingSphere
[ShardingSphere官方文档](https://shardingsphere.apache.org/document/current/cn/)描述的非常详细，包括整个分库分表的架构。

ShardingSphere是一个Apache顶级项目。由 [ShardingSphere-JDBC](#ShardingSphere-JDBC) 和 [ShardingSphere-Proxy](#ShardingSphere-Proxy) 这 2 款既能够独立部署，又支持混合部署配合使用的产品组成。

提供了**数据分片**，读写分离，分布式事务，数据加密，数据迁移等功能。

## ShardingSphere-JDBC
ShardingSphere-JDBC 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

每个应用维护自己的连接池，访问的数据源由ShardingSphere-JDBC实现，用户只需要配置好库以及分片的策略。

![](Pasted%20image%2020220824165407.png)


## ShardingSphere-Proxy
Proxy定位为透明化的数据库代理端，单独部署，目前提供 MySQL 和 PostgreSQL 协议，透明化数据库操作，对 DBA 更加友好，Navicat可以直接连Proxy。

应用层不需要任何改动，所有操作由Proxy代理完成，对项目的影响最小。但需要额外对Proxy做高可用部署，并且多了一次网络请求。

![](Pasted%20image%2020220824170404.png)

## 对比JDBC和Proxy

|      | JDBC  | Proxy |
|  ----  | ----  | ---- |
| 数据库  | 任意 | MySQL/PostgreSQL |
| 连接消耗数  | 高 | 低 |
| 语言 | Java | 任意 |
| 性能 | 损耗低 | 损耗较高 |
| 无中心化 | 是 | 否 |
| 静态入口 | 无 | 有 |


## 混合架构
ShardingSphere-JDBC适用于OLTP 应用（写），ShardingSphere-Proxy适用于OLAP应用（读）。

但个人觉得，应用端如果需要考虑引入依赖的成本，还是选择使用JDBC，Proxy用来运维。

![](Pasted%20image%2020220824170746.png)


## 使用限制

### 支持
- DML（CRUD）、DDL（数据库操作）、DCL（权限操作）、TCL（事务操作）。
- 分页、去重、排序、分组、聚合、表关联等复杂查询。
- 子查询。
- 跨域关联查询

### 不支持
- CASE WHEN
- Oracle的rownum + BETWEEN分页查询。
- SQLServer的 WITH xxx AS (SELECT …) 的方式进行分页。由于 Hibernate 自动生成的 SQLServer 分页语句使用了 WITH 语句，因此目前并不支持基于 Hibernate 的 SQLServer 分页。 目前也不支持使用两个 TOP + 子查询的方式实现分页。

### 分页实现

## 原理
### 路由
![](Pasted%20image%2020220913171424.png)


## 实践操作

### 数据迁移


### 操作分片


### 基因ID分片


