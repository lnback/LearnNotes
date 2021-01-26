---

---
# 什么是RPC？
## 1.RPC框架
### 1.1 基本概念
RPC(Remote Procedure Call,远程过程调用)是一种计算机通信协议，允许调用不同进程空间的程序。RPC的客户端和服务器可以在一台机器上，也可以在不同的机器上。程序员使用时，就像调用本地程序一样，无需关注内部的实现细节。

不同的应用程序之间的通信方式有很多，比如浏览器和服务器之间广泛使用的基于HTTP协议的Restful API。与RPC相比，Restful API有相对统一的标准，因而更通用，兼容性更好，支持不同的语言。HTTP协议是基于文本的，一般拥有更好地可读性，但是缺点也很明显。

- Restful接口需要额外的定义，无论是客户端还是服务端，都需要额外的代码来处理，而RPC调用则更接近于直接调用。？？
- 基于HTTP协议的Restful报文冗余，承载了过多的无效信息，而RPC通常使用自定义的协议格式，减少冗余报文。
- RPC可以采用更高效的序列化协议，将文本转为二进制传输，获得更高的性能。
- 因为RPC的灵活性，所以更容易扩展和集成诸如注册中心、负载均衡等功能。

## 2.RPC框架需要解决什么问题
RPC框架需要解决什么问题？或者换一个问题，为什么需要RPC框架？

我们可以想象在两台机器上，两个应用程序之间需要通信，那么首先，需要确定采用的传输协议是什么？如果这两个应用程序位于不同的机器上，那么一般会选择TCP协议或者HTTP协议；那如果两个应用程序位于相同的机器，也可以选择Unix Socket协议。传输协议确定之后，还需要确定报文的编码格式，比如采取常用的JSON和XML，那如果报文比较大，还可能会选择protobuf等其他编码方式，甚至编码之后，再进行压缩。接收到获取报文则需要相反的过程，先解压再解码。

解决了传输协议和报文编码问题，接下来还需要解决一系列的可用性问题，例如，连接超时了怎么办？是否支持异步请求和并发？

如果服务端的实例很多，客户端并不关心这些实例的地址和部署位置，只关心自己能否获取到期待的结果，就引出了注册中心(registry)饥饿负载均衡(load balance)的问题。简单地说，客户端和服务端互相不感知对方的存在，服务端启动时将自己注册到注册中心，客户端调用时，从注册中心获取到所有可用的实例，选择一个来调用。这样服务端和客户端只需要感知注册中心的存在就够了。注册中心通常还需要实现服务动态添加、删除，使用心跳确保服务处于可用状态等功能。

再进一步，假设服务端是不同的团队提供的，如果没有统一的RPC框架，各个团队的服务提供方就需要各自实现一套消息编解码、连接池、收发线程、超时处理等“业务之外”的重复技术劳动，造成整体的低效。因此，“业务之外”的这部分公共的能力，即是RPC框架所需要具备的能力。
