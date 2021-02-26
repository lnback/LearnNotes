# Go基础
## slice和数组
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
## 并发编程
### sync.Mutex
### sync.RWMutex
### sync.Pool
### sync.Map
### sync.WaitGroup

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
### 垃圾回收
#### 三色标记法
#### 回收规则
#### 缺点
### 栈内存管理
## CSP
### 通过通信实现
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

## channel、goroutine
### channel
### goroutine

## 反射