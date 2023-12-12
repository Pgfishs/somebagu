 # 数组和切片有什么区别

数组定长，大小不可改变，长度是数组类型的一部分；切片可以动态扩容，类型与长度无关。Slice底层数据是数组，是对数组的封装

数组是一片连续的内存，slice是结构体，包含三个字段：长度、容量、底层数组

**一个底层数组可以被多个slice指向，对slice操作有可能影响别的slice**

# slice扩容机制（append）

1.18之前

当原 slice 容量小于 1024 的时候，新 slice 容量变成原来的 2 倍；slice 容量超过 1024，新 slice 容量变成原来的1.25倍。

在1.18版本更新之后，slice的扩容策略变为了：

当原slice容量(oldcap)小于256的时候，新slice(newcap)容量为原来的2倍；原slice容量超过256，新slice容量newcap = oldcap+(oldcap+3*256)/4。需要内存对齐，进行内存对齐之后，新 slice 的容量是要 大于等于 按照前半部分生成的newcap。

append返回的是一个新slice，对传入slice不影响

# 向空slice添加元素会发生什么

其实 nil slice 或者 empty slice 都是可以通过调用 append 函数来获得底层数组的扩容。最终都是调用 mallocgc 来向 Go 的内存管理器申请到一块内存，然后再赋给原来的nil slice 或 empty slice，然后摇身一变，成为“真正”的 slice 了。

# slice作为参数传参

go都是值传递，直接传进去不会影响原slice，只有将slice指针传进函数进行操作或者在函数中直接对原slice底层数据进行操作，才会影响原slice中的内容

# map原理

用哈希函数将key分配到不同的bucket里，使用**链表法**和**开放地址法**碰撞（将不同key哈希进同一个bucket）。**链表法**将bucket实现成一个链表，落入同一个bucket中的ke也都会插入这个链表；**开放地址法**是碰撞发生后，通过一定规律在数组后面的“空位”放置新key

# **Map底层如何实现**

map内存模型是hmap，buckets是一个指针，指向结构体bmap（桶）。桶里最多装8个key，这8个key是经过哈希计算后，哈希结果是“一类的”，在桶内又会根据key计算出的hash值的高8位决定key落入桶中的什么位置

当map的key和value都不是指针且size都小于128字节时，bmap会被标记为不含指针，避免gc扫描整个hmap。bmap中k-v都是连续存放的（k-k-k....v-v-v...）省略padding字段省内存。每个buckets最多只有8个kv对，当第9个kv对落入当前bucket时，再构建一个bucket通过overflow指针连接起来

# **slice和map分别作函数参数时有什么区别**

map作为参数传参时，在函数参数内部对map操作会影响map自身，slice不会。*hmap是指针，slice是结构体，go中都是值传递，*hmap被copy到本地后仍指向同一个map，slice被copy后成为一个新的slice

# **创建map哈希函数选择**

程序启动时会检测cpu是否支持aes，支持则使用aes hash否则使用memhash，在函数alginit()中实现。alg字段中的typeAlg包含hash和equal两个函数，hash计算类型的哈希值，equal计算两个类型是否哈希相等

# **Key定位过程**

key经过哈希计算后得到64bit位的哈希值，通过最后B个bit位计算落入什么桶，（如果B=5那么桶数量（buckets长度）为2），再用高8位计算key在buckets中的位置，最开始桶内还没有key，新加入key找到并插入第一个空位

当两个不同的key落在同一个桶内，发生哈希冲突时，会使用链表法：在buckets中，从前往后找到第一个空位，这样在查找某个key时，先找到对应桶，再遍历bucket中的key。如果所有的bucket中没找到且overflow不为空，还要去overflow bucket中寻找，直到找到或者所有key槽位都找遍了，包括所有overflow bucket

# 如何实现map两种get操作

map有两种语法，带comma和不带comma，查询的key不在map中时，带comma会返回一个bool提示key是否在map中，而不带comma则会返回对应的0值。编译器直接做分析，给到底层两个不同函数

# map遍历过程

遍历所有bucket和overflow bucket，挨个遍历bucket中的cell，从有key的cell中取出k-v

如果遍历发生在扩容过程中，就会涉及遍历新老bucket；扩容不是一个原子操作，每次最多搬运两个bucket。老bucket会分裂到两个新bucket中，遍历操作会按照新bucket序号顺序进行，碰到老bucket未搬迁情况时，要在老bucket中找到将来要搬迁到新bucket中的key

# 赋值过程

调用mapassign：对key计算哈希，根据哈希计算位置赋值，外层遍历buckets和overflow bucket，内层遍历cell。函数会检查map标志位flags，如果为1说明有其他协程在“写”，导致程序panic，所以**map是协程不安全的**。

两个指针inserti和insertk，inserti指向key的hash所在tophash位置，insertk指向cell。Inserti和insertk分别指向第一个找到的空闲的cell，如果之后map中没有找到key，就插入新key；如果8个key位都满了，在跳出循环后发现inserti和insertk都是空，就在bucket后挂上overflow bucket。安置key之前还要判断是否需要扩容，再重新走一遍查找过程，然后更新map的count值

# 删除过程

mapdelete函数首先检查h.flags标志，如果为1说明其他协程正在操作

两层循环找到key的位置，对value清零，最后将count值-1，将对应的tophash置换成Empty

# 扩容过程

装载因子loadfactor := count/2^b；count标识元素个数，2^B表示bucket数量

满足两个条件，就会触发bucket扩容

