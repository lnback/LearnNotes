# Go基础
## slice、数组、Map
### slice和数组有什么异同
slice的底层数据是数组，slice是对数组的封装，他描述一个数组的片段。两者都可以通过下标来访问单个元素。

数组是定长的，长度定义好之后，不能再更改。在go中，数组是不常见的，因为其长度是类型的一部分，限制了它的表达能力。

而切片非常灵活，可以动态地扩容。切片的类型和长度无关。

数组就是一片连续的内存，slice实际上是一个结构体，包含三个字段：长度、容量、底层数组。
```go
type slice struct{
    array unsafe.Pointer //元素指针
    len int //长度
    cap int //容量
}
```

底层数组是可以呗多个slice同时指向的，因此对一个slice的元素进行操作是有可能影响到其他slice的

- [3]int和[4]int是同一个类型吗？

不是。因为数组的长度也是类型的一部分，这是与slice不同的一点。
- 举个栗子
```go
func main(){
    slice := []int{1,2,3,4,5,6,7,8,9,10}
    s1 := slice[2:5] //len 3 cap 8
    s2 := s1[2:6:7]  //len 4 cap 5

    s2 = append(s2,100) //len 5 cap 5 这里的append也会修改slice和s1的数据 slice[8]
    s2 = append(s2,200) //len 6 cap 10 在这里扩容了 len*2 扩容后s2使用的内存和s1、slice无关了

    s1[2] = 20 //也会修改slice，但不会修改s2

    fmt.Println(s1) //[3,4,20] 
    fmt.Println(s2) //[5,6,7,8,100,200]
    fmt.Println(slice) //[1,2,3,4,20,5,6,7,8,100,10]
}
```
### slice扩容

先说扩容规则：
- 当slice小于1024时，扩容为原来容量的两倍
- 当slice大于1024时，扩容为原来容量的1.25倍（大于等于）

什么时候扩容：
一般都是在向slice追加了元素之后，才会引起扩容。追加元素调用的是`append`函数
```go
func append(slice []Type,elems ...Type) []Type
```
append函数的参数长度可变，因此可以追加多个值到slice中，还可以用...传入slice，直接追加一个切片。
```go
slice = append(slice,elem1,elem2)
slice = append(slice,otherSlice...)
```
使用 append 可以向 slice 追加元素，实际上是往底层数组添加元素。但是底层数组的长度是固定的，如果索引 len-1 所指向的元素已经是底层数组的最后一个元素，就没法再添加了。

这时，slice 会迁移到新的内存位置，新底层数组的长度也会增加，这样就可以放置新增的元素。同时，为了应对未来可能再次发生的 append 操作，新的底层数组的长度，也就是新 slice 的容量是留了一定的 buffer 的。否则，每次添加元素的时候，都会发生迁移，成本太高。

每次扩容时，都会调用`growSlice`看看源代码：
```go
func growSlice(et *_type,old slice,cap int) slice{
    newcap := old.cap
    doublecap := newcap + newcap
    if cap > doublecap{
        newcap = cap
    }else{
        if old.len < 1024 {
            newcap = doublecap
        }else{
            for newcap < cap{
                newcap += newcap / 4
            }
        }
    }
    capmem = roundupsize(uintptr(newcap) * ptrSize)
    newcap = int(capmem / ptrSize)
}
```
前面说的扩容规则中为什么是大于等于1.25呢？因为后面要对newcap做一个内存对齐，这个和内存分配策略相关。进行内存对齐之后，新slice的容量要大于等于**老slice**的2倍或者1.25倍。

在这里有一个坑
```go
func main(){
    s := []int{1,2}
    s = append(s,1,2,3)

    fmt.Println(cap(s)) // 6

    s1 := []int{1,2}
    s = append(s,1)
    s = append(s,2)
    s = append(s,3)
    fmt.Println(cap(s)) // 8
}
```

为什么一个是6一个是8呢？
在第一个结果为6的例子中，原本slice的容量为2，`append`三个元素后长度变成5，容量最小要变成5，也就是传入的第三个参数应该为5，即`cap=5`。而`doublecap`是原slice容量的2倍也就是4。满足第一个if条件，`newcap`变成了5。
在扩容的时候会调用`roundupsize`，这个函数通过计算后会返回一个capmem，这个capmem会影响newcap。
```go
func roundupsize(size uintptr) uintptr {
    if size < _MaxSmallSize {
        if size <= smallSizeMax-8 {
            return uintptr(class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]])
        } else {
            //……
        }
    }
    //……
}

const _MaxSmallSize = 32768
const smallSizeMax = 1024
const smallSizeDiv = 8
```

最后返回的是`class_to_size[size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]]`这个式子的结果

