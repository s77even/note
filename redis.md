## redis

八大特征：速度快，持久化，多数据结构，多语言，多功能，简单，主从复制，高可用分布式



速度快:redis将数据存储在内存，单线程操作，避免了上下文切换，采用了非阻塞IO多路复用机制，特有的数据结构和内部编码

**redis慢查询**

慢查询发生在命令的执行阶段，只会统计执行的时间，是由命令本身引起的。

两个参数slowlog

slower than  超过多少时间算慢查询，设置为0表示记录所有命令

max len	慢查询日志最多的存储条数，使用了一个列表来存储慢查询日志，设置为负数表示关闭日志功能··吗



#### 持久化

​				RDB（半持久化模式）：不定期的将数据快照保存到磁盘上（save 同步  bgsave异步 自动）

​				AOF（全持久化模式）：把每一次数据的变化操作以日志的形式记录下来（always everysec（默认） no（os 决定））**重写**

​		RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘，save是同步生成快照，会造成主业务的阻塞，bgsave实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。自动生成rdb文件是监控在一定时间内记录改变的条数，900s-1、300-10/60-10000 都会自动生成rdb来保证保存最新的数据（实际是内部执行了bgsave）

Rdb触发方式：不容忽略方式：1 全量复制 （主从） 2 reload  3 shutdown

​		AOF持久化以日志的形式记录服务器所处理的每一个写、删除操作，查询操作不会记录，以文本的方式记录，可以打开文件看到详细的操作记录。appendonly  yes  打开aof配置

**aof重写**   bgrewriteaof  fork 进行内存中数据的aof重写 生成新的aof文件

**aof重写配置 ** minsize （64mb）重写需要的最小尺寸    percentage （100）文件增长率   cursize 当前尺寸  basesize 上次aof启动和重写的尺寸 aofrewritebuff  在重写中新增加的新的日志命令 在完成重写后追加到重写完成的新aof文件中



RDB优势：灾难恢复比较简单，只需要将快照文件压缩后转移到其他存储介质上；性能最大化，fork出子进程完成持久化工作；数据集很大的话，RDB的启动效率要大于AOF

劣势：会造成数据丢失，宕机出现后，没来得及写入磁盘的数据都会丢失

AOF的优势：AOF可以带来更高的数据安全性，提供了三种同步策略：每秒同步，没修改同步，不同步；对日志文件的写入操作采用的append模式，写入过程出现宕机，也不会破换日志文件中存在的内容，redis还提供了redis-check-aof工具来进行日志修复；redis**AOF文件重写机制**可以高效的保存每一条指令，减小AOF文件的大小，加快恢复的速度；可以使用该文件完成数据的重建。

aof追加阻塞  everysec aof同步花费时间超过两秒会造成阻塞 等到同步完成  所以在此时如果发生宕机 会丢失两秒以上的数据



劣势：AOF文件要大于RDB文件大小，AOF恢复大数据集时的速度要更慢、

选择标准：性能更重要还是更高的缓存一致性更重要



fork的内存开销 写时复制 在fork时会共用父进程的内存 如果有大量的写操作  就会复制一份内存页 造成大量内存开销

fork 的阻塞  在内存不足的情况下， 申请fork子进程可能会造成阻塞



#### 数据结构 string hash set zset list （bitmap hperloglog boolenfilter GEO  ）

string：字符串的类型不能大于512MB，可用于缓存，计数器，分布式锁，分布式id生成器（自增操作原子性）底层实现简单动态字符串

set get del incr decr incrby decrby setnx setxx mget mset getset append strlen 



hash：可以用来存储对象，是一个键值对集合，string类型的field和value的映射表，在filed比较少，且各个value值也比较小的时候，hash采用ziplist实现，随着filed增多和value增大，hash会变成linklist来实现，效率会变低。

hget hset hdel hgetall hexist hlen hmget hmset



list：简单的字符串列表，按照插入顺序排序，可以在列表头部或尾部增添元素 ，可以实现简单的消息队列的功能，可以基于lrange 实现一个分页功能，底层采用ziplist或linklist实现

lpush lpop rpush rpop lrange lindex lset llen



