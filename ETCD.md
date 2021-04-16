### ETCD

etcd是一个分布式的可靠的kv型数据库，底层采用了Raft算法作为共识算法，采用了基于gRPC的通信，使用go语言编写。(Go具有出色的跨平台支持（针对不同平台可编译称为不同的机器码），可以编译称为小的二进制文件，拥有强大的社区。)

etcd的存储方式采用了类似目录的结构，只有叶子节点才能真正存储数据，相当于文件，叶子节点的父节点一定是目录，目录不能存储数据

ETCD使用场景

键值对存储 服务注册与发现 消息发布与订阅



etcd 架构

![image-20210318210827050](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210318210827050.png)

boltdb kv模型的数据库，为不需要完整数据库服务器的项目提供一个简单可靠的数据库

wal  预写式日志 用于持久化存储的日志格式

snapshot 快照，存储etcd 数据

raft 保证分布式系统强一致性

grpc  为客户端提供可调用服务



client的http调用和grpc调用：

http调用：通过注册到http模块的处理方式法进行处理，将解析好的消息调用do方法进行处理。

grpc调用：通过向grpc注册的quotakvserver，grpc解析消息后调用kvserver的方法，kvServer中包含有一个RaftKV的接口，由EtcdServer这个结构实现。所以最后就是调用到EtcdServer的Range、Put、DeleteRange、Txn、Compact等方法



节点之间的grpc消息：每个EtcdServer中包含有Transport结构，Transport中会有一个peers的map，每个peer封装了节点到其他某个节点的通信方式。包括streamReader、streamWriter等，用于消息的发送和接收。streamReader中有recvc和propc队列，streamReader处理完接收到的消息会将消息推到这连个队列中。由peer去处理，peer调用raftNode的Process方法处理消息



分布式一致性算法：（强一致性算法）

Google的Chubby分布式锁服务，采用了**Paxos**算法

raft

ZooKeeper分布式应用协调服务，Chubby的开源实现，采用**ZAB**算法

### Raft算法 

etcd使用raft算法来保证集群中的数据的一致性，raft协议简单且容易实现

客户端发送到任何一个节点的请求都能够收到一致的返回，当一个节点出现故障后，其他节点仍然能以已有的数据正常进行

Quorum机制

集群中半数以上的节点可用时，集群才可继续提供服务

假设有 N 个副本，更新操作 wi 在 W 个副本中更新成功之后，则认为此次更新操作 wi 成功，把这次成功提交的更新操作对应的数据叫做：“成功提交的数据”。对于读操作而言，至少需要读 R 个副本，其中，W+R>N ，即 W 和 R 有重叠，一般，W+R=N+1。

raft状态机

<img src="C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201209220919360.png" alt="image-20201209220919360" style="zoom:50%;" />

raft的状态机结构简单，状态少，只有follower，candidate，leader

##### raft协议简要说明

1）在稳定状态，整个集群只有一个Leader节点，其他都是Follower节点。leader节点接收客户端的所有需求，follower会将写请求转发给leader，leader会将数据以日志的方式通过rpc方式同步给所有followers。
2）获取大部分支持票，表示支持票至少是集群节点数/2+1。
3）在相同任期Term下，每人只有一票，先发起投票的节点，先把票投给自己。
4）Raft为了保证选举成功概率，设置了一个选举定时器，选举定时器超时后则进入选举节点。由于每个节点定时器不一致，则提升选举的成功率。在极其特殊场景，才会出现定时器设置一样。当然这种概率也是可能的，但是我们可以人工干预（修改配置文件）。

#### 选主 Leader Election(为了保证日志复制的一致性)

##### 正常情况下选主

这是最简单的选主情况，**只要有超过一半的节点投支持票了，Candidate 才会被选举为 Leader**，5个节点的情况下，3个节点 (包括 Candidate 本身) 投了支持就行。

##### Leader 出故障情况下的选主

最先达到timeout的节点会发起投票，选中leader后，如果旧leader恢复了，会根据选举记录，旧的lead自觉降级为follower

##### 多个 Candidate 情况下的选主

存在可能多个follower的timeout在某一时刻同时到期称为cadidate，假设2，总节点数为4，两个candidate分别同时向一个follower发送了投票请求，并且两个follower分别返回了ok，这时两个candidate都只有两票，foloower的一票和自己的一票，但是一共需要三票才能够成功选举为leader，两个 Candidate 会分别给另外一个还没有给自己投票的 Follower 发送投票请求。但是因为 Follower 在这一轮选举中，都已经投完票了，所以都拒绝了他们的请求。所以在 Term 2 没有 Leader 被选出来。此时节点的状态是 Candidate，两个是 Follower，但是他们的倒计时器仍然在运行，最先 Timeout 的那个节点会进行发起新一轮 Term 3 的投票。两个 Follower 在 Term 3 还没投过票，所以返回 OK，这时 Candidate 一共有三票，被选为了 Leader。如果 Leader Heartbeat(leader向其他节点发送通知确认其他节点保持状态并重置节点的timeout)的时间晚于另外一个 Candidate timeout 的时间，另外一个 Candidate 仍然会发送选举请求。两个 Follower 已经投完票了，拒绝了这个 Candidate 的投票请求。Leader 进行 Heartbeat， Candidate 收到后状态自动转为 Follower，完成选主。

