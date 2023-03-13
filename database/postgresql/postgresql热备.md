- Date: 2021-11-25
- Keyword: #运维 #数据库 #PostgreSQL

# PostgreSQL热备

## What
复制即复制数据库，用于做到数据库热冗余。当一台数据库挂掉之后，可以保存数据并及时切换到备用库上。

### WAL日志
WAL(Write-Ahead Logging) 日志记录数据库的变化，格式为二进制格式。
WAL 的核心概念是对数据文件（表和索引所在的位置）的更改必须仅在这些更改被记录后写入，即在描述更改的日志记录已刷新到永久存储之后。

使用WAL会显着减少磁盘写入次数，因为只需要将日志文件刷新到磁盘以保证提交事务，而不是事务更改的每个数据文件。

如果WAL文件已经写入成功，但还没来得及刷新数据文件时当主机出现异常断电，则当数据库再次启动时会根据WAL日志文件信息进行事务前滚，从而恢复数据库到一致性状态。

### 物理复制（流复制）
pg从9.0版本开始支持物理复制，也称为`流复制`，流复制可以从主库复制一个一模一样的从库，并保证主从之间的数据一致。主库可读写，从库只可读。流复制是复制整个PG实例。

流复制有`同步`和`异步`两种，同步是会等备库接收`WAL日志流`并写入到备库[[#WAL日志]]成功时才完成事务，而异步则不需要等待备库的接收返回，从而提高效率。但是异步流复制的情况下，如果主库异常宕机，主库上已提交的事务可能还没来得及发送给备库，就会造成备库数据丢失。同步流复制则至少收到一个备库响应确认信息时才返回成功，保证了至少一个备库收到主库发送的WAL日志，但也增加了事务的响应时间。

> 同步复制中，如果备库宕机，会导致主库因为等待备库的响应而阻塞。所以，如果是一主一从的情况下不建议使用同步模式，只有一主多从的情况下可以考虑使用同步模式，这样只要不是所有从库都宕机，主库就不会被阻塞。


### 逻辑复制（选择性复制）
pg10.0版本开始支持逻辑复制，基于表级别的。可以选择需要复制的表进行复制。逻辑复制只能同步DML(修改数据的)，不能同步DDL(修改表结构的)。逻辑复制主从皆可读写。

### 两者的主要差异
- 流复制核心原理是将主库预写日志`WAL日志流`发送给备库，备库收到`WAL日志流`后进行重做。逻辑复制也是基于`WAL`，逻辑复制会根据预先设置的规则解析`WAL`，当解析到指定表的DML时，主库会将解析到的数据变化信息发送给备库。流复制是直接复制数据，逻辑复制是将数据解析成SQL发送，然后在备库重新执行。这样对于同一条记录，流复制的ctid是相同的，逻辑复制的ctid是不同的。
- 流复制是对整个PG实例进行复制，逻辑复制能只选择需要的数据表进行复制。
- 流复制能对DDL和DML进行复制，逻辑复制只能复制DML。
- 流复制主库可读写，从库只能读，逻辑复制主从皆可读写。
- 流复制要求主从大版本一致，逻辑复制支持跨版本。


## How
### 异步流复制
1. 修改`$PGDATA/postgresql.conf`文件中的配置。
```config

wal_level = replica # 9.x版本为hot_standby
archive_mode = on                
archive_command = '/bin/date'  
max_wal_senders = 16
wal_keep_segments = 512
hot_standby = on
```
- wal_level：wal日志的级别。`minimal`级别最低，`logical(hot_standby)`级别最高，开启流复制至少需要`replica(archive)`级别
- archive_mode: 开启归档模式，该模式下会将wal日志文件备份，备份动作由achive_command决定。
- archive_command：通过改命令备份WAL日志，流模式下归档不是必须的，所以这里随意填写了一个伪命令。
- max_wal_senders: 可发送wal的线程数，根据主库的性能配置，不能低于从库的数量，不能大于max_connections参数的值，默认为10。
- wal_keep_segments： 设置主库pg_wal目录保留的最小的wal日志文件数，以便当备库落后主库时可以通过主库备份的WAL进行追回。理论上该值越大，备库在异常断开后追平主库的机会越大。每个WAL默认为16MB,因此512\*16MB表示pg_wal目录会占用8GB
- hot_standby: 开启备库的读功能，off的情况下备库提供读功能。该配置只对备库有效，但由于主备身份是可以切换的，所以主库也配置该值，待主库变成备库时也能提供读操作。

配置`pg_hba.conf`,将备库地址和备份用户配置上。（注意：Database中ALL不包括replication，所以replication需要另外配置）主库IP:192.168.28.74, 备库IP:192.168.28.75
```config
host    replication     postgres        192.168.28.74/32            md5
host    replication     postgres        192.168.28.75/32            md5
```
正常情况下，只需要在主库配置一条备库的记录即可，这里配置两条的记录的原因是：主备库的角色是可以互换的，为了统一配置，建议主备库的pg_hba.conf配置完全一致。

2. 同步数据文件目录`$PGDATA`。
从库可以通过pg_basebackup指令来将主库上的数据文件同步下来。
```
pg_bascbackup -D $PGDATA -Fp -Xs -v -P -h 192.168.28.74 -p 5432 -U repuser
```
但有一些文件不是必须的，如果想要更快速的同步，可以让主库开启在线备份状态`select pg_start_backup('francs_bk1');`,然后将`$PGDATA`下的文件压缩（如果有额外的表空间也需要拷贝），并排除`pg_logs`,`pg_wal`目录，将压缩包传到从库解压。拷贝完成后，主库执行`select pg_stop_backup();`。

3. 备库上在`$PGDATA`下新建`recovery.conf`文件，或者从`$PGHOME/share/recovery.conf.sample`复制。
修改以下配置:
```config
restore_command = 'copy "%p" "D:\\postgresql\\9.2.3\\data\\archivedir\\%f"'
archive_cleanup_command = 'pg_archivecleanup "D:\\postgresql\\9.2.3\\data\\archivedir\\%f"'
recovery_target_timeline = 'latest'
standby_mode = on
primary_conninfo = 'host=192.168.28.75 port=5432 user=repuser password=abc123 application_name=node1'
trigger_file = 'recovery.fail'
```
-   restore_command: 用于当备库处理wal日志来不及时，临时将wal存储到指定路径下，路径需要手动创建。
-   archive_cleanup_command：用来清理不再需要的wal，配置与`restore_command`相同的路径。
-   recovery_target_timeline: 如果有多台备库，删除注释，用来同步备库之间的时间。
-   standby_mode：表示该库为备库模式。
-   primary_conninfo：备份用户连接主库的信息。
-   trigger_file：用来设置终止备库继续备份的文件，改文件被创建在data文件夹下时，会终止备份的工作。

将密码明文记录在`primary_conninfo`中不太合适，建议将密码配置在`.pgpass`中，内容如下所示，并修改文件的权限为0600以保护密码安全。

```config
192.168.28.74:5432:replication:repuser:abc123
192.168.28.75:5432:replication:repuser:abc123
```

>在PostgreSQL 12中，已经没有recovery.conf文件了，而是用standby.signal文件所代替，且原来需要在recovery.conf文件中配置的primary_conninfo参数，已经融合在postgresql.conf中。

### 同步流复制
整体的操作与配置与`异步流复制是相同的`,只是需要添加一些额外的配置.
recovery.conf
```
primary_conninfo = 'host=192.168.28.75 port=5432 user=repuser password=abc123 application_name=node2'
```
postgresql.conf
```
synchronous_commit=on
synchronous_standby_names='node2' # 与上面的application_name值一致
```
synchronous_commit参数的可选值为：on, off, local, remote_apply, remote_write。
- off: 事务提交后，WAL值需要写入WAL BUFFER(内存中)后就返回成功。如果数据库意外宕机会丢失部分事务。
- local: 单实例下，事务提交时需等待本地WAL BUFFER写入磁盘后才返回成功。
- on: 单实例下，与local表现一致。 流复制下，需要等待备库接收WAL并写入WAL文件才返回成功。
- remote_write: 流复制下，需等待备库接收WAL并写入WAL BUFFER中后返回。备库实例异常关闭不会丢失数据，但备库操作系统宕机会丢失。
- remote_apply: 流复制下，需等待备库接收WAL并写入WAL文件，并且备库已执行WAL才返回成功。
