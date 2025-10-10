https://juejin.cn/post/7559399994307772416  

快手DHPS 基于RDMA 通信的可负载均衡高性能服务架构

**一、项目背景**

# 一、项目背景

当前在线推理服务架构中，计算节点（推理服务）与存储节点（在线 PS 服务）之间存在海量的实时数据传输需求。随着模型参数量剧增，传统分布式架构需扩展到成千上万个服务节点，导致计算节点访问存储节点的带宽散出激增，进而推高访问延迟。加之当前主流的TCP网络通信存在CPU占用高、延迟高、吞吐低等劣势，严重制约了服务响应时间，限制了模型预估机器的横向扩展（Scale-Out）规模。

结合快手的业务需求，我们的目标是将传统分布式架构升级为高密计算存储分布式架构。通过RDMA通信构建计算节点与存储节点之间的高效互联体系，节省CPU算力，提高GPU算力密度，同时显著提升网络传输效率，为未来更大规模的AI基础设施建设奠定基础。为此，我们构建了国内第一个在在线系统中实现的可负载均衡的基于 RDMA 通信的高性能服务架构DHPS。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/6176e9462b4e4dfaacea71f913d15bc2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=x3QsGVh4aU5wIoA%2BFNa%2F46238BQ%3D)

# 二、技术实现

## ***2.1 整体架构***

DHPS架构通过端网协同设计，构建了覆盖计算、存储与网络的全链路高性能体系，实现了在线服务场景下RDMA技术的规模化落地与智能化调度，其架构创新可归纳为三大核心模块：

*   网络建设：构建了支持 AZ 级部署的四层网络，实现了超大规模 RDMA 与 TCP 混合运行能力。该架构支持业务进行AZ 级跨POD高性能通信，有效降低网络对 CPU资源的占用，提升业务部署灵活性，将业务部署范围扩展为AZ级。
*   软件优化：为充分发挥高密度机型的资源优势并显著提升系统吞吐性能，我们自主研发了高性能存储引擎和高性能 RDMA网络通信库 opt-rdma。
*   流量调度：基于对硬件和网络配置的感知，优先保证流量在同 POD 或同 AZ 内调度，以最大化利用 RDMA 的高吞吐与低延迟优势。同时，实现了用户无感知的 RDMA/TCP 协议自动选择和切换，通过实时采集 RDMA/TCP耗时和可用性数据，动态调整请求策略，并在检测到RDMA异常或拥塞时的自动回退。最终，实现了RDMA与TCP在AZ内的常态化混合流量均衡调度。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/212219d4a14043df9c51e834e45855d5~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=zZ4AtLtueHeJZ0A6oQts5pxB1zg%3D)

## ***2.2 高性能存储引擎研发***

为满足在线推理服务对支持高性能读取且需实时更新的特征向量（Embedding）存储需求，我们针对传统链式哈希方案存在的读路径冗长、过期管理效率低及内存碎片严重等问题，设计了新一代存储引擎。该引擎在保持原有接口兼容性的前提下，显著提升了读写性能，并有效降低了运维成本，其核心优化点包括：

*   索引优化：采用 12路Cuckoo Hash索引结构，并设计基于 8-bit Tag 的 SIMD 匹配算法，充分利用同一Cache Line内的数据，大幅减少哈希冲突，缩短读路径长度。
*   批量读取优化：针对批量读场景，积极预取以隐藏内存访问延迟，从而大幅提升读取吞吐量。
*   过期回收机制：设计了基 TTL分层的精确过期与强制回收方案，在确保高效回收过期数据的同时，将CPU开销控制在最低水平。
*   内存管理：采用Key-in-Value 存储布局，并定期执行内存整理（Compaction），有效减少内存碎片，确保内存浪费率不高于 5%。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b2ffdb561de64a568707f3ae208ae03f~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=%2F17dvU%2BDtHlJ%2BmoO19BxnUS97Xs%3D)

## ***2.3 RDMA 的高性能网络通信库***

过去十几年间，网卡带宽已实现从千兆到 800Gb 的跃迁。然而，在网络 I/O 密集型业务场景中，随着网卡性能的提升，操作系统处理网络 I/O 的开销也同步增大，这不仅推高了通信延迟，更严重制约了服务整体吞吐量的提升。其根本原因在于 CPU 算力的发展速度远滞后于网卡性能的提升。为突破这一瓶颈，亟需借助专用芯片来提升网络传输效率，减少 CPU 参与度。RDMA 技术应运而生。

##

