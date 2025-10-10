https://juejin.cn/post/7559399069144989711

OPPO RDMA 在典型场景下的技术应用

当前 OPPO 的数据中心中已经有一定数量支持 RDMA 的网卡（包含 IB 及 ROCEv2），除了机器学习场景以外，之前的文章 ORPC

也已经分享了 OPPO 在 RPC over RDMA 传输的实践，具体 RDMA 相关前置知识也可以参考此篇文章。为了充分发挥 RDMA 低延迟、远程内存访问、bypass cpu/os、及高带宽的优势，我们选取了一些业务程序进行传输方案的改造和测试，并总结探讨一般业务程序改造为 RDMA 传输的经验。

**业务适配 RDMA 类型**

 RDMA 传输的适配，从业务场景的使用角度来看，大致可分为如下几种类型。

**场景一：** 机器学习、分布式存储等场景，使用社区成熟的方案，如在机器学习场景中使用的 NCCL、Tensorflow 等框架中都适配了多种传输方式（包含 tcp、rdma 等），块存储 Ceph 中也同时支持 tcp 及 rdma 两种通信模式，这种业务场景下业务侧更多关注的是配置及使用，在 IAAS 基础设施侧将 RDMA 环境准备好后，使能框架使用 rdma 的传输模式即可。

**场景二：** 业务程序使用类似于 RPC 远程调用的通信方式，业务侧需要将原有使用的 RPC（大部分是 GRPC）调用改为 ORPC 调用，在这种场景下业务和传输更像是两个独立的模块，通过 SDK 的方式进行调用，所以适配起来改造的代码并不多，通常是业务层面修改调用 RPC 的接口方式。但由于业务方可能使用多种编程语言，RPC over RDMA 需要进行编程语言进行适配。

**场景三：** 业务程序通信是私有化通信，比如使用 socket 套接字结合 epoll 完全自有实现的一套通信机制。这种场景下其实改造也区分情况，即业务 IO 与网络 IO 是否耦合，若比较解耦，代码中抽象出一层类似于最新 Redis 代码中 ConnectionType 这样的架构\[2]，那么只需要实现一套基于 RDMA 通信且符合 Redis ConnectionType 接口定义的新传输类型即可，改造量相对可控并且架构上也比较稳定；而若业务 IO 与网络 IO 结合的较为紧密的情况下，这种场景下往往改造起来会比较复杂，改造的时候需要抽丝剥茧的找出业务与网络之间的边界，再进行网络部分的改造。

**02**

     **Redis RDMA 改造方案分析**

   首先，以 Redis 改造为 RDMA 传输为例，分析基于 RDMA 传输的应用程序改造逻辑与流程。

   第一步是需要梳理出来 Redis 中与网络传输相关的逻辑，这部分有比较多的参考资料，这里简单总结一下。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/45059fb4d6934d3ea175245450c9d876~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=KY%2BfGKdZ9QmtU6UuMTzlRuM1wlw%3D)
     Redis 中实现了一套 Reactor 模式的事件处理逻辑名为 AE，其主要流程为：

1、使用 epoll 等机制监听各文件句柄，包括新建连接、以及已建立的连接等；

2、根据事件的不同调用对应的事件回调处理；

3、循环进行 epoll loop 并进行处理。

   参考\[2]中分析了当前 redis 的连接管理是围绕 connection 这个对象进行管理（可类比 socket 套接字的管理），抽象一层高于 socket 的 connection layer，以便兼容不同的传输层，各个字段解释如下。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/7bad63c776654b4e8211a03dfd747df6~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=2uq%2FabZbrkBXSN9VktvHD7ldKh0%3D)
**type**：各种连接类型的回调接口，定义了诸如事件回调、listen、accept、read、write 等接口，类比 tcp socket 实现的 proto\_ops。

**state**：当前连接的状态，如 CONNECTING/ACCEPTING/CONNECTED/CLOSED 等状态，类比 TCP 的状态管理。

**fd**：连接对应的文件句柄。

**iovcnt**：进行 iov 操作的最大值。

**private\_data**：保存私有数据，当前存放的是 redis 中 client 的指针。

**conn\_handler/write\_handler/read\_handler**：分别对应连接 connect、write、read 时的处理接口。

![]()
![]()
**get\_type**: connection 的连接类型，当前 redis 已支持 tcp、unix、tls 类型，返回字符串。

**init**：在每种网络连接模块注册时调用，各模块私有初始化，如 tcp、unix 类型当前未实现，tls 注册时做了一些 ssl 初始化的前置工作。

