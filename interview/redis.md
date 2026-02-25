## Redis
### 与mencache区别
- Redis不仅仅支持简单的k/v类型的数据，同时还提供list，set，zset，hash等数据结构的存储。memcache支持简单的数据类型，String。
- Redis支持数据的备份，即master-slave模式的数据备份。
- Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用,而Memecache把数据全部存在内存之中
- redis的速度比memcached快很多
- Memcached是多线程，非阻塞IO复用的网络模型；Redis使用单线程的IO复用模型。

### 淘汰
- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
- no-enviction（驱逐）：禁止驱逐数据

### 分布式锁
先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。

我记得set指令有非常复杂的参数，这个应该是可以同时把setnx和expire合成一条指令来用的！

SET key value [EX seconds] [PX milliseconds] [NX|XX]

### 找前缀
- 使用keys指令可以扫出指定模式的key列表。redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。
- 这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。


### 作为异步队列
一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。

也可以使用blpop

如果对方追问能不能生产一次消费多次呢？使用pub/sub主题订阅者模式，可以实现1:N的消息队列。

如果对方追问pub/sub有什么缺点？在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如rabbitmq等。

如果对方追问redis如何实现延时队列？我估计现在你很想把面试官一棒打死如果你手上有一根棒球棍的话，怎么问的这么详细。但是你很克制，然后神态自若的回答道：使用sortedset，拿时间戳作为score，消息内容作为key调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。

### 持久化
bgsave做镜像全量持久化，aof做增量持久化。因为bgsave会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要aof来配合使用。在redis实例重启时，会使用bgsave持久化文件重新构建内存，再使用aof重放近期的操作指令来实现完整恢复重启之前的状态。

bgsave的原理是什么？你给出两个词汇就可以了，fork和cow。fork是指redis通过创建子进程来进行bgsave操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。

### Pipeline有什么好处，为什么要用pipeline？
可以将多次IO往返的时间缩减为一次，前提是pipeline执行的指令之间没有因果相关性。使用redis-benchmark进行压测的时候可以发现影响redis的QPS峰值的一个重要因素是pipeline批次指令的数目。

在一般情况下，都会在Redis前面使用一个代理，来作负载以及高可用。这里在公司里面使用的是Codis，以Codis 3.2版本为例（3.2版本是支持Pipeline的）。

Codis在接收到客户端请求后，首先根据Key来计算出一个hash，映射到对应slots，然后转发请求到slots对应的Redis。在这过程中，一个客户端的多个请求，有可能会对应多个Redis，这个时候就需要保证请求的有序性（不能乱序），Codis采用了一个Tasks队列，将请求依次放入队列，然后loopWriter从里面取，如果Task请求没有应答，则等待（这里和Java的Future是类似的）。内部BackenRedis是通过channel来进行通信的，dispatcher将Request通过channel发送到BackenRedis，然后BackenRedis处理完该请求，则将值填充到该Request里面。最后loopWriter等待到了值，则返回给客户端。

### 主从同步
Redis可以使用主从同步，从从同步。第一次同步时，主节点做一次bgsave，并同时将后续修改操作记录到内存buffer，待完成后将rdb文件全量同步到复制节点，复制节点接受完成后将rdb镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。

### 集群
哨兵模式：m-s模式，m发生故障则提醒人力来调整

Redis Sentinal着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。写maste人，读所有。Redis较难支持在线扩容，在集群容量达到上限时在线扩容会变得很复杂。为避免这一问题，运维人员在系统上线时必须确保有足够的空间，这对资源造成了很大的浪费。（写瓶颈）

Redis Cluster着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储. 完全去中心化的设计。这带来的好处是部署非常简单，直接部署Redis就行，不像Codis有那么多的组件和依赖。但带来的问题是很难对业务进行无痛的升级，如果哪天Redis集群出了什么严重的Bug，就只能回滚整个Redis集群。 提升可用性：每个节点可以有多个从节点；事务性问题；增加客户端代理和zk等来保存节点状态信息


### 如何实现乐观锁、悲观锁
#### 乐观锁
大多数是基于数据版本（version）的记录机制实现的。即为数据增加一个版本标识，在基于数据库表的版本解决方案中，一般是通过为数据库表增加一个”version”字段来实现读取出数据时，将此版本号一同读出，之后更新时，对此版本号加1。此时，将提交数据的版本号与数据库表对应记录的当前版本号进行比对，如果提交的数据版本号大于数据库当前版本号，则予以更新，否则认为是过期数据。redis中可以使用watch命令会监视给定的key，当exec时候如果监视的key从调用watch后发生过变化，则整个事务会失败。也可以调用watch多次监视多个key。这样就可以对指定的key加乐观锁了。注意watch的key是对整个连接有效的，事务也一样。如果连接断开，监视和事务都会被自动清除。当然了exec，discard，unwatch命令都会清除连接中的所有监视。

#### Redis事务
Redis中的事务(transaction)是一组命令的集合。事务同命令一样都是Redis最小的执行单位，一个事务中的命令要么都执行，要么都不执行。Redis事务的实现需要用到 MULTI 和 EXEC 两个命令，事务开始的时候先向Redis服务器发送 MULTI 命令，然后依次发送需要在本次事务中处理的命令，最后再发送 EXEC 命令表示事务命令结束。Redis的事务是下面4个命令来实现 

- multi，开启Redis的事务，置客户端为事务态。
- exec，提交事务，执行从multi到此命令前的命令队列，置客户端为非事务态。
- discard，取消事务，置客户端为非事务态。
- watch,监视键值对，作用是如果事务提交exec时发现监视的监视对发生变化，事务将被取消。


### 一些骚操作
- https://juejin.im/entry/5720237071cfe400573e8919
- https://www.javazhiyin.com/21978.html