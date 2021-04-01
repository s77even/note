### 微服务

##### 互联网架构的演进之路

单体架构 ->垂直架构 -> SOA架构 -> 微服务架构

单体架构：所有功能集成在一个项目工程中，架构简单，开发成本低，周期短，小项目首选。系统性能受限，代码耦合，开发维护困难。

垂直架构：将一个项目垂直拆分为一个个单体结构项目，可以针对不同模块进行优化，方便水平扩展，系统相互独立。

SOA架构：将重复共用的功能抽取为组件，以服务的方式给各系统提供服务，ESB总线作为项目与服务之间通信的桥梁

微服务架构：将系统服务层独立抽取为一个一个微服务，遵循单一原则，服务之间采取restful轻量协议传输。



微服务优点：拆分粒度更细，有利于资源重复利用，提高开发效率                      （松耦合）

​						更加精准针对每个服务制定优化，提高系统可维护性					（独立发布）

​						轻量协议通信相比ESB更加轻量级													（快读迭代）

​						产品迭代周期更短																			  （故障隔离）

缺点:微服务过多，服务治理成本高

​		分布式系统开发技术成本高



微服务架构核心点：

API Gateway

进程间通信

服务注册发现

事件驱动的数据管理

微服务部署策略

微服务化改造

**微服务的特点**

单一职责，自治，简化部署，灵活拓展，灵活组合，高可靠，容错



**客户端如何访问这些微服务**——**网关API Gateway**

提供统一服务入口，让微服务对前台透明化

聚合后台的服务 节省流量 提高性能

提供安全，过滤，流控等管理功能





**微服务与微服务之间的通信**  （同步（rpc rest）异步（异步消息调用））

**RPC 远程过程调用协议**

微服务之间的不同服务之间的通信调用， 调用远程服务器上的程序的方法

RPC架构：

客户端：服务调用发起发

服务端：远程计算机机器杭运行的程序，其中包含客户端要调用和访问的方法和结构

客户端存根stub：客户端一方负责存放服务端地址，端口消息，将自己的请求参数打包成网络消息，发送到服务端，并负责接收和解析服务端返回的数据包，

服务器存根stub：接收客户端发送来的数据包，解析数据包，调用具体的服务方法，将调用结果打包发送给客户端乙。

Rpc设计相关技术：

动态代理技术   动态代理技术自动生成stbu程序

序列化和反序列化   protobuf json xml



**protobuf 协议语法**

格式：定义传输数据的格式，并命名为以 .proto为扩展名的消息定位文件+



足够简单，

序列化后体积跟小，大小只有xml的1/20，json 的1/10

解析速度更快，比xml快20倍，支持多语言环境





**message 定义一个消息**

```protobuf
message Order{
// 1 2 3代表字段的顺序 最小从1开始 最大是536870977（2^99 -1） [19000-19999]为预留 不能使用
	 int32 OrderID =1;
     int64 num =2;
     string timeStamp =3;
}
```

**protoc**

protoc  ./xxx.proto  --go_out=. /

**net/rpc包 使用**

c/s 模型

客户端和服务端要定义好要传递和接受的结构体模型定义。

服务端：

服务定义及暴露   定义类型和相关类型的方法集

```go
func (t *T) MethodName(request T1,*response T2)error
```

**对外暴露的服务方法要符合定义标准：**

**只能有两个参数，只能是输出类型或内建类型**

**第二个参数response必须是指针类型**

**方法的返回值之能事error**

**方法的类型是可输出的**

**方法本身也是可输出的**



定义完成 后，使用rpc包的注册方法注册服务对象，rpc.register(&recever)传入对象

也可使用 名称注册 rpc.RegisterName("MathUtil",mu) //使用名称注册



将服务注册到http协议，方便使用者通过http协议调用

rpc.HandleHttp()

监听服务通信端口，net.Listen("tcp","ip:port")

接受通信端口发来的连接请求   http.serve(listen,nil)



客户端：

对服务端通信端口发起rpc连接请求：rpc.DialHTTP()

client, err := rpc.DialHTTP("tcp", "localhost:8081")

准备好请求方法的传入参数，注意第二参数为指针类型

同步调用或异步调用 发送rpc请求  返回结果被保存在第二参数中

同步调用：client.call("调用类型.调用方法"，第一参数，第二参数指针)