**ae\_handler**: redis 中的网络事件处理回调函数，redis 中使用 aeCreateFileEvent 为某个 fd 及事件注册处理函数为 ae\_handler，当 redis 的主循环 aeMain 中发现有响应的事件时会调用 ae\_handler 进行处理，如在 tcp 连接类型中 ae\_handler 为 connSocketEventHandler，该函数分别处理了链接建立、链接可读、链接可写三种事件。

**listen**: 监听于某个 IP 地址和端口，在 tcp 连接类型中对应的函数为 connSocketListen，该函数主要调用 bind、listen。

**accept\_handler**: redis 作为一个服务端，当接收到客户端新建连接的请求时候的处理函数，一般会被.accept 函数调用，比如在 tcp 连接类型中，connSocketAccept 调用 accept\_handler，该方法被注册为 connSocketAcceptHandler，主要是使用 accept 函数接收客户端请求，并调用 acceptCommonHandler 创建 client。

**addr**: 返回连接的地址信息，主要用于一些连接信息的 debug 日志。

**is\_local**：返回连接是否为本地连接，redis 在 protected 模式下时，调用该接口判断是否为本地连接进行校验。

**conn\_create/conn\_create\_accepted**：创建 connection，对于 tcp 连接类型，主要是申请 connection 的内存，以及 connection 初始化工作。

**shutdown/close**：释放 connection 的资源，关闭连接，当某个 redis 客户端移除时调用。

**connect/blocking\_connect**：实现 connection 的非阻塞和阻塞连接方法，在 tcp 连接类型中，非阻塞连接调用 aeCreateFileEvent 注册连接的可写事件，继而由后续的 ae\_handler 进行处理，实现非阻塞的连接；而阻塞连接则在实现时会等待连接建立完成。

**accept**：该方法在 redis 源码中有明确的定义，可直接调用上述 accept\_handler，tcp 连接类型中，该方法被注册为 connScoketAccept。

**write/writev/read**：和 linux 下系统调用 write、writev、read 行为一致，将数据发送至 connection 中，或者从 connection 中读取数据至相应缓冲区。

**set\_write\_handler**：注册一个写处理函数，tcp 连接类型中，该方法会注册 connection 可写事件，回调函数为 tcp 的 ae\_handler。

**set\_read\_handler**：注册一个读处理函数，tcp 连接类型中，该方法会注册 connection 可读事件，回调函数为 tcp 的 ae\_handler。

**sync\_write/sync\_read/sync\_readline**：同步读写接口，在 tcp 连接类型中实现逻辑是使用循环读写。

**has\_pending\_data**：检查 connection 中是否有尚未处理的数据，tcp 连接类型中该方法未实现，tls 连接类型中该方法被注册为 tlsHasPendingData，tls 在处理 connection 读事件时，会调用 SSL\_read 读取数据，但无法保证数据已经读取完成\[3]，所以在 tlsHasPendingData 函数中使用 SSL\_pending 检查缓冲区是否有未处理数据，若有的话则交由下面的 process\_pending\_data 进行处理。has\_pending\_data 方法主要在事件主循环 beforesleep 中调用，当有 pending data 时，事件主循环时不进行 wait，以便快速进行下一次的循环处理。

**process\_pending\_data**：处理检查 connection 中是否有尚未处理的数据，tcp 连接类型中该方法未实现，tls 连接类型中该方法被注册为 tlsProcessPendingData，主要是对 ssl 缓冲区里面的数据进行读取。process\_pending\_data 方法主要在事件主循环 beforesleep 中调用。

**get\_peer\_cert**：TLS 连接特殊方法。

结合当前代码中 tcp 及 tls 实现方法，梳理出和 redis connection 网络传输相关的流程：

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/2e331fa8142c4af0ae7ffb133252d608~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=%2B8feNxNi7BWwqaf4Ypbd8a4Rgs8%3D)
图：Redis Connection Call Graph

   对于 redis 来说新增一个 RDMA 方式的传输方式，即是要将 connection 中的各种方法按照上述定义去使用 RDMA 编程接口去实现。RDMA 编程一般采用 CM 管理连接加 Verbs 数据收发的模式，客户端与服务端的交互逻辑大致如下图所示，参考\[16]。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9e74aafbc9c94e6ab6b51ae9a69e6551~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=8%2B7s47OGMsZJqAmUAGrJ%2B6wdpUo%3D)
图：RDMA C/S Workflow

   字节跳动的 pizhenwei 同学目前在 redis 社区中已经提交了 redis over rdma 的 PR，参见\[4]，具体的代码均在 rdma.c 这一个文件中。由于 RDMA 在做远程内存访问时，需要使用对端的内存地址，所以作者实现了一套 RDMA 客户端与服务端的交互机制，用于通告对端进行远程内存写入的内存地址，参见\[5]。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e729e5e849e94fe5bfc9ec07c39ea983~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=6JcaAKI27fVBnhsY8iGNtwDZgYA%3D)
