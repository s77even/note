### Memcache

redis:支持比较多的数据类型(String/list/set/sortset/hash)，redis支持集合计算的(set类型支持)，每个key最大数据存储量为1G，redis是新兴的内存缓存技术，支持持久化操作。

memcache：老牌的内存缓存技术，对相关领域支持比较丰富，window和linux都可以使用，各种框架(tp/yii等等)都支持使用，session的信息可以非常方便的保存到该memcache中，每个key保存的数据量最大为1M，支持的数据类型比较单一，就是String类型，不支持持久化。



**redis:其为主从模式，一个redis负责数据写入，其他多个redis负责数据读取**

**memcache:其不是主从模式，该分布式是平均分摊工作，每个子服务器之间都是平级的，每个服务器都要执行数据的写入、读取操作**。



# 缓存失效

超过有效期：具体是通过“懒惰”机制删除该过期数据，与过期session的删除类似。

过期session删除机制：session是以文件形式保存的硬盘中，如果有的session文件已经过期了，则该session文件不会立即被删除，而是后期其他用户访问网站使用session的同时会有一定的几率触发删除过期的session文件。

memcache的过期数据删除也是懒惰机制实现，如果有一个key过期了，其本身不会马上被删除，而是我们调用get方法获取数据的同时会删除该过期的数据。

缓存空间耗尽

如果存储的数据超过memcache最大的存储限制(默认是64M)，此时还继续存入数据，则会把最近不常使用的key就删除了。该机制名称为LRU(least recently use)优先删除最近很好使用的key。

该LRU机制可以根据实际情况禁用，如果继续使用满载的memcache则系统要报错。