### mutex

```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}
```

state表示锁当前的状态，0值为未上锁的锁

sema用作信号量，通过PV操作在等待队列中阻塞唤醒goroutine，等待锁的goroutine会被挂载到等待队列，并且陷入睡眠不被调度，unlock时才被唤醒。



mutex锁有两种状态：正常状态和饥饿状态

正常模式下 所有等待锁的goroutine会按照FIFO顺序等待，唤醒goroutine不会立即拥有锁，而是会和新请求锁的goroutine竞争，新请求的goroutine具有优势；他正在CPU上运行，所以被唤醒的go很大可能会竞争失败，在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面，如果一个等待的goroutine超过1ms没有获得锁，就会把锁转变为饥饿模式



饥饿模式下，锁的所有权将从unlock的goroutine直接交给等待队列的第一个，新来的goroutine不会尝试获得锁，会放在等待队列的尾部。

如果等待的goroutine 获得了锁，满足任一条件都可以将锁转变为正常状态：

1  他是等待队列中最后一个

2   他等待的时间小于1ms



lock

如果锁处于unlock的正常模式，通过CAS获取锁，fast path，获取失败就通过slow path

slow path下获取锁分两种情况：饥饿模式和正常模式

正常模式：自旋  获取

饥饿模式：没有自旋操作  为了保证公平 防止尾部等待  



unlock

使用AddInt32函数快速解锁，fast path，如果解锁不成功，就会通过slow path开始慢解锁

正常模式：不存在等待者，互斥锁相关状态位不为0 当前方法直接放回，不需要唤醒别的等待者

​					存在等待者，唤醒等待者，移交锁的所有权

饥饿模式：直接将锁交给下一个正在尝试获取的等待者。

