# Anjuke Private Service
Zhang Erning <erning@anjuke.com>  
v1.2, Sep. 2013


## Goal

我们的目标是为PHP应用程序搭建一个轻量、灵活、高性能的异步远过程调用(RPC)的解决方案。同时也希望能够方便隔离前端的应用程序和后端的服务进程，使得产品的开发和部署更加便捷。

**轻量**

  * 可以将服务进程与PHP应用服务部署在同一台物理服务器上
  * 一台应用服务器上能够部署多种服务进程

**灵活**

  * PHP应用程序可以采用进程间通讯(IPC)的方式与部署在同一服务器上的服务进程交互
  * PHP应用程序也能够通过网络通讯(TCP)的方式与部署在其他服务器上的服务进程交互
  * 与服务进程通讯的方式可以通过配置文件调整，PHP的应用程序本身并不关心
  * 服务进程可以采用各种语言编写

**高性能**

  * 为PHP提供异步方法调用的基本目的就是为了解决了PHP程序在处理请求的时候单线程的问题，因此不能以牺牲性能为代价


## Architect

为了实现这几个基本目标，我们选用**ØMQ**作为底层实现的基础，采用PHP直接与服务进程通讯的方式，而不是通过庞大复杂的中间件。

服务进程(Service)以ØMQ的**XREP**套接字绑定在某个位置，客户程序(Client)用ØMQ的**XREQ**套接字连接服务进程。客户程序发送请求；服务进程处理请求，之后返回结果；客户程序接收结果并进行后续操作。

```
 +-------+      +-------+      +-------+      +-------+
 |  PHP  |      |  PHP  |      |  PHP  |      |  PHP  |
 +---+---+      +---+---+      +---+---+      +---+---+
    |              |              |              |  
    |              |              |              |  
    \------zmq-----+------zmq-----+------zmq-----/
            |              |              |
            |              |              |
      /-----+-----\  /-----+-----\  /-----+-----\
      |           |  |           |  |           |
      | Service A |  | Service B |  | Service C |
      |           |  |           |  |           |
      \-----------/  \-----------/  \-----------/
```

### Asynchronous

要求客户程序和服务进程间实现异步的方法调用。客户程序允许一次发送多条请求给服务进程，之后一起等待结果。

```
           Synchronous RPC                          Asynchronous RPC
 
  Client      Service A    Service B        Client      Service A    Service B
     |            |            |               |            |            |
 0 S |----------->|            |           0 S |----------->|            |
 1   |            |            |           1   |------------------------>|
 2   |            |            |           2   |            |            |
 3   |<-----------|            |           3   |<-----------|            |
 4   |------------------------>|           4   |            |            |
 5   |            |            |           5   |            |            |
 6   |            |            |           6 F |<------------------------|
 7   |            |            |           7   |            |            |
 8   |            |            |           8   |            |            |
 9 F |<------------------------|           9   |            |            |
     |            |            |               |            |            |

 The services can't proceed multiple       The services can proceed multiple
 requests in parallel. Thus wait time      requests in parallel. Thus client
 becomes long.                             can gather results faster.        
```

### Parallel Pipelining

当服务进程从一个客户程序收到多条请求时可以并行处理这些请求，当任何一个请求处理完成时，应当立即将这个请求的结果返回给客户程序。也就是说，服务进程返回结果的顺序不必与客户程序请求的顺序一致。

客户程序发起的每一个请求都会分配一个唯一的请求序列号。服务进程返回结果时，要求将相应请求的序列号原封不动的返回给客户程序，客户程序通过这一个序列号识别当前收到的结果是哪一个请求的。

      RPC without parallel pipelining           RPC with parallel pipelining

          Client           Service                  Client           Service
             |               |  |                      |               |  |
         0 S |--1----------->|  |                  0 S |--1----------->|  |
         1   |--2-------------->|                  1   |--2-------------->|
         2   |               |  |                  2   |               |  |
         3   |               |  |                  3   |<-2---------------|
         4   |               |  |                  4   |               |  |
         5   |<-1------------|  |                  5 F |<-1------------|  |
         6 F |<-2---------------|                  6   |               |  |
         7   |               |  |                  7   |               |  |
         8   |               |  |                  8   |               |  |
         9   |               |  |                  9   |               |  |
             |               |  |                      |               |  |

     The service must return in received       The service can return just when it
     order, even if former task is heavy       finished and the client can receive
     and following one is light.               replies faster.


### Flexible Worker

通常服务进程需要维护多个处理程序(Worker)，这样在接收到多个请求的时候能够并行处理这些请求。我们建议使用ØMQ来实现线程间(或进程间)的通讯，这样不必再为多线程间的共享数据而引入各种各样的锁机制，减少不必要的麻烦。

          A Service with several workers
     
     /-----------------------------------------\
     | Service                                 |
     |                                         |
     |          +--------+  +--------+         |
     |          : Worker |  : Worker |         |
     |          +--------+  +--------+         |
     |                                         |
     |    +--------+  +--------+  +--------+   |
     |    : Worker |  : Worker |  : Worker |   |
     |    +--------+  +--------+  +--------+   |
     |                                         |
     \-----------------------------------------/ 


我们这里还规范了一份处理程序和其所属的服务进程间的通讯协议。采用这一规范来实现的处理程序将能够被我们的通用服务进程调度和管理，使处理程序是采用不同的语言开发。我们的规范要求处理程序采用ØMQ的**XREQ**套接字连接至其所属的服务进程，发出准备好的消息，并等待分配的任务。


## Message Format

  * 客户程序与服务进程之间通讯的消息格式应当完全采用如下**APS/Client**规范。

如无特殊约定，以下的时间参数单位均为毫秒；其中Timestamp、Expiry为1970年1月1日零时起所经过的毫秒数。

### APS/Client
客户程序与服务进程间通过**REQUEST**和**REPLY**互相通讯。由客户程序向服务进程发起的是**REQUEST**命令；服务进程返回的是**REPLY**命令。

#### REQUEST
  * Frame 1: "APS12" (5 bytes string)
  * Frame 2: Sequence, Timestamp, Expiry (msgpacked array)
  * Frame 3: Method (printable string)
  * Frame 4: Request body (msgpack(params))
  
The following optional extra message

  * Frame 5: (octets)
  * ...

#### REPLY
  * Frame 1: "APS12" (5 bytes string)
  * Frame 2: Sequence, Timestamp, Status (msgpacked array)
  * Frame 3: Reply body (msgpack(result))

Following optional extra message

  * Frame 4: (octets)
  * ...


#### Extra Messages

  扩展信息由客户端与服务端自行协商定义。因此可以没有扩展信息的frame，也可以有一个或多个frames。

#### Method Conventions

方法名是可以显示的英文字母数字以及下划线组成。点符号(`.`)允许作为方法名的一部分，具有分割方法名的用途。

  * `method` - 简单函数调用形式
  * `resource.method` - 类REST风格，例如 `users.get`、`users.post`、`users.$uid.patch`
  * `package.module.method` - Java/Python的方法调用形式
  * `.method`  - 点开头的方法作为保留方法，留给APS服务端使用

保留方法

  * `.ping`
  * `.status`


## Availability

See project [kite-line](../../../kite-line).

## Realibility

可靠性的控制不此描述，由客户程序和通用服务进程的具体实现负责可靠性。
