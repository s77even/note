### golang slice 扩容

slice 底层由一个 ptr和len cap构成，ptr指向底层数组



操作系统预分配的内存规格  byte 8 16 32 64 80 96 112...

先求出当前切片容量x，求出append追加后的容量 y

判断 x*2 和y 的关系

**1**

 2x < y 使用y， 根据y的个数乘上切片中存储数据类型的大小，求出需要分配的内存，然后再预分配的内存规格选择一个大于的距离最近的规格，然后除以数据类型大小求出个数，即扩容后的个数

[]int  x=4，y=9  ,4* 2<9,  9 *8=72, 取内存规格中的80， 80 /8 =10， 求出扩容后容量为10



**2**

2x>y ,判断是否个数x大于1024， 如果不大于1024， 取2x * 数据类型的大小，寻找与分配内存规格，再求出个数

[]int  x =4, y=7, 4* 2 = 8 >7, 7* 8=56,取内存规格为64， 64/8=8，扩容后容量为8

如果大于1024， 先扩容 1/4，匹配内存规格，求出扩容后规格。





### Map

底层采用的是hash，使用链表解决的哈希冲突

```go
type hmap struct{
    count int //元素个数，len的返回值
    flags uint2 // 
    B uint8  // buckets数组长度的对数  2^B为buckets个数
    noverflow uint8 // overflow的bucket的近似数
    hash0 uint32   //hash函数
    buckets  unsafe.Pointer //指向buckets数组 如果元素个数为0 即为nil
    oldbuckets unsafe.Pointer // 扩容时候会是old 的两倍
    nevacuate uinptr // 扩容进度 小于此地址的buckerts完成了迁移
    extra *mapextra  //扩充项
    
}
```

buckets指向一个结构体bmap，就是我们常说的桶，桶里面最多装8个key，hash值的高8wei来决定key落入桶的哪个位置

bmap中，key和value是分开存放的，key0-8存放在一起，value0-8存在一起，减少了padding，节省内存空间



hash函数：程序会检测cpu是否支持aes哈希，否则就使用memhash

key的定位：

key 经过哈希计算后得到哈希值，共 64 个 bit 位，计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。

最开始桶内还 没有 key，新加入的 key 会找到第一个空位，放入

buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解 决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到 对应的桶，再去遍历 bucket 中的 key。如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直 到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket



**map 的扩容**

在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个 条件，就会触发扩容： 

1 装载因子loadFactor := count / (2^B)  超过阈值，源码里定义的阈值是 6.5。

2 overflow 的 bucket 数量过多：当 B 小于 15，overflow 的 bucket 数量超过 2^B；当 B >= 15，如果 overflow 的 bucket 数量超过 2^15。 



map的扩容采取了渐进式的方式，原有的key不会一次性搬迁完，nevacuate 标识的是当前的进度

针对触发扩容条件1，将B增加1，这样容量就翻倍，然后对扩容后内存进行预分配，使用渐进式扩容的方式，





map并非是并发安全的，sync map是线程安全的，通过读写分离，降低锁时间来提高效率，适合读多写少的场景。使用read dirty两个map来进行读写分离。写入性能差

store 写入  load 读取 range遍历  delete 删除  loadorstore



read 是atomic类型 可以并发安全的读取，不能新增

dirty 是非线程安全的原始map 包含新写入的key和reaad中未被删除的key

两个map其实底层是共享数据的，对value 的更改同时可见。

删除操作：延迟删除，先查看read中，如果read中有置为nil 在dirty提升为read时删除，只在dirty中就直接删除

读取操作：先去read中读取，如果没有就采用枷锁去dirty中读取，并检测miss变量，m大于len（dirty）会提升为read

doucheck：j先检查read 没有命中 就上锁 然后再检查。因为有可能在这期间dirty提升为read，key存在了

写入操作：先看read中是否存在key，然后存在就直接更新，不存在就上锁，然后doeblecheck，如果dirty中存在就更新，dirty中不存在，就先复制到ditry中，在写入。