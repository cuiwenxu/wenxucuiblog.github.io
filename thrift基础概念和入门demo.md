@[TOC](thrift基础概念和入门demo)
# 概述
Thrift是一个轻量级、跨语言的远程服务调用框架，最初由Facebook开发，后面进入Apache开源项目。它通过自身的IDL中间语言, 并借助代码生成引擎生成各种主流语言的RPC服务端/客户端模板代码。

Thrift支持多种不同的编程语言，包括C++、Java、Python、PHP、Ruby等，本系列主要讲述基于Java语言的Thrift的配置方式和具体使用。
# thrift技术栈
Thrift对软件栈的定义非常的清晰, 使得各个组件能够松散的耦合, 针对不同的应用场景, 选择不同是方式去搭建服务。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210613150709642.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
Thrift软件栈分层从下向上分别为：传输层(Transport Layer)、协议层(Protocol Layer)、处理层(Processor Layer)和服务层(Server Layer)。
- 传输层(Transport Layer)：传输层负责直接从网络中读取和写入数据，它定义了具体的网络传输协议；比如说TCP/IP传输等。
- 协议层(Protocol Layer)：协议层定义了数据传输格式，负责网络传输数据的序列化和反序列化；比如说JSON、XML、二进制数据等。
- 处理层(Processor Layer)：处理层是由具体的IDL（接口描述语言）生成的，封装了具体的底层网络传输和序列化方式，并委托给用户实现的Handler进行处理。
- 服务层(Server Layer)：整合上述组件，提供具体的网络线程/IO服务模型，形成最终的服务。
# thrift优点
## (一) 开发速度快
`通过编写RPC接口Thrift IDL文件，利用编译生成器自动生成服务端骨架(Skeletons)和客户端桩(Stubs)。从而省去开发者自定义和维护接口编解码、消息传输、服务器多线程模型等基础工作。`

- `服务端：只需要按照服务骨架即接口，编写好具体的业务处理程序(Handler)即实现类即可。`
- `客户端：只需要拷贝IDL定义好的客户端桩和服务对象，然后就像调用本地对象的方法一样调用远端服务。`

## (二) 接口维护简单
通过维护Thrift格式的IDL（接口描述语言）文件（注意写好注释），即可作为给Client使用的接口文档使用，也自动生成接口代码，始终保持代码和文档的一致性。且Thrift协议可灵活支持接口的可扩展性。
## (三) 学习成本低
因为其来自Google Protobuf开发团队，所以其IDL文件风格类似Google Protobuf，且更加易读易懂；特别是RPC服务接口的风格就像写一个面向对象的Class一样简单。
初学者只需参照：thrift.apache.org，一个多小时就可以理解Thrift IDL文件的语法使用。
## (四) 多语言/跨语言支持
Thrift支持C++、 Java、Python、PHP、Ruby、Erlang、Perl、Haskell、C#、Cocoa、JavaScript、Node.js、Smalltalk等多种语言，即可生成上述语言的服务器端和客户端程序。

# Thrift的数据类型
Thrift 脚本可定义的数据类型包括以下几种类型：

- 基本类型：
bool: 布尔值
byte: 8位有符号整数
i16: 16位有符号整数
i32: 32位有符号整数
i64: 64位有符号整数
double: 64位浮点数
string: UTF-8编码的字符串
binary: 二进制串

- 结构体类型：
struct: 定义的结构体对象

- 容器类型：
list: 有序元素列表
set: 无序无重复元素集合
map: 有序的key/value集合

- 异常类型：
exception: 异常类型

- 服务类型：
`service: 具体对应服务的类`

# Thrift的协议
Thrift可以让用户选择客户端与服务端之间传输通信协议的类别，在传输协议上总体划分为文本(text)和二进制(binary)传输协议。为节约带宽，提高传输效率，一般情况下使用二进制类型的传输协议为多数，有时还会使用基于文本类型的协议，这需要根据项目/产品中的实际需求。常用协议有以下几种：

- TBinaryProtocol：二进制编码格式进行数据传输
- TCompactProtocol：高效率的、密集的二进制编码格式进行数据传输
- TJSONProtocol： 使用JSON文本的数据编码协议进行数据传输
- TSimpleJSONProtocol：只提供JSON只写的协议，适用于通过脚本语言解析

