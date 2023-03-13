# Redis事务
redis提供了`multi`和`exec`命令来显示的开启与提交事务，discard放弃事务。

开启事务后，接下来的所有命令会先入队，当提交事务后才会执行。但提交后执行过程中失败的指令不会影响事务。

redis还提供了watch来实现乐观锁，watch可以监听多个key，只有当被监听的key未修改时，事务才会执行。当一个事务执行了exec或者discard后，watch会自动取消。unwatch来取消key的监听。

