
什么是分布式锁？

分布式锁是控制分布式系统或不同系统之间共同访问共享资源的一种锁实现，如果不同的系统或同一个系统的不同主机之间共享了某个资源时，往往需要互斥来防止彼此干扰来保证一致性。
分布式锁需要具备哪些条件？

互斥性：在任意一个时刻，只有一个客户端持有锁。
无死锁：即便持有锁的客户端崩溃或者其他意外事件，锁仍然可以被获取。
容错：只要大部分Redis节点都活着，客户端就可以获取和释放锁
分布式锁的主要实现有哪些？

数据库
Memcached（add命令）
Redis（setnx命令）
Zookeeper（临时节点）
单机Redis的分布式锁

加锁语句：

jedis.set(key, value, nxxx, expx, time)
key-键;value-值;nxxx-nx(只在key不存在时才可以set)|xx(只在key存在的时候set);expx--ex代表秒，px代表毫秒;time-过期时间，单位是expx所代表的单位。
解锁语句

jedis.del(key)
主要要解决的几个问题

设置锁的过期时间，解决持有锁的客户端A挂了却一直持有锁，导致B客户端无法获得锁。
如何保证锁不会被误删除，通过设置锁的value，使用lua脚本保证释放锁的原子性，防止出现i++问题
过期时间如何保证大于业务执行时间，设置定时开启刷新过期时间线程，业务完成，释放锁时同时取消定时刷新过期时间任务
可重入性

可重入性是指线程在持有锁的情况下再次请求加锁,如果一个锁支持同一个线程的多次加锁，那么这个锁就是可重入的，比如Java里面ReentrantLock就是可重入锁。Redis分布式锁如果要支持可重入，需要客户端的set方法进行包装，使用线程的Threadlocal变量存储当前持有锁的计数。 还需要考虑内存锁计数的过期时间。
集群Redis的分布式锁

在Redis的分布式环境中，Redis 的作者提供了RedLock 的算法来实现一个分布式锁。
加锁

获取当前Unix时间，以毫秒为单位。
依次尝试从N个实例，使用相同的key和随机值获取锁。在步骤2，当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。
解锁

向所有的Redis实例发送释放锁命令即可，不用关心之前有没有从Redis实例成功获取到锁.
总结

Java环境有 Redisson 可用于生产环境，但是分布式锁还是Zookeeper会比较好.