# Thrift的传输层
常用的传输层有以下几种：

- TSocket：使用阻塞式I/O进行传输，是最常见的模式
- TNonblockingTransport：使用非阻塞方式，用于构建异步客户端
- TFramedTransport：使用非阻塞方式，按块的大小进行传输，类似于Java中的NIO

# Thrift的服务端类型

- TSimpleServer：单线程服务器端，使用标准的阻塞式I/O
- TThreadPoolServer：多线程服务器端，使用标准的阻塞式I/O
- TNonblockingServer：单线程服务器端，使用非阻塞式I/O
- THsHaServer：半同步半异步服务器端，基于非阻塞式IO读写和多线程工作任务处理
- TThreadedSelectorServer：多线程选择器服务器端，对THsHaServer在异步IO模型上进行增强

# Thrift入门示例
## 1、首先编写idl文件
```java
namespace java hello.thrift
service HelloWorldService{
 string helloString(1:string para)
}
```
## 2、使用thrift -gen生成代码
>thrift -gen java hello.thrift

一个简单的hello.thrift生成的内容多达上千行，这些代码结构如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210613153746800.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

对于开发人员而言，使用原生的Thrift框架，`仅需要关注以下四个核心内部接口/类：Iface, AsyncIface, Client和AsyncClient。`

- Iface：服务端通过实现HelloWorldService.Iface接口，向客户端的提供具体的同步业务逻辑。
- AsyncIface：服务端通过实现HelloWorldService.AsyncIface接口，向客户端的提供具体的异步业务逻辑。
- Client：客户端通过HelloWorldService.Client的实例对象，以同步的方式访问服务端提供的服务方法。
- AsyncClient：客户端通过HelloWorldService.AsyncClient的实例对象，以异步的方式访问服务端提供的服务方法。

其中Iface和AsyncIface是给服务端用的，相当于服务端skeleton。Client和AsyncClient是给客户端用的，相当于stub。接下来看这两部分分别怎么使用
## 3、编写服务端&客户端代码
### 3.1、编写service实现类
新建maven工程，引入thrift的依赖，这里使用的是版本0.10.0。
 ```xml
  <dependency>
      <groupId>org.apache.thrift</groupId>
      <artifactId>libthrift</artifactId>
      <version>0.10.0</version>
  </dependency>
  ```
将生成类的HelloWorldService.java源文件拷贝进项目源文件目录中，并实现HelloWorldService.Iface的定义的helloString()方法。

HelloWorldServiceImpl.java代码如下
```java
public class HelloWorldServiceImpl implements HelloWorldService.Iface {
    @Override
    public String helloString(String username) throws TException {
        return "Hello " + username;
    }
}
```
### 3.2、服务器端代码编写
SimpleServer.java
```java
public class SimpleServer {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(9000));
        TServerSocket serverTransport = new TServerSocket(serverSocket);
        HelloWorldService.Processor processor =
                new HelloWorldService.Processor<HelloWorldService.Iface>(new HelloWorldServiceImpl());

        TBinaryProtocol.Factory protocolFactory = new TBinaryProtocol.Factory();
        TSimpleServer.Args tArgs = new TSimpleServer.Args(serverTransport);
        tArgs.processor(processor);
        tArgs.protocolFactory(protocolFactory);

        // 简单的单线程服务模型 一般用于测试
        TServer tServer = new TSimpleServer(tArgs);
        System.out.println("Running Simple Server");
        tServer.serve();
    }
}
```
### 3.3、客户端代码编写
SimpleClient.java
```java
public class SimpleClient {
    public static void main(String[] args) {
        TTransport transport = null;
        try {
            transport = new TSocket(ServerConfig.SERVER_IP, ServerConfig.SERVER_PORT, ServerConfig.TIMEOUT);
            TProtocol protocol = new TBinaryProtocol(transport);
            HelloWorldService.Client client = new HelloWorldService.Client(protocol);
            transport.open();

            String result = client.helloString("Leo");
            System.out.println("Result =: " + result);
        } catch (TException e) {
            e.printStackTrace();
        } finally {
            if (null != transport) {
                transport.close();
            }
        }
    }
}
```
# demo url

https://github.com/cuiwenxu/thriftdemo/tree/master/













