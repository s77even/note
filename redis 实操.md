### redis 实操

**主从复制**  数据备份 读写分离 高可用  一主多从 一主一从 slave只能有一个master  数据流向只能 从master到slave

实现方式：slaveof   配置

salveof 命令方式 ： 在主client上执行命令  salveof 127.0.0.1:6380  返回OK（但异步执行） 完成数据的复制（全量复制）并且从节点会将自身原先所有数据清除

salveof no one 断开主从复制关系  在从节点上执行命令slaveof no one 

配置文件方式：在配置文件中添加 slaveof ip port  同时配置 slave-read-only yes 从节点只能读

info replication 查看redis主从复制状态

**全量复制和部分复制**  

**runid** redis cli的唯一标识 重启后会发生变化

**offset** 主从同步的标志  写入数据后发生偏移 主从偏移一致说明数据同步  **部分复制的重要依据**

**全量复制**  首先将本身的rdb文件同步给slave，在同步过程中的新增数据单独记录 通过之后的偏移量对比 将新的数据同步给slave

![image-20210302200611835](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210302200611835.png)

首先第一次要求主从复制  由于不知道主节点runid 和偏移地址  会发送psync ？ -1

主节点收到后会要求slave进行全量复制 fullresync 并告知slave自己的runid 和 当前偏移地址

从节点保存主节点地址  主节点开始bgsave 生成rdb文件 并同时将新的数据记录到rep_back_buffer（1M）中

将rdb文件和buffer中记录发送给从节点 从节点刷新掉自己原本拥有的数据 加载rbd文件  完成数据最终同步

**全量复制的开销** bgsave时间消耗  rbd文件网络传输 slave节点清空自身数据 加载rdb



**部分复制**  针对在全量复制中因网络连接断开而丢失的数据

![image-20210302201322834](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210302201322834.png)

同步过程中 网络断开，断开这期间的数据记录都会丢失，从节点重新连接到主节点，会发送psync加上自己的偏移量，

主节点会去对比当前偏移量是否在自己buffer范围内 ，如果不在 说明丢失数据过多 会发生一次全量复制

如果在自己buffer范围内 会将offset到最新的记录之间的数据同步给slave



**主从复制中故障处理**

故障转移   从节点宕机  问题不大

​				主节点宕机  ：在一个从节点中执行 slaveof no one 升级为主节点 并将其他从节点的主更改为新的主节点（sentinel 自动转移）



**主从复制常见问题**  主从配置不一致 规避全量复制 规避复制风暴

配置不一样： 主从maxmemory不一致 ： 会造成丢失数据

 						数据结构优化参数不一致  

规避全量复制：第一次全量复制不可避免  节点运行不匹配  （节点故障重启后 会进行一次全量复制 推荐使用sentinel和cluster）

​							复制缓冲区不足  部分复制无法满足 会发生全量复制

规避复制风暴：**复制风暴**：单主节点重启后 从节点全部都要复制主节点数据 对网络传输和cpu开销压力较大

​							sentinel 架构 





### **redis sentinel**

redis sentinel节点： 监控 多个 高可用 公平  不执行存储  可以执行ping  sentinel彼此能够感知

客户端从sentinel获取redis信息  sentinel知道主从节点信息 

**故障转移**： 自动完成  多个sentinel确认master出现问题 选举一个sentinel作为leader  选出一个从节点作为master并通知其他slave

老的master重启复活后称为新的master 的slave

**sentinel 主要配置**

sentinel monitor mymaster 127.0.0.1 7000 2   mymaster主节点名称 ip port quorum（2个sentinel认为master有问题即故障转移）

sentinel down-after-milliseconds mymaster 30000  （ms）  超过30000ms 通信不达 认为故障

sentinel parallel-syncs mymaster 1  复制是并发还是穿行 1表示串行

sentin failover-timeout mymaster  18000 故障转移时间



**sentinel启动会发生配置从写 修改sentinel配置文件  发现从节点**

**三个定时任务**

**1 每十秒每个sentin对master和slave执行info  发现slave节点  确认主从关系** 

**2 每两秒每个sentinel通过master节点的channel（sentinel:hello）和其他sentinel交换自身信息和对节点的看法 （pub/sub）**

**3 每一秒每个sentinel对其他sentinel和redis节点执行ping**



**主观下线** sentinel节点对redis节点失败的偏见   down-after-millisecond 超时未响应 

**客观下线**  超过quorum个数sentienl节点对redis节点失败达成共识 认为节点真的出现问题



**领导者选举**  raft算法

**故障转移** 由sentinel领导者节点完成

从slave节点中选举一个“合适的”节点作为新的master节点  对该节点执行slave of no one 让其成为master

向剩余slave节点发送命令，让他们成为新的master的slave，并复制数据

更新对原来master节点的配置为slave 对其保持关注 当期恢复后命令他去复制新的master的数据

选择一个**合适的**slave ： 

选择slave节点优先级最高的节点 

选择偏移量最大的slave 

选择runid最小的节点 启动最早



主节点下线：sentinel failover <mastername> 完成手动故障转移

从节点下线：临时下线和永久下线

节点上线：主节点 sentinel failover替换  从节点 slaveof no one即可



### **redis cluster**

呼唤集群：并发量 (10万/s)  数据量 （单机内存不够） 网络流量 

解决方式：分布式 （简单的认为加机器）

集群：规模化需求 

分布式数据库：数据分区 （顺序分区 哈希分区）

