### rabbitmq kafka 区别

优先选择RabbitMQ的条件：

- 高级灵活的路由规则；
- 消息时序控制（控制消息过期或者消息延迟）；
- 高级的容错处理能力，在消费者更有可能处理消息不成功的情景中（瞬时或者持久）；
- 更简单的消费者实现。

优先选择Kafka的条件：

- 严格的消息顺序；
- 延长消息留存时间，包括过去消息重放的可能；
- 传统解决方案无法满足的高伸缩能力。

**kafka**

主要特点就是基于Pull模式的处理消息消费，追求高吞吐，一开始的目的是日志收集和传输，对消息的重复丢失错误没有严格的要求，适合产生大量数据的数据收集业务，有ack机制能够保证不丢失

高效的读写基于page cache ，不存在内存和磁盘之间的io，使用zookeeper管理集群，

**rabbitmq**

rabbitmq是erlang语言编写，基于amqp协议实现的消息中间件，可以用来解耦异步削峰

基本概念 exchange queue binding routing key

dirct  

fanout 

topic

TTL 生存时间 消息的过期时间在消息 发送时指定，在创建交换机时指定。

死信队列：当消息在一个队列中成为死信能够被重新publish到另一个交换机，就是dlx

消息死信的几种情况：消息被拒绝，消息TTL过期，队列达到最大长度