**交互逻辑及说明如下：**

1、增加了 RedisRdmaCmd，用于 Redis 客户端与服务端的控制面交互，如特性交换、Keepalive、内存地址交换等；

2、在客户端及服务端建立完成 RDMA 连接后，需要先进行控制面的交互，当内存地址交换完成后，方可以进行 Redis 实际数据的交互及处理；

3、控制面消息通过 IBV\_WR\_SEND 方式发送，Redis 数据交互通过 IBV\_WR\_RDMA\_WRITE\_WITH\_IMM 发送，通过方法的不同来区分是控制面消息还是 Redis 的实际数据；

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/cc249835ab6c4846b8e11d8c34d24b42~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695653&x-orig-sign=V4rPIhx8MGWg3Nj1%2B5Lu9iGOAMA%3D)
4、客户端及服务端共享了一片内存，则需要对内存的使用管理，目前有三个变量用户协同读写双方的内存使用。

*   tx.offset 为 RDMA 发送侧已经对内存写入的偏移地址，从发送端角度看内存已经使用到了 tx.offset 位置，下次发送端再进行 RDMA 写入时，内存地址只能为 tx.offset + 1；
*   rx.offset 为 RDMA 接收侧已经收到的内存偏移地址，虽然数据可能实际上已经到了 tx.offset 的位置，但由于接收侧需要去处理 CQ 的事件，才能获取到当前数据的位置，rx.offset 是通过 IMM 中的立即数进行传递的，发送侧每次写入数据时，会将数据长度，所以 rx.offset <= tx.offset；
*   rx.pos 为接收方上层业务内存的偏移地址，rx.pos <= rx.offset。
*

5、当 rx.pos 等于 memory.len 时，说明接收侧内存已满，通过内存地址交换这个 RedisRdmaCmd 进行控制面交互，将 tx.offset、rx.offset、rx.pos 同时置零，重新对这片共享内存协同读写。

**Connection 各方法的主要实现逻辑及分析如下：**

**listen**：主要涉及 RDMA 编程图示中 listen、bind 的流程，结合 redis 的.init 相关调用流程，会将 cm\_channel 中的 fd 返回给网络框架 AE，当后续客户端连接该 fd 时，由 AE 进行事件回调，即后续的 accepHandler。

**accept\_handler**：该函数作为上述 listen fd 的事件回调函数，会处理客户端的连接事件，主要调用.accept 方法进行接收请求，并使用 acceptCommonHandler 调用后续的.set\_read\_handler 注册已连接的读事件，参见图 Redis Connection Call Graph。

**accept**：要涉及 RDMA 编程图示中 accept 的流程，处理 RDMA\_CM\_EVENT\_CONNECT\_REQUEST、RDMA\_CM\_EVENT\_ESTABLISHED 等 cm event，并进行 cm event 的 ack。

**set\_read\_handler**：设置连接可读事件的回调为.ae\_handler。

**read\_handler**：实际处理中会被设置为 readQueryFromClient。

**read**：从本地缓冲区中读取数据，该数据是客户端通过远程 DMA 能力写入。

**set\_write\_handler**：将 write\_handler 设置为回调处理函数，这里和 tcp、tls 实现的方式有所区别，并没有注册 connection 的可写事件回调，是因为 RDMA 中不会触发 POLLOUT（可写）事件，connection 的写由 ae\_handler 实现。

**write\_handler**：实际工作中被设置为 sendReplyToClient。

**write**：将 Redis 的数据拷贝到 RMDA 的本地缓冲区中，通过 ibv\_post\_send，这部分数据会通过远程 DMA 能力写入对端。

**has\_pending\_data**：检查内部的 pending\_list，在收到 RDMA\_CM\_EVENT\_DISCONNECTED 等事件时，会将当前 connection 加入到 pending\_list 中，由后续 beforeSleep 时调用 process\_pending\_data 进行处理。

**process\_pending\_data**：检查 pending 的 connection，并调用 read\_handler 读取 connection 中的数据。

**ae\_handler**：该方法有三个处理流程，第一是处理 RDMA CQ 事件，包括接收处理 RedisRdmaCmd 控制面消息，接收 RDMA IMM 类事件增加 rx.offset；第二是调用 read\_handler 和 write\_handler，这部分是与 tcp、tls 流程一致；第三是检查 rx.pos 和 rx.offset 的值，若 rx.pos == memory.len 时，发送内存地址交换这个 RedisRdmaCmd 控制面消息。
