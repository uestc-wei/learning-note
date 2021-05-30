# TRPC学习笔记

## 一、为什么用TRPC，与http restful 有何不同，优点在哪里？

rpc与restful的一些比较：

- restful将所有的应用定义成为“资源”，通过对资源进行GET、POST、PUT、DELETE操作。由于http的通用性，这种方式比较通用。
- rpc由于是基于传输层，长链接，不必每次通信都去建立3次握手，减少开销，效率更高。HTTP请求往往围绕资源，而RPC的请求往往围绕一个动作。

总结一下：http适用于暴露接口给任何人使用，rpc灵活高效，在内部更加适用。

TRPC作为一个能和微服务治理系统灵活对接的微服务框架，通过插件化的方式，简化了平台对接。一般的组合为：tRPC框架+tRPC工具集合+插件+微服务运营平台。

## 二、基础概念

### 1.1 RPC

RPC（Remote Procedure Call Protocol）：远程过程调用（==客户端远程调用服务端对象方法==）。

一些要点：

- RPC就是从一台机器（客户端）上通过参数传递的方式调用另一台机器（服务器）上的一个函数或方法（可以统称为服务）并得到返回的结果。
- 客户端发起请求，服务器返回响应（类似于Http的工作方式） RPC 在使用形式上像调用本地函数（或方法）一样去调用远程的函数（或方法）。
- RPC 会隐藏底层的通讯细节（不需要直接处理Socket通讯或Http通讯） RPC 是一个请求响应模型。

![1620736192(1)](C:\Users\sammywei\Desktop\1620736192(1).jpg "RPC要素图")

- Client：RPC协议的调用方。
- Server：远程服务方法的具体实现。
- Stub/Proxy：RPC代理。存在于客户端，负责去管理消息格式、管理网络传输协议、处理异常。
- Message Protocol：消息格式。对传输的消息进行编码和解码。
- Transfer/Network Protocol：传输协议。
- Selector/Processor：存在于RPC服务端，管理RPC接口的注册、判断客户端是否有权限等等。
- IDL：接口语言，跨语言

### 1.2 实现rpc框架需要解决的问题

- `寻址`: 客户端调用的时候怎么告诉服务端调用的是哪个函数或方法
- `序列化反序列化`: 由于客户端和服务端不是同一个进程不能通过内存来传递参数，因此需要客户端先把参数序列化成字节流传给服务端，服务端收到字节流后反序列为自己能读取的格式，序列化反序列可以使用Protobuf、JSON等。
- `网络传输`: 客户端和服务端需要通过网络连接来传输数据，因此需要有一个网络的传输层。网络传输可以使用Socket、TCP、UDP、HTTP、HTTP2等。

 ### 1.3 常用的rpc框架

```java
   1. dubbo
   2. grpc
```

### 1.4 TRPC

#### 1.4.1 TRPC术语

![image-20210512104143710](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512104143710.png)

- 路由标识（Naming Service）,格式为 **trpc.{app}.{server}.{service}**
  - app：一般为一组服务的集合，即一个业务系统
  - server：一个服务进程，对应一个ip
  - service：服务提供者，一般对应ip+端口+协议
- 接口标识 （proto service），格式为**/{package}.{proto service}/{method}**，
  - package：包名，对应pb中的package
  - proto service：协议服务名，一组rpc方法的集合。一个server对应多个proto service
  - method：rpc方法，一个proto service对应多个method

#### 1.4.2 接口映射

上述的 负责rpc接口的定义和归类，naming service网络通信的建立和协议的处理。proto service注册到naming service中。

- 单服务模式，一个服务端对应一个proto service，对应一个naming service，一个端口对外提供服务。

![image-20210512112342046](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512112342046.png)

- 单服务多协议，一个server对应多个name service，比如多个协议。

![image-20210512112628005](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512112628005.png)

- 多服务多协议：一个server对应多个proto service，每个proto service对应多个name service

  ![image-20210512112815353](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512112815353.png)

#### 1.4.3路由寻址

![image-20210512113553845](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512113553845.png)

上图是一次RPC调用的过程（selecor）。

