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

# Map底层如何实现

map内存模型是hmap，buckets是一个指针，指向结构体bmap（桶）。桶里最多装8个key，这8个key是经过哈希计算后，哈希结果是“一类的”，在桶内又会根据key计算出的hash值的高8位决定key落入桶中的什么位置

当map的key和value都不是指针且size都小于128字节时，bmap会被标记为不含指针，避免gc扫描整个hmap。bmap中k-v都是连续存放的（k-k-k....v-v-v...）省略padding字段省内存。每个buckets最多只有8个kv对，当第9个kv对落入当前bucket时，再构建一个bucket通过overflow指针连接起来

# slice和map分别作函数参数时有什么区别

map作为参数传参时，在函数参数内部对map操作会影响map自身，slice不会。*hmap是指针，slice是结构体，go中都是值传递，*hmap被copy到本地后仍指向同一个map，slice被copy后成为一个新的slice

# 创建map哈希函数选择

程序启动时会检测cpu是否支持aes，支持则使用aes hash否则使用memhash，在函数alginit()中实现。alg字段中的typeAlg包含hash和equal两个函数，hash计算类型的哈希值，equal计算两个类型是否哈希相等

# Key定位过程

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
# sync.map如何实现并发安全
内部实现了**分片锁**，将整个映射分成若干个小的分片，每个分片都有独立的锁，每个分片负责一部分映射的kv对
读操作是并发安全的，写操作时获取对应分片的锁，不会影响其他分片
sync.map内部是一个数组，每个数组元素都是一个分片，写操作时，根据键的哈希值决定使用哪个分片，通过哈希值和分片的数量取模，得到对应的分片引索，然后获取分片的锁进行写操作
分片锁可以提高并发性能，且不同分片有自己的锁，不会造成阻塞，但是进行更复杂的鬓发操作时需要进行其他的操作

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
# Channel的读写
### 写
如果channel是nil，不能阻塞直接返回false；对于不阻塞的send，检测失败场景，channel是非缓冲型且等待接收队列里没有goroutine，或是缓冲型但已经装满了元素；
- 如果检测到channel已关闭，直接panic
- 如果能从等待接收队列出队一个sudog（代表一个goroutine），说明此时channel是空的，没有元素，才会有等待接收者。这时会调用send函数将元素从发送者栈拷贝到接收者栈，由`sendDirect`完成
`sendDirect`涉及到一个goroutine直接写另一个栈的操作，一般不同的栈都是独有的，也违反了GC一些假设；写的过程中通过写屏障保证正确完成操作，好处是减少了一次内存copy：不用先拷贝到channel的buf，直接发到接收者；然后解锁、唤醒接收者，等待调度器
判断缓冲区是否塞满，如果满了就把sender关起来（goroutine阻塞住）；如果真的被阻塞，先构造一个sudog入队，然后调用goparkunlock将当前goroutine挂起，等待唤醒
待发送的元素地址存储在sudog中，也就是当前的goroutine中
### 读
接受操作有两种写法，带ok反应channel是否关闭；不带ok，当接收到相应类型的零值是无法知道是真实接收者发来的值还是channel关闭后返回的默认零值
`chanrecv1`处理不带ok的情况，`chanrecv2`通过返回received反应channel是否被关闭，接收值放到参数elem指向的地址，最后转向了chanrecv函数
等待队列中有goroutine存在，说明buf是满的
- 非缓冲channel，直接将内存拷贝从sender goroutine到receiver goroutine
- 缓冲channel但是buf满了，接收到循环数组头部的元素，并将发送者元素放到循环数组尾部
最后调出sudog中的goroutine，调用goready改成runnable，待发送者被唤醒，等待调度器调度
- 如果buf还有数据，正常的将buf里接受游标处的数据拷贝到接受数据的地址
- 如果block值是false，直接返回