![image-20210303093924586](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303093924586.png)



**哈希分布**：节点取余 一致性哈希 虚拟槽分区

**节点取余**： 添加一个节点 会造成80%的数据迁移  建议多倍扩容（迁移50）   

**一致性哈希** 哈希环 顺时针决定临近节点  对节点取余的优化  节点伸缩只影响顺时针临近节点 

​                    一般情况还是使用翻倍伸缩  保证最小迁移数据的同时能够保证负载均衡

**虚拟槽分区**  预设虚拟槽 每个槽映射一个数据子集 槽一般比节点数大 良好的哈希函数（CRC16） 

![image-20210303095355577](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303095355577.png)



服务端管理服务节点  每个节点知道其他节点负责的槽16383的范围  节点之间通过gossip协议通信 meet操作

流言协议： ping pong meet fail   目的是为了维护节点之间的元数据信息，包含那些数据，是否出现故障等

meet 通知新节点加入集群中， 

ping 交换节点的元数据，每个节点每秒向集群中其他节点发送ping，

pong ping和meet的结合

fail 某个节点判断另一个节点挂掉了， 会用fail广播通知其他节点，来标记失败节点下线



cluser enable ： yes 开启集群模式

主从复制 高可用



**集群搭建**

原生命令安装    官方工具ruby安装 redis-cli --cluster 代替redis-trib

1. 配置开启节点  

   cluser enable ： yes 开启集群模式

   cluster config file

   cluster node timeout 15000 

   cluster require full coverage no 要求集群所有节点正常可用才对外提供服务 正常业务不开启   

直接开启后 集群处于下线状态 不能对外提供服务  clusterdown

cluster nodes 查看信息 包括nodeid



2 meet 

cluster meet ip port  建立通信

redis-cli -h 127.0.0.1 -p 7000 cluster meet 127.0.0.1 7001.....7005

3 指派曹 

cluster addslots {...}

4 主从分配

cluster replicate node-id



**集群伸缩**

扩展集群：

加入节点 新加入的节点必须和原来节点配置文件相同  

加入集群  使用meet操作加入集群 但是并没有立即开始开始  要等待迁移

**迁移槽和数据  reshard host:port**

redis-cli --cluster reshard 127.0.0.1:7000

![image-20210303111632598](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303111632598.png)

要求分配4096个槽给port7006 的id   并且使用all或者指定nodeid

会从指定槽或全部槽中 分配一部分给新增的节点

![image-20210303111921076](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303111921076.png)



收缩机群：

下线迁移槽

redis-cli --cluster reshard --cluster-form {nodeid} --cluster-to{nodeid1} --cluster-slots {nums} ip：port

redis-cli --cluster reshard --cluster-form {nodeid} --cluster-to{nodeid2} --cluster-slots {nums} ip：port

redis-cli --cluster reshard --cluster-form {nodeid} --cluster-to{nodeid3} --cluster-slots {nums} ip：port



忘记节点 并关闭节点

redis-cli --cluster del-node ip：port nodeid   指定ip端口的忘记指定nodeid的节点

先下线主节点 后下线从节点   **先下线主节点会触发从节点的故障转移**

![image-20210303113732625](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303113732625.png)

使用delnode忘记节点同时会关闭节点



**客户端路由**

![image-20210303142907469](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303142907469.png)

![image-20210303143043021](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210303143043021.png)







**故障转移**

故障发现 **通过节点之间的ping.pong消息实现故障发现 不需要依赖sentinel**

主观下线：某个节点认为另一个节点不可用  与某节点的最后通信时间超过node timeout 标记为pfail状态

客观下线：当半数以上持有槽的主节点都标记某节点主观下线，更新节点为客观下线，向集群广播下线节点的fail消息

  节点的ping消息中会包含其他节点的pfail节点信息



故障恢复：

检查资格：每个从节点检查与故障主节点的断线时间，超过timeout(15000ms)*cluster slave validity factor（10）取消资格

偏移量更大的 优先级更高的 更早进行选举 容易获得更多的选票

替换主节点：当前从节点变为主节点 slaveofnoone  撤销故障主节点负责的槽 并分配给自己  向集群广播自己的pong 表示替换成功



redis cluster 常见问题

集群完整性  带宽消耗 pubsub 数据倾斜 请求倾斜 读写分离 数据迁移 集群vs单机

集群完整性：   cluster requere full coverage 选项为yes时 集群槽必须全部可用 才对外提供服务

带宽消耗 ：官方建议 不要超过一千个节点  

​					pingpong消息 （发送频率（timeout/2） 数据量（槽数据 2KB 和集群1/10的状态数据  (十个节点状态约1KB)） ）			

PUB/SUB：pub在集群中每个节点广播 加重带宽的消耗  解决:单独布置一套sentinel解决发布订阅

集群倾斜：数据倾斜 请求倾斜

数据倾斜：节点和槽分配不均匀（rebalance）   不同槽对应的键值数量差异较大  包含bigkey 内存配置不一致

请求倾斜：热点key

集群读写分离：集群模式中的从节点不接受任何读写请求 重定向到负责槽的主节点

数据迁移：import 只能从单机迁移到集群 不支持在线迁移 单线程

​                  在线迁移：伪装成source节点的从节点获取数据 完成迁移

**集群和单机的对比**

key 批量操作支持有限   mget mset必须在一个slot

事务和Lua支持有限 操作的key'必须在一个节点

不支持多个数据库 在集群模式下只有一个db0