###### 拒绝投票的情况(挑选更好的领导者)

当一个cadidate发起投票请求时，请求里会包括自身的日记记录信息，信息索引以及该记录的任期号，投票者会比较日志信息，如果投票者的日志更完整，他会拒绝投票，最终赢得选举的服务器可以保证比大多数投票者拥有更完整的日志记录。

#### Raft复制日志 log Replication (为了达成一致性)

日志结构

![image-20201210112856911](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201210112856911.png)

###### 日志的特性

如果在不同的日志中两个日志条目的索引和下标相同 那么他们的命令就是相同的

如果在不同的日志里两个日志条目拥有相同的任期号和索引，那么他们之前的日志都是相同的

(一致性检查)

（每次 RPC 发送附加日志时，leader 会把这条日志条目的前面的日志的下标和任期号一起发送给 follower，如果 follower 发现和自己的日志不匹配，那么就拒绝接受这条日志，这个称之为**一致性检查**）



###### 正常情况的日志复制

客户端发送请求到leader，leader先将数据写到本地日志，但并未确认(uncommited)。leader给其他follower发送appendentry请求，(appendentry 不包含数据时是心跳消息)经过一致性检查后，如果数据在follower没有冲突，follower则将数据暂时写到本地，返回给leader ok，leader收到多数返回ok后，将数据在本地状态改为committed，并返回客户端确认，leader再次向follower发送appendentry请求，follower的uncommited数据改为committed，完成一条数据的日志复制。

###### 日志不一致时(leader崩溃)

正常情况下 leader和follower的日志进度满足一致性检查，即上条已提交的index和任期匹配，日志不一致可能出现跟随者可能丢失记录，跟随者可能有不同的记录，需要做的是剔除所有不同的日志记录，并将所有丢失的记录根据领导者的日志填充完整

![image-20201210155107035](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201210155107035.png)



###### 客户端协议

客户端将命令发给领导者，如果接受者不是领导者，会告知客户端并将客户端你重定向到领导者，然后再次发送请求，领导者记录命令并且将其变为commited状态发送给状态机执行后，才会将结果返回给客户端。

(此时可能发送领导者的崩溃，此时结果还没有返回给客户端，客户端认为命令执行失败，会再次发送请求给新选举出来的领导者，会导致命令被执行了两次的风险。

解决方案：让客户端为每条命令生成一个唯一的 ID ，并将其与命令一起发送给领导者，当领导者记录该条命令时，也会包括这个唯一 ID ，但在领导者接受命令之前，它会进行检查，看其他记录中是否已存在相同的 ID ，如果存在相同的，那么它就会知道该条命令请求是多余的，所以它会找到该条记录，并忽略这条新命令，并将老的执行结果返回给客户端。)



##### 两段协议（??)

所有分布式决策必须使用的方式



#### GO的编译

静态编译：不依赖动态库 编译好后只要平台一致 就可以任意部署

交叉编译：在一个平台上编译生成另一个平台所需要的二进制文件 可通过GOOS=...设置

指定架构：目标平台的体系架构 通过GOARCH=386,amd64,arm设置

【在linux下编译amd64位windows可执行程序】

【交叉编译时 cgo选项要关闭】CGO_ENABLED=0 GOOS=windows GOARCH=amd64 go build main.go



#### etcd参数入口

（newconfig）默认配置 命令行 配置文件 环境变量 从另一个etcd集群中读取（discovery）



##### etcd两种运行模式 startetcd

##### server

proxy（4层  7层）



main->etcdmain.Main()->1,checkSupportArch 2,判断是否通过命令行传入配置  3启动服务 

->(etcdmain)startEtcdOrProxyV2 根据配置文件决定调用startEtcd还是startProxy->startEtcd 创建两个通道并阻塞等待创建完成或失败的通知 ->embeb.StartEtcd->etcdserver.NewServer 创建新的etcd服务端和监听段并加入集群 ->Server.Start() 



##### channel

etcd使用了很多channel 

大部分使用了无缓冲channel  并且逻辑比较简单 如阻塞等待通知 <img src="C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201209165454869.png" alt="image-20201209165454869" style="zoom: 80%;" />

防止了channel使用逻辑复杂而造成channel频繁阻塞带来性能的影响



#### GET(BoltDB)



#### etcd 定时器

选举定时器     1000ms

心跳定时器      100ms

etcd中只有一个定时器的实现 但是实现了两种逻辑的管理 利用节点不同的类型回调不同的tick回调函数完成不同的逻辑

<img src="C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201211111144436.png" alt="image-20201211111144436" style="zoom:70%;" /><img src="C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20201211111238785.png" alt="image-20201211111238785" style="zoom:67%;" />





###### etcd raft状态机

状态机的三种写法：case方式 数组方式 数组+函数指针方式  etcd中使用了case方式  因为状态较少

etcd中raft有四种状态

```go
var stmap=[...]string{
    "StateFollower",
	"StateCandidate",
	"StateLeader",
	"StatePreCandidate",/* 预置Candidate */
}

```