# 关闭一个Channel的过程
检测channel是否为空/关闭，然后将channel上锁，把channel的sender和receiver都连成一个sudog链表再解锁，最后再把sudog全部唤醒
recvq和sendq中分别保存了阻塞发送者和接收者，关闭channel后，等待接收者会收到一个相应类型的零值，等待发送者则会直接panic
唤醒后，sender检测到channel已经关闭，panic；receiver进行扫尾工作后返回。selected返回true，received根据channel是否关闭，返回不同的值，channel关闭received为false，否则为true
# 从关闭的channel仍能读出数据吗
从有缓冲的channel读数据，当channel关闭仍能读出有效值，当返回的ok为false时，读出数据才是无效的
# 操作channel的结果
![[Pasted image 20231222164504.png]]
# 优雅关闭channel
- 不改变channel自身状态情况下无法得知channel是否关闭
- 关闭一个close channel会导致panic，关闭channel的一方不知道channel是否关闭就去贸然关闭channel很危险
- 向close channel发送数据会导致panic，同上危险
两个不优雅关闭channel方法
- 使用defer-recover机制，即使发生了panic也可以兜底
- 使用sync.once保证只关闭一次
对于只有一个sender的channel，直接从sender端关闭
对于有多个sender，关闭channel解决方案是通过增加一个传递关闭信号的channel，recevier通过信号channel下达关闭数据的channel指令，sender监听到关闭信号后停止发送数据
**或**通过增加中间人，M个receiver都向他发送关闭dataCH的请求，收到第一个请求后就直接关闭dataCH的指令
# channel发送和接收元素的本质
值的拷贝，无论是从sender goroutine到chan buf，从chan buf到receiver goroutine或直接从sender到receiver
# channel引起资源泄漏的情况
channel会引起goroutine泄露
goroutine操作channel后，处于发送或接受阻塞状态，而channel处于满或空状态， 一直得不到改变。同时垃圾回收器也不会回收此类资源，导致goroutine处于等待队列中
程序运行中，对一个channel如果没有任何goroutine引用，gc会将其回收不会引起内存泄漏
# channel的happened-before
关于channel的send、send-finished、receive、receive-finished的happened before关系
- 第n个`send`一定happened before第n个`receive finished`，无论是缓冲型还是非缓冲型
- 容量为m的缓冲型channel，第n个`receive`一定happened before第n+m个`send finished`
- 对于非缓冲型的channel，第n个`receive`一定happened before第n个`send finished`
- channel close一定happened before `receiver`得到通知
# channel应用
### 停止信号
[如何优雅关闭channel相关](./#优雅关闭channel)
### 任务定时
与timer结合：实现超时控制、实现定期执行某个任务
### 解耦生产方和消费方
服务启动时启动n个worker，作为协程工作池，这些携程工作在一个for{}无限循环中，从某个channel消费工作任务并执行
### 控制并发数
需要定时执行几百个任务，但是并发数又不能太高，可以通过channel控制并发数
# Context是什么
goroutine上下文，包含goroutine的运行状态、环境、现场等信息，用来在goroutine之间传递上下文信息，包括取消信号、超市信号、截止时间等
# Context作用
在一组goroutine之间传递共享的值、取消信号、deadline等，解决goroutine之间退出通知、元数据传递的功能
### 传递共享的数据
传递context类似于ThreadLocal变量
### 取消goroutine
### 防止goroutine泄露
# Context.Value的查找过程
通过比较节点key的值，顺着链路一直往上找（context结构类似反过来的链表），如果是查找的key值直接返回value，如果最终找到根节点则返回nil；父节点无法获取子节点的值，子节点却可以获取父节点的值
`WithValue`创建context节点的过程实际是创建链表的过程，两个节点的key值是可以相等的，但是是不同的两个节点，向上查找到最后一个故在的context节点，整体上`WithValue`创建的是一个低效率的链表，同时尽量不要用context传值
# context如何被取消
`cancel()`:c.done；递归的取消所有子节点；从父结点删除自己。达到的效果是通过关闭channel，将取消型号传递给了所有子节点，goroutine收到取消信号的方式是select中的`读 c.done`被选中
调用`WithCancel()`方法的时候，也就是创建一个可取消的context节点时，返回的cancelFunc函数会传入true
# 什么是反射
在程序运行时能**观察**并修改自己的行为
Go提供了反射机制在运行时更新变量和检查他们的值、调用他们的方法，但是在编译时不知道这些变量的具体类型
# 什么时候需要使用反射
1. 不能明确接口调用哪个函数，需要根据传入的参数在运行时决定
2. 不能明确传入函数的参数类型，需要在运行时处理任意对象
#### 不推荐使用反射的原因
- 难以阅读
- 编译器无法发现反射代码中的类型错误，可能要运行很久直接panic
- 对性能影响大
# 反射的实现
`reflect.Type`提供关于类型相关的信息，和_type关系紧密
`reflect.Value`结合_type和data两者，可以获取甚至改变类型的值
`reflect.Type`通过Typeof()将实参转化为Type接口，通过其中方法获取实参信息
`reflect.Value`通过Valueof()将实参的信息组装成Value结构体返回，包含结构体指针、真实数据地址、标志位，通过Value的方法操作Value字段ptr指向的实际数据
### 反射三大定律
- 反射是一种检测储存在interface中类型和值的机制，可以通过`Typeof`&`Valueof`得到
- 将`Valueof`的返回值通过`Interface()`函数反向转化为`interface`变量
- 只能操作一个可被设置的反射变量；如果要操作原变量，反射变量Value必须要hold住原变量地址
# 反射的应用
IDE自动补全、对象序列化、fmt相关函数的实现，ORM对象关系映射等
# 比较两个对象完全相同
DeepEqual比较是否深度相同，一般情况下递归使用 == 比较，特殊情况下比如func不可比较只有都是nil时才“深度相同”
# Go指针和unsafe.pointer区别
**限制一**：不能进行指针运算，会编译错误`invaild operation`
**限制二**：不同类型指针不能相互转换
**限制三**：不同类型指针不能`== 或 !=`比较，只有类型相同或可以相互转化才可以进行比较，可以直接对nil进行比较
**限制四**：不同类型指针变量不能相互赋值
# 逃逸分析
分析指针动态范围叫逃逸分析，当一个指针的对象被多个方法或线程引用时，指针发生了逃逸
Go的逃逸分析是编译器指向静态代码分析后堆内存管理进行的优化和简化，决定一个变量时分配到堆还是栈
Go分析逃逸最基本原则是：一个函数反对一个变量的引用，那么它就会发生逃逸。Go中变量经过编译器证明返回后不会再被引用，才会被分配到栈上，其他情况下分配到堆上
# GoRoot和GoPath
GoRoot时Go安装路径，比较重要的有compile编译器，link连接器
GoPath提供一个可以寻找`.go`源码的路径，是一个工作空间的概念
# Go编译过程
编译过程：对源文件进行词法分析、语法分析、语义分析、优化、生成汇编文件。然后汇编器将汇编代码变为机器可执行的指令
# Go编译命令
### go build
编译指定packages里源码路径和依赖包，会忽略`*_test.go`文件
### go install
编译并安装指定代码包和依赖包
### go run
编译并允许代码文件
# goroutine和线程区别
#### 内存消耗
创建一个goroutine栈内存消耗为2KB，运行中会扩容。创建一个thread则需要消耗1MB栈内存，还需要"a guard page"区域用于和其他thread栈空间隔离
使用goroutine处理请求非常轻松，而是用线程处理请求则浪费资源，容易出现OOM
#### 创建和销毁
thread是内核级的，创建和销毁都有巨大消耗，而goroutine是由go runtime管理的，是用户级的
#### 切换
线程切换时需要保存各种寄存器，而goroutine只需要三个寄存器：Program Counter、Stack Pointer和BP
线程切换需要1000-1500ns，而goroutine只要200ns
# Go Scheduler
Go执行由GoProgram和Runtime构成，之间通过函数调用实现内存管理、channel通信等功能，用户程序进行的系统调用会被Runtime拦截帮助进行调度和垃圾回收
Runtime维护所有goroutine，通过scheduler进行调度，goroutine和threads是独立的，但goroutine要依赖threads才能执行
### Scheduler底层原理
三个基础结构体实现goroutine调度，G、M、P
g代表一个goroutine，包含：表示goroutine栈的一些字段，只是当前goroutine状态，指示当前运行到的指令地址，也就是PC值
m代表内核线程，包含正在运行的goroutine等字段
p代表虚拟的Processor，维护一个储运Runnable状态的g字段，m需要获得p才能运行g
还有一个核心结构体sched总揽全局
Runtime创建时会启动一些G，垃圾回收/执行调度/运行代码，同时会创建一些M开始G运行
内核线程在核心上调度，而G在M上调度
### 核心思想
- reuse threads
- 限制同时运行的线程数为N（CPU核心数）
- 线程私有的runqueues，并可以从其他线程的stealing goroutine运行，线程阻塞后可以将runqueues传递给其他线程
当一个线程阻塞时，将和他绑定P上的goroutines转移到其他线程
# Goroutine调度时机
有机会进行调度

|情形|说明|
|---|---|
|使用关键字go|go创建一个新的goroutine，scheduler会考虑调度|
|GC|进行GC的goroutine在M上运行，所以肯定会发生调度。scheduler还会做其他很多的调度，比如调度不涉及堆访问的goroutine运行。GC不管栈上内存，只回收堆上内存|
|系统调用|goroutine系统调用时，会阻塞M，所以他会被调走，同时创建一个新goroutine调度上来|
|内存同步访问|atomic、mutex、channe等操作阻塞goroutine，等条件满足后还会被调度上来重新运行|
# M:N模型
runtime负责goroutine生老病死，runtime会在程序启动时候，创建M个线程，之后创建的N的goroutine都会在这M个线程上执行，这就是M:N模型
同一时刻一个线程只能跑一个goroutine，发生阻塞时runtime会把当前goroutine调走，执行其他goroutine，榨干CPU
# 工作窃取
scheduler职责是将所有处于runnable的goroutine均匀分配到P上运行的M，当一个P发现自己的LRQ没有G时，会从其他P偷一些G来运行
scheduler使用M:N模型，任意时刻M个goroutine（G）要分配到N个内核线程（M），跑在最多位GOMAXPROCS的逻辑处理器上（P），每个M必须依附一个P，每个P同一时刻只能运行一个M，如果P上M阻塞了，就需要其他M来运行P中LRQ的goroutine
# GPM是什么
### G
取goroutine首字母，主要保存goroutine一些状态信息和CPU一些寄存器值，比如IP寄存器，以便轮到本goroutine执行时，CPU知道如何开始执行
	goroutine被调离CPU时，调度器负责把CPU寄存器值存在g的成员变量中，当goroutine被调度起来时，调度器负责把g成员变量恢复到CPU寄存器中
g结构体关联了两个比较简单的结构体，stack表示goroutine运行时的栈，gobuf保存了PC、SP等寄存器的值
### M
machine首字母，代表一个工作线程/系统线程，G需要调度到M上才能运行
结构体m保存了M的栈信息，当前M上运行的G信息，绑定的P信息。当M无事可做时，在休眠前会自旋的找工作：检查全局队列，查看network poller，试图执行gc，偷别人的任务
### P
processor，为M执行提供上下文，保存M执行G时一些资源，比如本地可运行G队列、memory cache等
一个M只有绑定P才能执行goroutine，M被阻塞时P会传递给其他M
刚开始运行初始化时，所有P都在_Pgcstop，随着P初始化，被置于_Pidle，当M需要运行时，P进入Prunning状态；当G需要进入系统调用时，P被设置为_Psycall，被系统监控抢夺会被修改为_Pidle；如果在程序运行中发生GC，P会被设置为_Pgcstop，在runtime.startTheWorld时调整为_Prunning
# Go scheduler初始化
源码中的结构体是`schedt`，保存调度器的状态信息、全局可运行队列G等，程序运行过程中，`schedt`只有一份实体，维护了调度器所有信息
proc.go和runtime2.go文件中，有重要的全局变量，程序初始化时，全局变量会被初始化为对应类型的零值
调整SP，将SP调整到地址是16倍数的位置，CPU中有一组SSE指令，出现的内存地址必须是16的倍数
初始化g0栈，g0栈为运行runtime代码提供一个环境
主线程绑定m0。因为m0是全局变量，而m0又要绑定到工作线程才能执行，runtime会启动多个工作线程，每个线程都会绑定一个m0，且代码中要保持一致，都用m0显示；TLS机制保证一个线程内部各个函数都能访问，其他线程不能访问的的变量
初始化m0，涉及到`schedinit`：设置最多工作线程数、初始化m0、初始化P个数（CPU核数），初始化所有P，将m0挂载到allm上（若创建新m会与m0相连）
初始化allp，设置procs，决定创建p的数量
# 主goroutine如何创建
启动一个goroutine时候，在go编译器作用下，最终会转化成`newproc`函数
`newproc`有两个参数，新创建goroutine需要执行的任务fn，代表一个函数func；还有fn参数的大小`siz`
**为什么要传fn参数大小**，因为goroutine和线程一样都有自己的栈，goroutine的栈比较小，newproc函数创建一个新的goroutine函数来执行fn函数，在新goroutine上执行指令就需要新goroutine的栈，通过`siz`决定拷贝数据大小，从老goroutine转移到新goroutine上
`newproc`第二个参数`funcval`是一个变长结构
[还是看网站吧](https://golang.design/go-questions/sched/main-goroutine/)
# GC
Go是追踪式GC，从根对象出发扫描完整个堆，判断并清除垃圾；使用三色标记法进行GC
- Go分配算法基于tcmalloc，几乎没有碎片问题，对对象进行整理不会带来实质性的提升
- Go的编译器会进行逃逸分析将新生对象存储在栈上，长期存在的对象才会被放在堆上，goroutine死亡后的栈会被直接回收，无需GC参与，分代假设没有带来收益
# 三色标记法
三色抽象规定了三种不同类型的对象：
- 白色对象（可能死亡）：未被回收器访问到的对象。在回收开始阶段，所有对象均为白色，当回收结束后，白色对象均不可达
- 灰色对象（波面）：已被回收器访问到的对象，但回收器需要对其中的一个或多个指针进行扫描，因为他们可能还指向白色对象
- 黑色对象（确定存活）：已被回收器访问到的对象，其中所有字段都已被扫描，黑色对象中任何一个指针都不可能直接指向白色对象
当垃圾回收开始时，只有白色对象。随着标记过程开始进行时，灰色对象开始出现（着色），这时候波面便开始扩大。当一个对象的所有子节点均完成扫描时，会被着色为黑色。当整个堆遍历完成时，只剩下黑色和白色对象，这时的黑色对象为可达对象，即存活；而白色对象为不可达对象，即死亡。这个过程可以视为以灰色对象为波面，将黑色对象和白色对象分离，使波面不断向前推进，直到所有可达的灰色对象都变为黑色对象为止的过程
# STW
`STW` 可以是 `Stop the World` 的缩写，也可以是 `Start the World` 的缩写。通常意义上指指代从 `Stop the World` 这一动作发生时到 `Start the World` 这一动作发生时这一段时间间隔，即万物静止。STW 在垃圾回收过程中为了保证实现的正确性、防止无止境的内存增长等问题而不可避免的需要停止赋值器进一步操作对象图的一段过程
# 如何观察 Go GC
方式1：`GODEBUG=gctrace=1`
方式2：`go tool trace`
方式3：`debug.ReadGCStats`
方式4：`runtime.ReadMemStats`
# Go中GC内存泄露的情况
有GC的语言中内存泄漏是指：本应很快被释放的内存附着在了长期存在存活的内存上，导致无法及时释放回收内存
- **形式1：预期能被快速释放的内存因被根对象引用而没有得到迅速释放**。当有一个全局对象时，可能不经意间将某个变量附着在其上，且忽略的将其进行释放，则该内存永远不会得到释放
- **形式2：goroutine 泄漏：** Goroutine 作为一种逻辑上理解的轻量级线程，需要维护执行用户代码的上下文信息。在运行过程中也需要消耗一定的内存来保存这类信息，而这些内存在目前版本的 Go 中是不会被释放的。因此，如果一个程序持续不断地产生新的 goroutine、且不结束已经创建的 goroutine 并复用这部分内存，就会造成内存泄漏的现象
# 什么是写屏障、混合写屏障，如何实现
可以证明，当以下两个条件同时满足时会破坏垃圾回收器的正确性：
- **条件 1**: 赋值器修改对象图，导致某一黑色对象引用白色对象；
- **条件 2**: 从灰色对象出发，到达白色对象的、未经访问过的路径被赋值器破坏。
三色不变性所定义的波面根据这两个条件进行削弱：
- 当满足原有的三色不变性定义（或上面的两个条件都不满足时）的情况称为**强三色不变性（strong tricolor invariant）**
- 当赋值器令黑色对象引用白色对象时（满足条件 1 时）的情况称为**弱三色不变性（weak tricolor invariant）**
当赋值器进一步破坏灰色对象到达白色对象的路径时，即打破弱三色不变性， 也就破坏了回收器的正确性；或者说，在破坏强弱三色不变性时必须引入额外的辅助操作。 弱三色不变性的好处在于：**只要存在未访问的能够到达白色对象的路径，就可以将黑色对象指向白色对象**
如果考虑并发的用户态代码，回收器不允许同时停止所有赋值器，就是涉及了存在的多个不同状态的赋值器。为了对概念加以明确，把回收器视为对象，把赋值器视为影响回收器这一对象的实际行为（即影响 GC 周期的长短），从而引入赋值器的颜色：
- 黑色赋值器：已经由回收器扫描过，不会再次对其进行扫描。
- 灰色赋值器：尚未被回收器扫描过，或尽管已经扫描过但仍需要重新扫描。
赋值器的颜色对回收周期的结束产生影响：
- 如果某种并发回收器允许灰色赋值器的存在，则必须在回收结束之前重新扫描对象图。
- 如果重新扫描过程中发现了新的灰色或白色对象，回收器还需要对新发现的对象进行追踪，但是在新追踪的过程中，赋值器仍然可能在其根中插入新的非黑色的引用，如此往复，直到重新扫描过程中没有发现新的白色或灰色对象。
于是，在允许灰色赋值器存在的算法，最坏的情况下，回收器只能将所有赋值器线程停止才能完成其跟对象的完整扫描，也就是我们所说的 STW
为了确保强弱三色不变性的并发指针更新操作，需要通过赋值器屏障技术来保证指针的读写操作一致。因此我们所说的 **Go 中的写屏障、混合写屏障，其实是指赋值器的写屏障**，赋值器的写屏障作为一种同步机制，使赋值器在进行指针写操作时，能够“通知”回收器，进而不会破坏弱三色不变性
**Dijkstra插入屏障**的好处在于可以立刻开始并发标记。但存在两个缺点：
1. 由于 Dijkstra 插入屏障的“保守”，在一次回收过程中可能会残留一部分对象没有回收成功，只有在下一个回收过程中才会被回收；
2. 在标记阶段中，每次进行指针赋值操作时，都需要引入写屏障，这无疑会增加大量性能开销；为了避免造成性能问题，Go 团队在最终实现时，没有为所有栈上的指针写操作，启用写屏障；而是当发生栈上的写操作时，将栈标记为灰色，但此举产生了灰色赋值器，将会需要标记终止阶段 STW 时对这些栈进行重新扫描。
**Yuasa删除屏障**的优势则在于不需要标记结束阶段的重新扫描，结束时候能够准确的回收所有需要回收的白色对象。缺陷是 Yuasa 删除屏障会拦截写操作，进而导致波面的退后，产生“冗余”的扫描
Go 在 1.8 的时候为了简化 GC 的流程，同时减少标记终止阶段的重扫成本，将 Dijkstra 插入屏障和 Yuasa 删除屏障进行混合，形成混合写屏障。该屏障提出时的基本思想是：**对正在被覆盖的对象进行着色，且如果当前栈未扫描完成，则同样对指针进行着色**