RDMA是一种高性能网络数据传输技术，它允许计算机绕过操作系统内核和 CPU，直接通过网络适配器访问远程主机内存。其核心原理是由网卡硬件实现内存到内存的直接传输，彻底消除了传统网络通信中的性能瓶颈，具备高吞吐、低延迟、内核旁路以及近乎零 CPU 消耗等显著优势。

然而，RDMA 的应用也面临两大挑战：

*   编程复杂性：RDMA原生的 Verbs API 接口复杂、使用门槛高。
*   硬件依赖性：RDMA依赖底层硬件（如支持 RDMA 的网卡、交换机等）的支持。部署 RDMA 应用时，必须确保向后兼容性，以维护原有网络的稳定性。

为此，我们研发了一套基于 RDMA 的高性能网络通信组件。该组件旨在：

*   简化编程模型：封装并屏蔽底层复杂的 RDMA Verbs API，对外提供一套完整的网络传输解决方案。
*   保障兼容性与灵活性：同时兼容 TCP 协议，内部集成 RDMA 与 TCP 两套传输链路。在运行过程中，能够自动感知底层网络环境与硬件能力，动态选择最优的传输链路进行通信。

![]()
![]()
接下来，我们将从易用性、高性能和鲁棒性三个方面来介绍该通信组件。

高易用性：

*   统一协议：RDMA Verbs API 的复杂性主要源于其底层硬件交互机制和设计理念，开发者需要显式管理十几种资源队形，且异步通信模式与传统同步 Socket 编程差异显著。
*   提供类似 RPC 的接口：为契合业务系统广泛采用的 RPC 模式并实现用户无感知迁移，我们封装了底层连接管理、状态转换、内存管理、线程模型、流量控制等细节，对外提供与原有 RPC框架完全兼容的统一接口。
*   监控一体化：深度集成了 kess、rpcmonitor 等本地监控组件，真正做到了“拿来即用”的用户体验。

高性能

*   无锁机制：内部采用无锁、全异步设计，完全运行于用户态，彻底消除了上下文切换带来的性能开销；
*   Zero Copy：采用了单边通信模式，可以通过网卡直接访问对端内存，实现了零 CPU 参与、内核旁路以及数据的零拷贝；
*   QP 共享：通过链接资源池、注册内存资源池等预分配策略，实现资源的“即拿即用”以及高效复用。
*   原子操作：线程模型采用 Master-Worker 模式，Master 线程以 Polling 模式从完成队列获取请求，交由 Worker 线程处理业务逻辑，两类线程之间通过原子变量进行信息传递，从而达到超低延迟、超高吞吐，单机 QPS 轻松突破数千万。

鲁棒性

*   可靠性：采用 RC（Reliable Connection）通信模式，保证数据的不重、不丢、保序；
*   Fallback 机制：智能感知，可动态感知底层网络环境及硬件特性，自动选择采用 RDMA 通信还是 TCP 通信，自动选择最优通信路径；
*   协议兼容：TCP兼容，同时支持基于传统TCP的RPC通信模式，如遇 RDMA 硬件故障、数据错误等，可实现失败回退TCP；
*   硬件兼容：硬件兼容，支持硬件混跑，对不同代际网卡、不同品牌网卡做了兼容适配。

## ***2.4 流量调度与负载均衡***

快手的业务服务（计算服务和存储服务）采用多 AZ 部署，其中计算服务是无状态的，而存储服务则是有状态的多副本多 Shard 部署，我们能够保证在每个 AZ 内部都会至少有一个完整的存储服务副本。

在流量调度方面，我们在遵循以下优先级规则，智能动态调整计算节点到存储节点的流量比例来实现负载均衡：

*   最高优先级：优先网络 POD 内调度，优先 RDMA 通信
*   次优先级：其次AZ 内跨 POD 调度，优先 RDMA 通信
*   最低优先级：跨 AZ 调度，通过 TCP 通信

我们实现了故障检测机制，在 RDMA 通信过程中如果遇到硬件故障、连接异常、网络拥塞或其它原因导致 RDMA 通信失败的情况，可以做到快速切换 TCP，保障服务性能。

为保障服务高可用与性能稳定：

1、快速故障切换： 实现了高效的故障检测机制。当 RDMA 通信过程中遭遇硬件故障、连接异常、网络拥塞或其他导致通信失败的情况时，系统可快速、自动回退至 TCP 协议。

2、智能动态调节： 鉴于网络性能受多重因素影响，调度系统实时采集请求处理指标（如时延、成功率），并据此实施多级节点与网络选择策略：