1、装载因子超过阈值，源码里定义为6.5

2、Overflow的bucket过多：B小于15，bucket总数小于2^15时，overflow的bucket数量超过2^B；B大于等于15，bucket总数大于2^15时，overflow的bucket数量超过2^15

条件1装载因子超过6.5表明很多的bucket要装满了

条件2对条件1的补充，map中元素少但bucket多，导致搜索修改效率低

# 为什么key是无序的  
key在经过扩容、搬迁后位置发生巨大变化，遍历bucket返回的key就不会按顺序。Go在遍历map时，从随机的bucket的随机的cell开始遍历

# float可以作为key吗

除了slice、map、function都可以作为key，具体包括：布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组，共同特征是支持 == 和 ！=。任何类型都可以作为value，包括map

NaN != NaN hash[NaN]!=hash[NaN]

# map可以边遍历边删除吗

map不是一个线程安全的数据结构，多个协程同时读写一个map是未定义行为，被检测到直接panic。如果一个协程同时读写，理论可以，但是遍历结果不同了，有可能遍历包含被删除的key，也有可能不包含，取决于删除key的时间是在遍历到key所在的bucket之前或后

通过读写锁sync.RWMutex解决；sync.Map是线程安全的map可以使用

# 可以对map的元素取地址吗

不行，通过hack方式比如unsafe.pointer取到k/v地址也会因为扩容导致地址变化

# 如何比较两个map相等

深度相同的条件：

- 都为nil
- 非空、长度相等，指向同一个map实体对象
- 相应的key指向value“深度”相同

使用map1 == map2是错误的，只能比较map是否为nil，因此只能遍历map的每个元素，比较元素是否都是深度相同

# map是线程安全的吗

不是。在对map操作过程中都会检测写标志位，一旦发现标志位为1直接panic，等标志位复位之后再进行操作

# go语言和鸭子类型的关系

Go语言通过接口实现了ducktype

# 值接收者和指针接收者的区别

不管方法接收者是什么类型，该类型的值和指针都可以调用，不必严格符合接受者的类型

实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法

使用值接收者还是指针接收者，取决于该类型的**本质**。

如果类型是go内置的原始类型，比如字符串、整型等就定义值接收者；内置的引用类型，slice、map等，声明的时候实际是创建了一个header，对于他们也是直接定义值接受者类型的方法，调用函数时是直接copy了这些类型的的header

如果类型具备非原始的本质，不能被安全的复制，这种类型总是应该被共享，就定义指针接收者，比如go源码中的文件结构体File Struct

# iface和eface的区别

iface和eface都是go中描述接口的底层结构体，区别在于iface描述的接口包含方法，而eface是不包含任何方法的空接口interface{}

Iface内部维护两个指针，tab指向一个itab，表示指针类型以及赋给这个接口的实体类型，data指向接口具体的值，一般而言是一个指向堆内存的指针

Go的各种数据类型都是基于_type字段增加一些额外的字段进行管理

# 接口的动态类型和动态值

iface的data指向具体数据，被称为动态类型和动态值，接口值包括动态类型和动态值

当动态类型和动态值都为nil时，接口值==nil

# 类型转换和断言的区别

类型断言是对接口变量进行的操作, v := x.(类型）

类型转换，转换前后的两个类型要相互兼容 v := 类型(x)
# Go接口和C++接口异同
C++接口是使用抽象类实现的
C++定义接口是侵入式，Go是非侵入式，不需要显式声明，只需要实现接口定义的函数，编译器就会自动识别
C++通过虚函数表实现基类调用派生类的函数，Go通过itab中的fun字段实现接口变量调用实体类型的函数。
C++虚函数表是在编译期生成的，Go的fun是在运行期间动态生成的
# 什么是CSP
不要通过共享内存来通信，而要通过通信来实现内存共享
这就是Go的并发哲学，依赖CSP模型，基于channel实现。Go的并发模型用goroutine和channel替代，goroutine和线程类似，channel和mutex类似
Go的并发原则，尽量使用channel，把goroutine当免费资源，随便用
## 编译器自动检测类型是否实现接口
`var _ io.Writer = (*myWriter)(nil)
编译器会由此检查 myWriter 类型是否实现了 io.Writer 接口。
# Channel的底层数据结构
```
type hchan struct {
	// chan 里元素数量
	qcount   uint
	// chan 底层循环数组的长度
	dataqsiz uint
	// 指向底层循环数组的指针
	// 只针对有缓冲的 channel
	buf      unsafe.Pointer
	// chan 中元素大小
	elemsize uint16
	// chan 是否被关闭的标志
	closed   uint32
	// chan 中元素类型
	elemtype *_type // element type
	// 已发送元素在循环数组中的索引
	sendx    uint   // send index
	// 已接收元素在循环数组中的索引
	recvx    uint   // receive index
	// 等待接收的 goroutine 队列
	recvq    waitq  // list of recv waiters
	// 等待发送的 goroutine 队列
	sendq    waitq  // list of send waiters

	// 保护 hchan 中所有字段
	lock mutex
}
```
buf指向底层循环数组，只有缓冲型channel才有；sendx，recvx指向底层循环数组，表示当前可以发送和接受的元素位置引索值；sendq，recvq分别表示被阻塞的goroutine，这些goroutine由于尝试读取channel或向channel发送数据被阻塞；waitq是sudog的一个双向链表，sudog是对goroutine的一个分装；lock用来保证每个读或写channel的操作都是原子的
# 向channel发送数据的过程是什么
