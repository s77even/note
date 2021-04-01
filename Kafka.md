### Kafka

观察者模式：发布订阅模式

定义对象间一对多的依赖关系，当对象改变状态，所有依赖于他的对象都会得到通知自动更新。

生产者消费者模式：

通过个中介来解决生产者消费者的强耦合问题，生产者消费者不直接通信，通过队列进行通信。

缓冲中介:

解耦 支持并发 支持忙闲不均

消息系统原理：

一个消息系统负责将数据从一个应用传递到另一个应用，只关注于数据，无需关注传递过程。

点对点消息传递：

发布订阅消息传递：消息被持久化到一个topic中 

**kafka架构**

**broker** 集群中一个或多个服务器，服务器节点称为broker 相当于一个节点

**topic** 每条消息有一个类别 称为topic 类似数据库的表名，物理上不同的topic分开存储

**partition** topic中的数据被分割为一个或多个partition，内部有序，尾部追加 

每个消息会有一个自增编号，表示顺序和消息的偏移量，每个partition'中的数据使用多个segment文件存储

如果需要严格保证消息的顺序消费场景，应该将分区设置为1

**leader** 每个patition有多个副本，有且仅有一个作为leader，负责数据的读写

**（producer 从zk 中找到partition的leader节点，将消息发送给leader，leader将消息写入到本地log，follower从leader  pull消息，写入本地log，返回leader一个ack，leader收到所有follwer的ack，最后commit提交，并向producer发送ack）**

**follwer** 跟随leader，所有写都通过leader，数据变更会广播给所有follwer，保持数据同步

如果leader失效，会从follwer中选举一个leader，

如果follwer挂掉或者卡住或同步太慢，会把follwer从isr列表中删除，重新创建一个follwer

**replication** 数据的备份

producer

consumer

consumer group

offset 可以唯一的标识一条消息，偏移量决定了读取数据的位置，消息被消费后不会马上删除，默认生命周期为一周，可以通过修改偏移量达到重复读取的目的，偏移量由用户控制

zk zk存储集群中meta信息



kafka 数据检索机制

partition以segment0-n存储在磁盘中，形式是一个一个的文件夹，只有一个active

logsegment文件 由两部分构成，index文件和log文件，分别表示索引和数据

partition全局从第一个segement从0开始，后续每一个segement文件名为上一个加上最后一条消息的偏移

20位数组字符长度，没有数字用0填充

![image-20210316221800037](C:\Users\wwwwwwl\AppData\Roaming\Typora\typora-user-images\image-20210316221800037.png).

消息具有固定的物理结构，包括偏移小消息体大小校验等可以确定一条消息的大小，读到哪里截止



**数据的安全性**

**producer**可以选择是否为数据的写入接受ack，

0 at most once 不保证消息发送成功， 不用等待集群ack返回  会丢数据 不会重复数据

1 只要leader 应答成功就发送下一条， 只确保leader接受成功

-1 all **at-least-once** 所有follower完成同步才发送下一条   不丢数据 会重复数据



**exactly once 语义**    要求既不丢失也不重复  			-1+幂等

启用 idompotence=true

开启幂等性的producer在初始化的时候会分配一个pid，发送到同一partition的消息会带有一个序列号，这些数据被持久化，pid partition seq，具有相同缓存的消息提交，只会持久化一条。

如果消息序号比seq大2以上，说明数据乱序有数据未写入，会拒绝消息

比seq小，说明重复

但是pid重启后会重新分配，所以**无法保证跨分区跨回话**的一个exactly once

**跨分区跨会话原子写入    事务**

 引入了一个全局唯一的事务id，这个id通过客户端来进行设定，并且将pid和tid进行绑定，当producer重启后根据tid去找到自己对应的一个pid，因此还引入了一个组件coordinator来维护这样一个tid和pid的关系，



**ISR机制**

AR  标识副本的全集

OSR 离开同步队列的副本

ISR 加入同步队列副本 

isr = leader + 没有落后太多的副本  ar = isr +osr

**isr** in sync replication

主节点挂掉后，从isr中选主   超过十秒没有同步 同步数据差4000条数据

脏节点选举  降级  不完美的从节点当选主节点 为了保证服务可用

**broker数据存储机制**

会保留所有消息，两种删除策略

基于时间  168 hours   基于大小 1G

**消费与提交**

consumer  offset  （ group + topic +partition）



**高效读写**

**顺序写** 磁盘IO

顺序写磁盘  producer的消息写入到log文件，采用追加写，顺序写。顺序写相对随机写，省去了寻址的时间。

 **零拷贝**

传统模式，文件传输需要多次拷贝和上下文的切换

DMA 为了防止在io时的cpu等待，产生了DMA，使用它来进行内存和io的数据传输

首先文件从硬盘拷贝到内核缓冲区，从内核缓冲区拷贝到用户缓冲区，从用户缓冲区拷贝到socket传输的相关缓冲区，再从socket拷贝到网卡。4次拷贝4次切换

mmap+write

mmap 直接利用操作系统的page来说实现文件到物理内存的直接映射，对物理内存的操作会在合适的时机被同步到硬盘，但是是不可靠的，调用flush才会落盘。

使用mmap实现了内核缓冲区读缓冲区和用户缓冲区之间的映射，内核缓冲区和用户缓冲区共享，减少了从内湖缓冲区到用户缓冲区的一次拷贝

sendfile

通过sendfile使数据直接在内核空间传输，减少了内核空间到用户空间的拷贝以及上下文切换，sendfile方法只适用于不需要用户额外处理数据的情况。

sendfile+DMA收集

又减少了一次内核空间中地缓冲区到socket的文件拷贝，



kafka优化

partition数目

replication factor 一般2  