set ：  Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据，底层使用了intset和hashtable两种数据结构存储，hashtable其实是value为空的hash

sadd scard srem sismember srandmember spop smembers sscan sinter sdiff sunion

 

zset多了一个权重参数score 集合中的所有元素能按照score进行排列，可以做排行榜，sorted set 底层实现的数据结构有skiplist ， ziplist

zset zadd  zcard zrem zscore zincrby zrange zrangebyscore zremrangebyscore zremrangebyrank zscan



### 底层数据结构

int raw embstr linklist ziplist skiplist hashtabel intset                                                                    



#### Redis 过期键的删除策略

###### redis采用懒惰删除+定时删除

立即删除：在设置键的过期时间时，会创建一个回调时间，当过期时间达到时，由时间处理器自动执行键的删除操作，能够保证内存中数据的最大新鲜度，但是删除操作会占用cpu的时间，会给cpu造成额外的压力，而且目前redis 事件处理器对时间的处理方式是采用无序链表的方式，查找一个key较为费时，所以不适合用来处理大量的时间事件



惰性删除：键值过期后，并不会被马上删除，等到下次使用时，会被检查是否超过过期时间，此时才能得到删除，会造成过期键也占用内存得不到释放



定时删除：立即删除会在短时间占用大量的cpu，惰性删除会浪费内存，所以定时删除是一个折中的方法，每隔一段时间执行一次删除操作，并通过限制删除操作的时间和频率 来减少对cpu的影响。





#### Redis数据淘汰策略

如果redis中数据非常多，将服务器的内存都耗尽，就会出现内存溢出的情况。redis使用**数据淘汰策略**解决内存溢出的问题

可以通过修改配置文件中的maxmemory修改最大内存设置



redis有6种淘汰策略

volatile-LRU(Least Recently Used)				从已设置过期时间的数据集中挑选最近最少使用的数据淘汰

volatile-TTL(time to live)								从已设置过期时间的数据集中挑选将要过期的数据淘汰

volatile-Random											从已设置过期时间的数据集中任意选择数据他淘汰

allkeys-LRU													从所有数据集中挑选最近最少使用的数据淘汰

allkeys-Random											从所有数据集中任意选择数据进行淘汰

noeviction													 不删除策略，达到最大内存限制时，如果需要更多内存，直接返回错误信息。





##### redis缓存穿透

客户端去查询一个一定不存在的值，缓存中不存在，数据库中也不存在，由于缓存不命中，数据库也查不到不会写入缓存，所以每次查询都要去数据库查询，失去了缓存的意义

解决方法：

1 布隆过滤器

bloomfilter类似于一个hash set，用于快速判断一个元素是否不存在于集合中，存在误判。

编写接口可以规范key 的命名规则，	不合法的key直接过滤掉，不让请求到达数据库

2 缓存空值

不管数据库中是否存在查询的数据，都在缓存中写入，没有查询到 值设为空，避免了查询不到的数据频繁访问数据库，但是空值太多会占据大朗内存，因此要对空值设置较短的过期时间，或定期清理空值，

3 限流和熔断降级

重要的接口一定要做好限流策略，防止用户恶意刷接口，同时要降级准备，当接口中的某些服务不可用时候，进行熔断，失败快速返回机制



##### redis缓存击穿

缓存击穿是指缓存中没有但数据库中有的数据，一般是缓存时间到期，这是由于并发量较大，同时读缓存没有读取导数据，又同时去数据库读取数据，造成数据库压力过大。

解决方案：

1 设置热点数据永不过期

2 加互斥锁

从缓存中读取缓存不存在的数据时，先获取锁再去读取数据库，读取到数据后更新缓存



##### redis缓存雪崩

缓存雪崩是指大批量数据达到过期时间，而查询数量巨大，造成数据库压力过大

解决方案：

1. 缓存数据的过期时间设置随机，防止同一时间大量数据过期现象发生。
2. 如果缓存数据库是分布式部署，将热点数据均匀分布在不同搞得缓存数据库中。
3. 设置热点数据永远不过期。



#### redis 事务



#### [redis和memcache 的区别](https://blog.csdn.net/ThinkWon/article/details/101530406)  