err = client.Call("MathUtil.CalculateCircleArea", req, &replay)

异步调用：client.go("调用类型.调用方法"，第一参数，第二参数指针，nil)最后参数为一个通道，传入nil会自动创建一个，返回值为rpc.call     调用其done方法从通道中获取消息来确认调用完成。

synccall := client.Go("MathUtil.CalculateCircleArea", req, replaysync, nil)

  <-synccall.Done



**如果需要传入多个参数，就定义一个结构体，将多个数据保存在结构体中进行传送**



Rpc和Protobuf结合使用：**grpc**

protoc 编译生成**兼容 grpc**的pb.go文件  

protoc ./message.proto --go_out=**plugins=grpc=**:./

在proto文件中 声明需要暴露的服务 如 ：

```protobuf
service OrderService{
  rpc GetOrderInfo(OrderRequst)returns (OrderInfo);
}
```

这样生成的。pb.go文件中 会自动针对服务端和客户端生成对应的调用方法， 在客户端就可以直接调用使用 不需要传入函数名去和参数去间接调用（实际上只是造成了一种直接调用的假象）



server端：

1 定义服务结构体

2 定义结构体方法集：serveice服务方法  要与pb中生成的方法 声明相同

如

```go
//GetOrderInfo(context.Context, *OrderRequst) (*OrderInfo, error)  自动生成
func (os *OrderService)GetOrderInfo(ctx context.Context,requst *message.OrderRequst)(*message.OrderInfo, error)
```

3 grpc 创建服务端  注册服务端服务 传入grpc服务端和服务结构体

```go
server := grpc.NewServer()
message.RegisterOrderServiceServer(server,new(OrderService))
```

4 监听端口 等待客户端连接

```go
listen, err := net.Listen("tcp",":8090")
if err != nil {
   panic(err.Error())
}
server.Serve(listen)
```



客户端：

1 建立与服务端的连接

```go
conn,err := grpc.Dial("localhost:8090",grpc.WithInsecure()) 
```

2 创建一个调用服务端的客户端 

```go
orderserviceclient := message.NewOrderServiceClient(conn)
```

3 定义要调用服务的传入参数  并直接调用方法 

```go
orderinfo ,err := orderserviceclient.GetOrderInfo(context.Background(),orderrequest)
```



grpc  restiful API

都是用了http作为底层的传输协议，严格的说，grpc是用的http2.0，retiful不一定

grpc通过protobuf来定义接口，有严格的接口约束条件，rest一般 采用xml和json

protobuf可以将数据序列化为二进制编码，减少了需要传输的数据量，提高了大幅的性能

grpc支持流式通信，http2.0本身也可使用streaming模式，restiful似乎很少这么用

通常的流式数据一般都会使用专门的协议传输，HLS等



http2.0  http1.0 htpp1.1

http2.0是1.1发布后的首个更新，主要基于spdy协议，speedy协议 优化了http协议的性能，通过压缩多路复用和优先级等技术，缩短网页的加载时间并提高安全性。

http1.x的一部分缺点

一次只允许在一个tcp连接上发起请求，http1.1虽然可以发起多个请求，但是也会存在阻塞问题，需要多次请求，一般还是建立多连接。

单向请求，

请求报文和响应报文首部信息冗余量大

数据未压缩，导致传输的数据大



二进制分帧层

帧是2.0通信的最小单位，2.0加强性能的核心就是二进制传输，在应用层和传输层加入一个分帧层，将所有传输的消息分为更小的消息和帧，采用二进制格式编码，生成headers和data帧

首部压缩

http1.1不支持首部压缩，2使用了专门设计的hpack算法，每次通信都会携带首部信息用于描述资源属性。

使用hpack压缩格式对传输的header进行编码，减少了header的大小，并在两端维护了一个索引表，用于记录出现过的header，后面传输只需要传输键值，就可以在表中完成查找。

多路复用

基于二进制分帧层，可以在共享tcp连接的基础上同时发送请求和响应，http消息被分解为独立的帧，在另一端根据流标识符和首部重新组装，避免了旧版本http头部阻塞的问题。

请求优先级

把消息分成很多帧后，优化这些帧的交错和传输顺序，可以确保哪些被其他相应数据所依赖的关键资源被优先传输传输

服务器推送

服务器可以对一个客户请求发送多个响应，推送额外的资源给客户端。加快了页面的响应速度