- 服务发现：客户端向名字服务询问指定服务的地址
- 服务路由：根据已有的策略对名字服务返回的地址进行筛选
- 负载均衡：根据一定策略对客户端实例进行选取
- 熔断：每个rpc请求进行上报，触发熔断

## 三、TRPC架构

### 1.1 RPC内核设计

![image-20210512114827267](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512114827267.png)

- 客户端发送请求

  1. 借助于动态代理的能力，客户端调用api的过程中实际上会导向到框架注册的代理handler方法中，会将用户调用api的相关元数据信息包括方法名，方法参数类型，方法参数值，方法的返回值，方法返回的类型(同步/异步/流式）等封装到我们的抽象的一个Request对象中。
  2. ClusterInvoker是我们抽象的一个集群分布式场景下的方法调用的抽象，接收代理层传递的Request对象 其中ClusterInvoker集成了服务发现，路由，熔断，降级的能力, 最终从集群中挑选一个Invoker(通常指某一个ip)进行请求调用
  3. ConsumerInvoker是我们抽象的某一个端点的方法调用抽象，其中内置了Filter链, 用于客户端的方法请求的前置，后置，异常等处安插自定义处理逻辑
  4. ConsumerInvoker内置了Transport能力, 将Request进行编码，序列化，压缩后的二进制流进行网络传输达到服务端

- 服务端接收&响应

  1. 框架会将用户的编写的本地方法包装成ProviderInvoker对象, ProviderInvoker正如前面所说是服务端的远程方法的一种抽象，会将服务端的远程方法的接口，方法列表，方法参数，方法返回值，方法返回类型（同步、异步）注册到ProviderInvoker对象中。
  2. ProviderInvoker注册到RpcServer中后，就具备了远程方法的能力，其中ProviderInvoker内置了filter的能力，在调用的前置，后置，异常等处安插自定义逻辑
  3. Transport接收到客户端的请求后，进行解码，解压缩，反序列化后得到客户端传递过来的参数对象以及协议相关的头信息
  4. 通过参数对象中携带的接口名与方法名就可以路由到对应的ProviderInvoker当中，就可以借助于反射的力量调用用户编辑的逻辑方法得到对应的响应值，然后将对应的响应值通过Transport进行编码，压缩，序列化得到二进制流进而通过网络传递到客户端

- 客户端接收响应

  Transport接收到服务端的请求后，对请求进行解码，解压缩，反序列化，就得到服务端传递的响应值，进行将响应值返回给动态代理的Handler进而返回给Caller.此时Caller就像调用本地方法一样调用了远程方法。

### 1.2 服务启动源码

- 入口：TRpcContainer.start() 容器启动之后，会加载对应的yaml配置文件，从中解析出ProviderConfig
- ProviderConfig.export() 初始化RpcServer, 同时将ProviderConfig中对应的接口类，接口实现对象等注册到RpcServer中，此时对应的接口方法就具备远程方法的能力了。
- ProviderConfig.registe() 将ProviderConfig相关的服务信息注册到对应的注册中心，此时对应的接口方法能够被其他的客户端服务发现了。

​        1. container.start()

<img src="C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512154748262.png" alt="image-20210512154748262" style="zoom:50%;" />

      2. start()中加载配置文件。调用configManager.start()进行初始化。

<img src="C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512155515931.png" alt="image-20210512155515931" style="zoom:50%;" />

3. configmanager.start() 读取文件后生成global config、serverConfig、clientConfig。开始初始化。

   4.configmanager.start()会调用serverConfig.registe()进行注册。

![image-20210512162734886](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512162734886.png)

5.serverConfig.registe()调用serviceConfig.registe()

![image-20210512163714743](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512163714743.png)

### 1.3 客户端启动源码

1. 入口：TRpcContainer.start() 容器启动之后，会加载对应的yaml配置文件，从中解析出ConsumerConfig
2. ConsumerConfig.getProxy() 初始化RpcClient, 同时借助于RpcClient.getProxy()生成代理对象。

### 1.4 自定义RPC

![image-20210512174852827](C:\Users\sammywei\AppData\Roaming\Typora\typora-user-images\image-20210512174852827.png)

​        目前tRPC协议默认使用pb协议文件中/{package}.{service}/{method}作为服务内部路由路径。