*   动态流量配比：根据目标节点的实时可用性与请求延迟等参数，动态调整发往该节点的 RDMA 与 TCP 请求比例。
*   节点级熔断：当针对特定存储节点的 RDMA 和 TCP 请求均失败时，触发单点熔断机制，自动将流量切换至性能更优的其他可用数据节点。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b0bab6a6b8e34388a6b3e5fa0df6e236~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=uCOtFlz%2BeG5q4mALfLyQrMYcayM%3D)

## ***2.5 AZ级RDMA高性能网络***

基础设施打造了端网一体的高性能网络解决方案，网络侧通过 DCN5.0 全异构网络架构搭载自研 51.2T 网络交换机支持高密服务器 800G 双上联接入，在提供超大带宽的同时保证了网络接入的高可靠。主机侧自研拥塞控制算法和网络协议落地行业首个基于商业非定制化网卡的 AZ 级 RDMA 多协议混跑方案，克服了 DCN Lossy 网络的限制，实现了 PFC-Free，打破了过去 RDMA 传输局限于 POD 内的传输距离限制，拓展传输域实现 AZ 内 RDMA 网络的互联互通。对比行业基于大 buffer 交换机和自研定制网卡的昂贵方案，在大幅降低成本的同时提供了 RDMA 网络传输的大连通域、低延迟、高吞吐。结合流量亲和性调度，实现在数据中心内 RDMA 流量与 TCP 流量常态化混跑，并提升了其稳定性和效率。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b28c73f2f0464d0b95d5edd802767167~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=EZC9RWfS4MBCUJy5ZKrW2i%2Fpqs8%3D)
1、全异构高性能物理网络

*   超高带宽网络接入架构：
*   *   支持 DHPS 服务器 2x200G / 4x200G 双上连接入
    *   支持多种规模 POD ，按需灵性交付。
    *   支持多交换芯片异构组网。
*   AZ 级 RDMA 网络：
*   *   连通域：具备 AZ 级超大 RDMA 通信域，支持业务服务端、存储端灵活部署。长期演进到可覆盖 Region 级。
    *   吞吐：达到4层网络带宽的80%，在满足公平性的前提下充分利用各层级间网络有效带宽。在吞吐和延时的均衡条件下，达到最大利用率。
    *   丢包：利用自研拥塞控制算法等软件能力，将丢包控制在业务无感范围内。
    *   延时：POD 内的 P99 延时优于 DCQCN 30% 以上，POD 间的 P99 延时优于 TCP 30% 以上，在保持高性能前提下，不因时延抖动导致业务受损。
*   自研 51.2T 网络设备：
*   *   支持 400G 端口一分二 breakout 特性。
    *   KNOS 网络操作系统适配 marvel、huawei、brcm 51.2T 多元芯片。
    *   支持基于 IFA2.0 INT 、UCMP 等特性。
    *   交换机缓存双栈 QOS 设计，避免 TCP 网络抖动影响 RDMA 性能，在最大化 RDMA 性能的同时，保持 TCP 的链接稳定性。

2、自研主机网络协议栈

*   PCC 自研拥塞控制算法：
*   *   基于 RDMA RoceV2，快手自研基于 RTT + ECN + Tx\_event 精细化信号的 Rate - Based 结合 window 拥塞控制。
    *   算法支持 AZ 级 4 层超大网络域，支持 POD 间 Lossy/POD 内 lossless 网络，以及 TCP/RDMA 双栈混跑。
    *   基于 smartnic 可编程能力，实现多元化网卡基于统一拥塞控制算法的高性能、稳定互通。
*   PFC-Free：
*   *   在 DCN 范围，RDMA 稳定传输不依赖 PFC，摆脱对成本高昂的大 buffer 设备依赖。
    *   避免了数据中心 PFC 风暴对网络稳定性的冲击，造成大面积故障。
*   TCP/RDMA 双栈混跑：
*   *   核心模型使能 RDMA 高性能连接，长尾业务常态 TCP 连接覆盖。
    *   RDMA 网络可 fallback 到 TCP，异常情况下可保证高可用性。
*   多路径：
*   *   通过多 QP 路径支持网络端到端全链路的负载分担
    *   通过 Multi-path 技术实现流量 QP 级调度

3、联合业务高效高性能网络运营

