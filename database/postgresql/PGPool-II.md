# PGPool-II

## 安装
官方下载rpm包。
执行`rpm -ivh rpm包`即可安装。
安装PgPool-II需要依赖`libmemcached.so.11()(64bit)`和`libpq.so.5()(64bit)`,如果缺少依赖，可以执行
```shell
yum install libmemcached
yum install libpq
```