这是`Go`源码中有关内存分配的两个slice。
```go
var size_to_class8 = [smallSizeMax/smallSizeDiv + 1]uint8{0, 1, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15, 16, 16, 17, 17, 18, 18, 18, 18, 19, 19, 19, 19, 20, 20, 20, 20, 21, 21, 21, 21, 22, 22, 22, 22, 23, 23, 23, 23, 24, 24, 24, 24, 25, 25, 25, 25, 26, 26, 26, 26, 26, 26, 26, 26, 27, 27, 27, 27, 27, 27, 27, 27, 28, 28, 28, 28, 28, 28, 28, 28, 29, 29, 29, 29, 29, 29, 29, 29, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 30, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31}

var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

传进来的`size`等于40。所以`(size+smallSizeDiv-1)/smallSizeDiv=5`；获取`size_to_class8`数组中索引为`5`的元素为`4`；获取`class_to_size`中索引为`4`的元素为`48`。

最终，新的slice的容量为`6`：

```go
newcap = int(capmem /ptrSize) // 6
```
### Map
#### 底层数据结构
Map主要由两部分构成
存放map信息的hmap
```go
type hmap struct{
    count int //当前哈希表中的元素数量
    flags uint8 //并发读写标志
    B uint8 //buckets数组的长度的对数，也就是说buckets数组的长度就是2^B
    noverflow uint16 //溢出桶数量
    hash0 uint32 //哈希种子
    
    buckets unsafe.Pointer //是一个指针，指向一个bmap
    oldbuckets unsafe.Pointer //在哈希扩容时用于保存之前buckets的字段，它的大小是当前buckets的一般
    
    nevacuate uintptr

    extra *mapextra
}
```
存放数据的bucket（哈希桶）
```go
type bmap struct{
    tophash [bucketCnt]uint8
}

