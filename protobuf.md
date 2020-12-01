[toc]
# Protobuf
## Protobuf是什么？
Protobuf是Google开发的一种**数据描述语言**，能够将结构化数据序列化，可用于数据存储、通信协议等方面。
可以理解为更快的、更简单、更小的JSON或XML，区别在于Protocol Buffers是二进制，而JSON和XML是文本格式。

## Protobuf应用场景
Protobuf可以用于结构化数据串行化（序列化）。很适合做**数据存储或RPC数据**交换格式。可用于通讯协议、数据存储等领域的语言无关、平台无关、可扩展的序列化结构数据格式。

## Protobuf优缺点
为什么有了XML和JSON等已经很普遍的数据传输方式，还要设计出Protobuf这种新的协议呢？
### 优点
#### 性能好/效率高
- 时间维度：采用XML格式对数据进行序列化时，时间消耗上性能尚可。但使用XML格式对数据进行反序列化时的时间花费上，耗时长，性能差。
- 空间维度：XML格式为了保持较好的可读性，引入了一些冗余的文本信息。所以在使用XML格式进行存储时，也会消耗空间。
  
**整体而言：Prtobuf以高效的二进制方式存储，比XML小3到10倍，快20到100倍。**

#### 代码生成机制
- **代码生成机制的含义：**
在GO语言中，可以通过定义结构体封装描述一个对象，并构造一个新的结构体对象。比如定义一个Person结构体，并定义一个Person.go的文件
```go
type Person struct{
    Name string
    Sex int
    Age int
}
```

在分布式系统中，因为程序代码分开部署，比如A,B。A系统在调用B系统时，无法直接采用代码的形式进行调用，因为A系统中不存在B系统中的代码。因此，A系统只负责将调用和通信的数据以二进制数据包的形式传递给B系统，由B系统根据获取到的数据包，自己构建出对应的数据对象，生成数据对象定义代码文件。这种利用编译器，根据数据文件自动生成结构体定义和相关方法文件的机制叫做**代码生成机制**。

- **代码生成机制的优点**
    - 首先，代码生成机制能够极大的解放开发者编写数据协议解析过程的时间，提高工作效率；
    - 其次，易于开发者维护和迭代，当需求发生变更时，开发者只需要修改对应的数据传输文件内容即可完成所有的修改。
#### 支持向后兼容和向前兼容
- **向后兼容**
    在软件开发迭代和升级过程中，“后”可以理解为新版本，越新的版本越靠后。“前”意味着早起的版本或者先前的版本。向后兼容就是系统升级迭代后，仍然可以处理老版本的数据业务逻辑
- **向前兼容**
    向前兼容是系统代码未升级，但是接受到了新的数据，此时老版本生成的系统代码可以处理接收到的新类型的数据。

支持前后兼容是非常重要的一个特点，在庞大的系统开发中，往往不可能统一完成所有模块的升级，为了保证系统功能正常不受影响，应最大限度保证通讯协议的向前兼容和向后兼容。
#### 支持多种编程语言
Protobuf作为Google开源的一种数据协议，还有很多种的语言版本。在Google官方发布的Protobuf的源代码中包含了C++、Java、Python三种语言。

### 缺点

#### 可读性较差
为了提高性能，Protobuf采用了二进制格式进行编码。二进制格式编码对开发者来说是没办法进行阅读的。在进行程序调试的时候，比较困难。

#### 缺乏自描述
比如XML语言是一种自描述的标记语言，即字段标记的同时就表达了内容对应的含义。而Protobuf不是一种自描述语言，开发者对于二进制格式的Protobuf，没有办法知道所对应的真实的数据结构，在使用Protobuf协议传输时，**必须配备对应的proto配置文件。**

## Protobuf协议语法
### Protobuf的协议的格式
Protobuf协议规定：使用该协议进行序列化和反序列化操作时，首先定义传输数据的格式，并命名以".proto"为扩展名定义消息定义文件
### Message定义一个消息
例如：
```protobuf
message Order{
    required string order_id = 1;
    repeated int64 num = 2;
    optional int32 timestamp = 3;
}
```
- **指定字段类型：** 在proto协议中，字段的类型包括字符串(string)、整形(int32、int64)、枚举(enum)等类型
- **分配标识符：** 在消息字段中，每个字段都有一个唯一的标识符。最小的标识号可以从1开始，最大到536870911。不可以使用其中的[19000-19999]的标识号，Protobuf协议实现中对这些进行了预留。如果非要在.proto文件中使用这些预留标识号，编译时就会报警。
- **指定字段规则：** 字段的修饰符包含三种类型，分别是
    -   **required：** 表示该值是必须要设置的。
    -   **optional：** 消息格式中该字段可以有0个或1个值（不超过1个）
    -   **repeated：** 表示该值可以重复，相当于Go中的slice。

## 使用Protobuf的步骤
### 创建
创建扩展名为.proto的文件，并编写代码。比如创建person.proto文件，内容如下
```protobuf
syntax = "proto2" // 协议版本
package example
message Person{
    required string Name = 1;
    required int32 Age = 2;
    required string From = 3;
}
```
### 编译
编译.proto文件，生成Go语言文件。执行如下命令：
```protobuf
protoc --go_out =生成路径 文件路径
```
### 使用
```go
package main

import (
	"beegodemo/proto"
	"fmt"
	proto2 "github.com/golang/protobuf/proto"
	"log"
)

func main() {
	student := &proto.Student{
		Name: "lly",
		Age: 12,
	}
	data,err := proto2.Marshal(student)

	if err != nil{
		log.Fatal(err)

	}

	stu := &proto.Student{}
	err = proto2.Unmarshal(data,stu)

	if err != nil {
		log.Fatal(err)
	}

	fmt.Println(stu)

}
```
### 执行
```go
go build -o hello.exe
hello.exe
```
