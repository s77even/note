## Rabbitmq

应用解耦，提高容错性和可维护性

异步提速

削峰填谷

劣势：引入外部依赖越多，系统的稳定会变差，增加了系统的复杂度。

amqp 高级消息队列协议

![image-20210318165057896](C:\Users\seven\AppData\Roaming\Typora\typora-user-images\image-20210318165057896.png)

Broker 接收和分发消息的应用 一个rabbitmq就是一个broker

vhost 将amqp基本组件划分到一个虚拟的分组中，类似于namespace的概念，当不同的用户使用同一个rabbitmq 提供服务时，可以划分多个vhost，相当于隔离空间

connection 生产者和消费者与broker之间的tcp 长连接

channel 如果每一次访问都需要建立connnection，开销是巨大的，效率也较低。channel是在connection内部建立的逻辑连接

exchange message到达交换机，根据分发规则匹配表中的routing key 分发消息到queue中，常用类型有direct topic 和fanout

queue 消息最终被发送到队列中，等待消费

binding exchange与queue之间的虚拟连接，binding可以包含routing key，binding信息被保存到exchange中的查询表，用于message的分发依据



rabbitmq六种工作模式：

直连模式，工作队列，pub/sub,routing,topic,rpc



如何保证消息的可靠传输

```
消息不可靠的情况可能是消息丢失，劫持等原因；
丢失又分为：生产者丢失消息，消息队列丢失消息、消费者丢失消息
1、生产者丢失消息：从生产者弄丢数据这个角度来看，RabbitMQ提供了事务和消息确认来确保生产者不丢消息；
事务机制缺点：吞吐量下降；
消息确认机制：进入消息确认模式，在该信道上的消息都会呗指派唯一的ID(从1开始)一旦消息被投递到所有匹配的队列之后，rabbitmq就会放一个ACK给生产者（包含消息的唯一ID），这就使得生产者知道消息已经正确到达目的队列。
如果RabbitMQ没能处理该消息，就会发送Nack消息，可以重试操作
2、消费者丢失消息;消费者丢失数据一般是因为采用了自动确认消息模式，改为手动确认消息即可！
消费者在收到消息之后，处理消息之前，会自动回复RabbitMQ已收到消息；如果这个时候处理消息失败，就会丢失该消息；
解决方案：处理消息成功后，手动回复确认消息。
```



高级特征：

消息的可靠投递：confirm  return

producer-broker-exchange-queue-consume

 消息从producer到exchange会返回一个confirmcallback

exchange到queue投递失败会返回一个returncallback



消息持久化机制

exchange持久化  声明队列必须设置持久化durable设置为 true。

queue持久化

消息的持久化	消息推送投递模式必须设置持久化，deliveryMode设置为2(持久)

**持久化的缺地就是降低了服务器的吞吐量，因为使用的是磁盘而非内存存储**



消费端限流

当消息的投递速度远快于消费速度时，随着时间累积就会出现消息积压，mq本身具备一定缓冲能力，但这个能力是有限制的，超过限制会发生崩溃

rabbitmq可有对内存和磁盘使用设置阈值，当达到阈值后，生产者会被阻塞，直到对应指标恢复正常，可以防止超大流量和消息挤压导致broker压垮。



TTL机制

首先queue自身的设置，可以保证队列中的所有的消息都有一个相同的过期时间

然后对消息也可以单独设着，每条消息的过期时间可以不同

两个都设置的情况下，会选择一个最小值来当做过期时间

超过过期时间的消息么就会变成**死信**，消费者无法在收到该消息，但是死信也可以被取出



死信队列

定义业务队列时，可以考虑定制一个死信交换机，并绑定到一个死信队列，当消息变成死信时，该消息就会发送到死信队列上，方便我们查看消息成为死信的原因。

消息变成死信的情况：

1 消息被拒绝  2  消息过期  3 队列达到最大长度

死信队列可以在处理异常情况下，分析改善优化系统



延迟队列

消息发送后不想立即被消费，需要等待一段时间后才触发消费。存放消息在延时交换机。