// 编译期间会给它创建一个新的结构：
type bmap struct{
    topbits [8]uint8
    keys [8]keytype
    values [8]valuetype
    pad uintptr
    overflow uintptr
}
```
bmap是存放k-v的地方，bmap的内部组成不是按照key/value/key/value/.../key/value这样把key/value对应起来的，这样存储的话要内存对齐很多次。如果是这样key/key/.../value/value这样存放的，只需要在最后内存对齐。
#### 赋值

#### 扩容
golang的扩容分为两个类型：
- overflow bucket太多

这种类型的扩容只会等量扩容，因为是overflow bucket太多，导致有很多空间没有利用起来。
- 负载因子大于6.5

这种类型的扩容会直接扩容两倍，把原来的B+1，申请一个$2^{B+1}$的新buckets。

#### 迁移
迁移的目的就是将老的buckets搬迁到新的buckets。

Go map 的扩容采取了一种称为“渐进式”地方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。
- 第一个类型：overflow buckets太多，由于buckets数量不变，因此可以按序号来迁移，比如原来在0号buckets，到了新的地方后还是0号buckets。
- 第二个类型：需要重新计算key的哈希，才能决定它落在哪个bucket。例如原来B=5，计算出key的哈希后只用看他的低5位。扩容后，B=6，因此需要多看一位，它的低6位决定key落在哪个bucket。

#### 其他
- Map不是一个线程安全的数据结构，可以使用sync.Map
- Map不能边遍历边删除
- 不能对map的key或value进行取址
- Map的key是无序的
## 并发编程
### sync.Mutex
### sync.RWMutex
### sync.Pool
### sync.Map
### sync.WaitGroup
### sync.Once

## 运行时
### 并发调度
#### GPM模型
![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210225150442.png)
1.  全局队列（Global Queue）：存放等待运行的G
2.  P的本地队列：同全局队列类似，存放的也是等待运行的G，存的数量有限。不超过256个。新建G时，G优先加入到P的本地对烈烈，如果队列满了，则会把一半的G移动到全局队列中。
3.  P列表：所有的P都在程序启动时创建，并保存在数组中，最多有`GOMAXPROCS`个。
4.  M：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列拿一批G放到P的本地队列，或从其他P的本地队列偷一半放到自己P的本地队列中。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。

Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了一个内核线程，OS调度器负责把内核线程分配到CPU的核上执行。

> 有关M和P的个数问题
1.  P的数量：

- 由启动时环境变量`GOMAXPROCS`或者是由runtime的方法`GOMAXPROCS()`决定。意味着在程序执行的任意时刻都只有`GOMAXPROCS`个goroutine在同时运行
2.  M的数量
- go语言本身的限制：go程序启动时，会设置M的最大数量，默认10000。但是内核很难支持这么多的线程数，这个限制可以忽略。
- runtime/debug中的SetMaxThreads函数，设置M的最大数量
- 一个M阻塞了，如果没有空闲的M，则会创建新的M。
> P和M何时被创建

1.  P何时创建：在确定了P的最大数量后，运行时系统会根据这个最大数量创建P。
2.  M何时创建：没有足够的M来绑定P并运行其中的可运行的G。比如所有的M都被阻塞了，而P中还有很多就绪任务，就回去寻找空闲的M，没有空闲的就回去创建新的M。
#### 调度器的设计策略
- 复用线程：避免频繁的创建、销毁线程，而是对线程的复用
  1.  work stealing机制：当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。
  2.  hand off：当本线程因为G进行系统调用阻塞时，线程释放绑定的P，而P转移给其他空闲的线程执行。
- 利用并行：`GOMAXPROCS`设置P的数量，最多有`GOMAXPROCS`个线程分布在多个CPU上同时运行。`GOMAXPROCS`也限制了并发的程度，比如`GOMAXPROCS` = 核数/2，则最多利用了一半的CPU核进行并行。
- 抢占：在coroutine中要等待一个协程主动让出CPU才能执行下一个协程。在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这时goroutine和coroutine不同点。
#### go func()调度策略
![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210226162435.jpeg)
1.  通过go func()来创建一个goroutine
2.  有两个存储G的队列，一个是局部调度器P的本地队列、一个事全局G队列。新创建的G会先保存在P的本地队列中，如果P的本地队列已经满了就会保存在全局的队列中。
3.  G只能运行在M中，一个M必须持有P，M与P是1:1关系。M会从P的本地队列弹出一个可执行状态的G来执行，如果P的本地队列为空，就会向其他的MP组合偷取可执行的G来执行。
4.  一个M调度G执行的过程是一个循环机制
5.  当M执行某一个G的时候如果发生了syscall或其余阻塞操作，M会阻塞，如果当前有一些G在执行，runtime会把这个线程M从P中摘除，然后再找一个空闲的M来绑定P，如果没有空闲的就新建一个M。
6.  当M系统调用结束时，这个G会尝试获取一个空闲的P执行，并放入到这个P的本地队列。如果获取不到P，那么这个线程M变成休眠状态，加入到空闲线程中，然后这个G会被放入全局队列中。
#### 调度器的生命周期
![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210226164603.png)
特殊的M0和G0
- M0：`M0`是启动程序后的编号为0的主线程，这个m对应的实例会在全局变量runtime.m0中，不需要在heap上分配，M0负责执行初始化操作和启动第一个G，在之后M0就和其他的M一样了
- G0：`G0`是每次启动一个M都会第一个创建的goroutine，G0仅用于负责调度G，G0不指向任何可执行的函数，每个M都会有一个自己的G0，。在调度或系统调用时会使用G0的栈空间，全局变量的G0是M0的G0。
#### 协程和线程的区别
- 内存占用

创建一个goroutine的栈内存消耗为2KB，实际运行过程中，如果栈空间不够，会自动进行扩容（TODO：怎么进行扩容）。创建一个thread则需要消耗1MB栈内存，而且还需要一个被称为"a guard page"的区域用于和其他thread的栈空间进行隔离。

比如对于一个HTTP Server而言，对于到来的每个请求，goroutine处理起来非常轻松。而使用thread作为并发原语的语言构建的服务来说，每个请求对应一个线程太浪费资源了。
- 创建和销毁

thread的创建和销毁都会有巨大的消耗，因为要和操作系统打交道，是内核级别的，通常解决的办法是线程池。而goroutine因为是由Go runtime负责管理的，创建和销毁的消耗非常小，是用户级的。
- 切换

thread切换时，需要保存各种寄存器，以便恢复

而goroutine只需要保存三个寄存器：PC,SP,BP

所以goroutine切换成本比thread要小很多。
### 内存分配
#### 逃逸分析
#### 堆栈
#### make和new的区别
在Go语言中`new`和`make`是内建的两个函数，主要用来创建分配类型内存。
- new：

函数签名
```go
func new(Type) *Type
```
它只接受一个参数，这个参数是一个类型，分配好内存后，返回一个指向该类型内存地址的指针。同时把分配的内存置为零，也就是类型的零值。

- make：

make也是用于内存分配的，但是和new不同。
它只用于：
  - chan
  - map
  - slice

的内存创建，而且他返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。
```go
func make(t Type,size ...IntegerType) Type
```
从函数声明中可以看出，返回的还是该类型。

- make和new的异同
  - 相同
  堆空间分配
  - 不同：
  make：只用于slice、map、channel的初始化，无可替代
  new：用于类型内存分配（初始化值为0），不常用
### 垃圾回收
Go中使用的是三色标记法+混合写屏障来做垃圾回收。
#### 三色标记法
三色标记法实际上就是通过三个阶段的标记来确定清楚的对象都有哪些
1.  只要是新创建的对象，默认的颜色都是标记为白色
2.  每次GC回收开始，从根节点开始遍历所有对象，把遍历到的对象从白色集合放入灰色集合。
3.  遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合。
4.  重复第三步，直到灰色中无任何对象。
5.  回收所有的白色标记表的对象，也就是回收垃圾。

以上的步骤是一定要依赖STW的，如果没有STW，程序的逻辑改变对象引用关系（并发），这种动作如果在标记阶段做了修改，会影响标记结果的正确性。

有两个问题，在三色标记法中是不希望发生的：
- 一个白色对象被黑色对象引用（白色节点被挂在黑色节点下）
- 灰色对象的可达链中的白色对象遭到破坏（灰色丢失了该白色节点）

如果发生了上述两个问题，有很多可达链中白色的对象都会被回收。

众所周知，STW会暂停程序的执行，会消耗很多的时间。所以在go中引入了屏障机制。
#### 屏障机制

“强-弱”三色不变式
- 强三色不变式：不存在黑色对象引用到白色对象的指针
- 弱三色不变式：所有被黑色对象引用的白色对象都处于灰色保护状态

插入屏障：
- 具体操作：在A对象引用B对象的时候，B对象被标记为灰色。（将B挂在A下游，B必须被标记为灰色）
- 满足：强三色不变式。（黑色对象不会直接挂上白色对象）
- 缺点：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活。
删除屏障：
- 具体操作：被删除的对象，如果自身为灰色或者白色，那么被标记为灰色。
- 满足：弱三色不变式。（保护灰色对象到白色对象的路径不断）
- 缺点：回收精度低，GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。

---
混合写屏障：
具体操作：
1.  GC开始将栈上的对象全部扫描并标记为黑色（之后不再进行第二次重复扫描，无需STW）
2.  GC期间，任何在栈上创建的新对象，均为黑色。
3.  被删除的对象标记为灰色。
4.  被添加的对象标记为灰色。

Golang中的混合写屏障满足弱三色不变式，结合了删除写屏障和插入写屏障的优点，只需要在开始时并发扫描各个goroutine的栈，使其变黑并一直保持，这个过程不需要STW，而标记结束后，因为栈在扫描后始终是黑色的，也无需再进行re-scan操作了，减少了STW的时间。

混合写层屏障在栈中不启动，在堆启动。整个过程几乎不需要STW，效率较高。
### 栈内存管理

### 
## unsafe.Pointer和uintptr
相比于C语言中指针的灵活，Go的指针多了一些限制。但这也算是Go的成功之处：既可以享受指针带来的便利，又避免了指针的危险性。
- 限制一：Go的指针不能进行数学运算
- 限制二：不同类型的指针不能进行转换
- 限制三：不同类型的指针变量不能使用==或!=进行比较
- 限制四：不同类型的指针变量不能进行赋值

### unsafe.Pointer
unsafe.Pointer在unsafe包中。提供了2点重要的能力：
- 任何类型的指针和unsafe.Pointer可以相互转换
- uintptr类型和unsafe.Pointer可以相互转换

unsafe.Pointer不能直接进行数学运算，但可以把它转换成uintptr，对uintptr类型进行数学运算，再转换成unsafe.Pointer。
### uintptr
```go
//uintptr是一个整数类型，它足够大，可以存储内存地址
type uintptr uintptr
```
还有一点要注意的是，uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。

### 利用unsafe包修改私有成员
```go
import (
	"fmt"
	"unsafe"
)

type Student struct {
	name string
	id int
}
func main()  {
	s1 := Student{
		name: "nn",
		id: 1,
	}
	id := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s1)) + unsafe.Sizeof(string(""))))
	*id = 2
	fmt.Println(s1)
}
```
## CSP
### 通过通信实现
## channel、goroutine
### channel
#### 特性
channel中的特性：
- 给一个空channel发送数据，造成永远阻塞
- 从一个空channel接收数据，造成永远阻塞
- 给一个已经关闭的channel发送数据，引起panic
- 从一个已经关闭的channel接收数据，如果缓冲区中为空，则返回一个零值
- 无缓冲的channel是同步的，而有缓冲的channel是非同步的

读写空阻塞，读关闭零值，写关闭错误。
#### 底层结构
```go
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
`buf`指向底层循环数组，只有缓冲型的channel才有。
`sendx`、`recvx`均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。
`sendq`、`recvq`分别表示被阻塞的goroutine，这些goroutine由于尝试读取channel或向channel发送数据而被阻塞。

创建一个容量为6的缓冲channel：
![](https://cdn.jsdelivr.net/gh/lnback/imgbed/img/20210301163818.png)


## 反射