*   RDMA 网络稳定性保障能力：
*   *   故障预防：建立端到端的配置管理、交付验收、巡检机制。在交付阶段使用KNP配置管理中心以及主机配置管理平台统一下发权威配置，保证初始配置的正确性；在交付阶段执行端到端的验收测试以及性能压测，提前发现并解决潜在问题；上线后定期巡检，及时发现并纠正配置误操作。
    *   故障发现：基于 Telemetry 和 gRPC 技术，秒级采集交换机与服务器网卡关键指标（RDMA 流量、交换机 Buffer 使用率、PFC、CNP、OOS 等）。在 RDMA 平面部署全量 pingmesh，秒级感知 RDMA 连通性故障问题。
    *   故障定位：构建RDMA POD 网络数据可视化系统，基于 RDMA 特点针对性开发了 ERSPAN  RDMA 丢包定位工具、PFC storm 故障自动定位系统、网卡毫秒级高精度指标采集工具 Probe ，整合多维度的数据，提供直观的 POD 全景运行状态视图，辅助快速定位。
    *   故障恢复：详细梳理 RDMA 故障场景及影响，制定清晰的故障处理 SOP（涵盖交换机与网卡）及应急预案。建立网络与业务团队的协同运营机制，实现故障快速联动止损与恢复。
*   RDMA 网络监控开放系统：
*   *   网络构建并开放主机监控数据接入、服务器网络信息查询、拓扑查询、网络变更、故障推送等系统服务能力，加强与业务团队的联合运营建设，实现故障的快速发现、定位和止损恢复。
*   高效交付平台：
*   *   联合业务建立了完整交付流程，开发了自动化部署、测试验收工具，依托于  KNP 大交付平台实现了自动化部署、端网一体化交付、自动化网络验收、业务联合验收。经过24年对流程和工具优化，达成：交换机网络交付验收（3BD），主机网络纳管交付（分钟级），业务联合性能验收（1BD）。

**三、性能收益**

# **三、性能收益**

DHPS架构通过端到端的技术革新，在多个维度实现了性能突破，为在线服务场景提供了量化可衡量的价值提升：查询吞吐提升 270%+，更新性能翻倍，内存碎片率下降 40%，网络延迟降低 35%，在超大规模集群中实现 99.999% 服务可用性，为企业级应用提供业界领先的高性能在线服务解决方案；在线 GPU 机器上面因为 CPU 节省可以带更多卡，能进一步从 4 卡机器升级到 8 卡机器甚至更多，提升大模型和搜推广模型结合的迭代上限。

以快手推荐大模型的精排服务为例，架构升级显著收益：

*   资源大幅缩减：老架构下存储服务节点需要 200 台左右64核 CPU 机器，新架构下只需要个位数的高密度 CPU 机器，机器成本资源节省 70%
*   延迟显著降低：计算节点与存储节点之间的通信延迟，由毫秒级降至百微秒级。
*   吞吐量大幅提升：如下表所示，通过采用新一代存储引擎 (Cubes - Cuckoo Hash) 与 自研高性能通信库 (opt-rdma)，极限吞吐提升超过 270%。

![]()![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/ef498bb8f0f94740b272471c6c8a5d94~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=H0YoBCQoMf7bLq3dcwXFrMQFIoU%3D)
Infer 收益（以推全的 million interest 服务为例，同等 qps 压力下）

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/9fdaaa99b75b40ba8cc01c5cdd937cbf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=LS36wc%2BSTUEEMx1VZPpwBnLlGrA%3D)
CPU 降低、延迟显著降低，为更高密度的 GPU 机器在在线服务中的应用创造了更大空间。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/6f39166a2f6c4af4bc98a048155e05f1~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=juZGt9akr9mnYdyiir%2Foh7iIXMQ%3D)
此外，DHPS架构优势在 TCP 和 RDMA 流量常态混合状态下稳定运行。在TCP与RDMA 混跑下，CPU 机器单机极限吞吐优于单 TCP 极限吞吐。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/1f89a1d756a940178e788190684678fa~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695732&x-orig-sign=9bpNxTVbtKiMyP1bpmBeA8njoEI%3D)

# **四、未来展望**

DHPS 作为国内首个在在线系统中实现的、基于 RDMA 通信的可负载均衡高性能服务架构，在满足快手在线系统严苛的高稳定性要求下，不仅实现了卓越性能（查询吞吐提升 270%），更显著提升了业务迭代能力上限，为大模型在搜索、广告、推荐等核心场景的落地奠定了坚实基础。

该架构的价值远超在线推荐场景。其 RDMA 自研通信库已作为核心组件集成至 KESS（快手统一的服务治理平台）。整套高性能基建设施（涵盖网络、存储、通信库）具备高度可复用性，可广泛应用于高性能计算 (HPC)、分布式存储系统、大规模模型推理服务等关键领域。这标志着搜索、广告、推荐（搜推广）领域的传统分布式架构，正在向面向 AI 大模型的高密度计算分布式架构演进。
