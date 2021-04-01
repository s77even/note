### Channel

CSP并发模型 通过通讯实现内存共享

带缓冲区和无缓冲区channel   同步和异步的区别  make创建

```go
type hchan struct { 
	qcount uint // total data in the queue 
	dataqsiz uint // size of the circular queue 
	buf unsafe.Pointer // points to an array of dataqsiz elements 
	elemsize uint16 
	closed uint32 
	elemtype *_type // element type 
	sendx uint // send index 
	recvx uint // receive index 
	recvq waitq // list of recv waiters 
	sendq waitq // list of send waiters
    lock mutex
}
```

无缓冲通道：同步  关键是找到匹配的接受和发送双方 找到则直接拷贝数据 找不到就将自身打包放入等待队列，由另一方复制数据并唤醒 type g struct {param} 唤醒参数

带缓冲区通道：

单向通道  一般不直接make 单向通道 而通过双向通道类型转换得到 



close() 可以显示关闭channel  一般在发送端调用 

向已关闭的channel写入数据会引发panic

读取已关闭的channel中的数据 会读取到缓存数据或channel数据类型零值

没有非常的必要必须显示关闭chan

理解：关闭channel 向channel中发送了一个关闭信号 信号前的数据能够正常读取 读取到关闭信号则表示通道被关闭 继续读取只能读取到零值 可用ok-idiom模式判断通道是否被关闭



channel底层是一个循环数组，