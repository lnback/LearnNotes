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
### 内存分配
### 垃圾回收

## CSP

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