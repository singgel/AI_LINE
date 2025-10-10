https://juejin.cn/post/7559280369797169192

[SIGCOMM'25] Revisiting RDMA Reliability for Lossy Fabrics

<https://mp.weixin.qq.com/s/_4bhHZj8vigXn1mfM7abkQ>

# **一、背景：算力激增驱动智算网络规模不断增大，现有传输技术面临挑战**

AI大模型快速发展，算力需求急速攀升，驱动集群网络组网规模不断扩大，通信距离也不断拉远。单一集群需要园区内多栋楼部署，同时受到部署策略、走线等物理因素限制，最大通信距离可达到2km-10km；如果要规划更高的算力规模，供电、散热等能源问题会成为瓶颈，需要多集群联合训练，跨AZ场景最大通信距离可达到百公里。

当前智算网络大多沿用已有数据中心技术，主要的技术路线是基于PFC流控的无损RDMA网络。但随着组网规模的进一步增大，PFC带来的头阻、死锁、运维等问题会更凸显，严重影响网络性能。另外，在交换机交换容量增大、交换芯片Buffer增长速度滞后等趋势下，该路线将会面临Buffer不足的问题。与此同时，业界也一直在探索高效的有损RDMA路线，例如在RDMA网卡(RNIC)中实现选择性重传机制。然而这条路线仍然面临ECMP冲突、RTO超时等问题，并且对多路径、逐包均衡等技术兼容性不好。

**针对上述问题，文章提出了DCP（Data Control Partitioning）数控分离技术，重构了高速有损网络的RDMA可靠性设计，推动智算网络向容损、逐包均衡等方向演进。** 该方案对控制信息和数据信息采用不同传输策略，对数据信息允许有损传输，对控制信息采用无损传输，可以大大降低对Buffer的依赖，彻底消除PFC带来的头阻、死锁等问题，同时兼容多路径传输、逐包均衡等技术，支持百万卡规模、百公里等大规模、长距离、高性能网络传输的需求。

# **二、DCP设计思路**

DCP是一种联合设计交换机和RNIC的传输架构，包含DCP-Switch和DCP-RNIC。DCP概念上定义了数据平面（DP）用于有效载荷传输和控制平面（CP）用于报文头部传输。与无损RDMA网络通过PFC同时保证DP和CP的无损性不同，DCP-Switch引入Packet Trimming功能，每当网络出现丢包时，会把丢失报文的头部封装成Header-Only（HO）报文传输给接收端；DCP-Switch使用加权轮询（WRR）调度器来优先处理控制队列，从而确保控制平面（CP）传输的无损性，同时允许数据平面（DP）以有损方式运行。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/24aaf554cdc34e2499a3f95060048a69~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695884&x-orig-sign=XPMGI4DmJgPt8%2F4UCaqpQWIx%2BmI%3D)
同时，DCP-RNIC利用无损控制平面的特性来增强RNIC的可靠性，实现了以下几项关键功能：

*   Precise and Fast HO-based Retransmission：发送方根据HO包携带的PSN精确并高效地重传丢失的包；
*   Order-tolerant Packet Reception：接收端RNIC可以直接将任何包（无论是有序还是乱序）写入其相应的应用程序内存地址，消除了对重排序缓冲区的需求；
*   Bitmap-free Packet Tracking：DCP-RNIC利用无损CP的“Exactly Once”特性，消除了包级别bitmap的需求，采用包计数来跟踪聚合的消息级信息，显著减少了内存开销和处理周期。

# **三、实验效果**

文章针对DCP进行了全面的技术验证，主要包括两部分：1）原型样机测试（含DCP-Swtich和DCP-RNIC）；2）大规模仿真实验。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b01323f9f8484ab79d2256c5a7ccc6cb~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695884&x-orig-sign=ZRL7LS78s2aTdy9Y0LZ4%2F0MIWW4%3D)
**原型样机测试结果：** 组网拓扑如上图所示，DCP传输技术与逐包负载均衡原生适配，相较于Mellanox RNIC，DCP在丢包恢复效率上提高了1.6×∼72×，在AI工作负载的完成时间上降低了42%；相较于IRN和MP-RDMA，DCP在通用负载测试上分别取得了2.1×和1.6×的性能提升。此外， DCP在10公里长距测试下实现了接近理想的高吞吐，DCP理论上可实现百公里高性能传输。

![]()
![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/7cbb7ee8da0848bdb4a52ca75c96b4ce~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695884&x-orig-sign=6%2Ffn3GG%2FdXvAP%2BKL3qIyOMAtA94%3D)
**仿真实验结果：** 组网拓扑如上图所示，DCP传输技术相较于MP-RDMA和IRN（业界SOTA的lossless和lossy传输解决方案），在智算流量场景（如AllReduce）下，平均降低了38%和45%的任务完成时间JCT（如下图a所示）；在通算流量场景下，分别降低了16%和10%的P95尾部流完成时间FCT。此外，在1000公里长距大规模实验中，相较于MP-RDMA和IRN方案，DCP分别降低了95%和51%的P95尾部完成时间（如下图d所示）。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e8ebe52d522f46ee88ad2392a39d09ff~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695884&x-orig-sign=k11ciEBdcKil5TeOQsB%2BON%2FXAAs%3D)
![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0e70cd6a4f81451186d09d1554e23b71~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695884&x-orig-sign=JFxgXb2KVOk8wxjbalsDr4l4OOM%3D)

# **四、总结**

华为网络技术实验室提出的DCP技术，是一种面向有损网络的高性能RDMA传输架构，通过将轻量级无损控制平面与硬件高效的RNIC设计相结合，消除了对PFC的依赖，支持包级负载均衡，并避免了RTO。原型和仿真表明，DCP 的性能显著优于现有的 RDMA 解决方案，有利于推进高性能 RDMA 传输技术在有损网络中的应用。

经了解，华为网络技术实验室在研究面向AI原生的传输技术AI-Native Transport（ANT），通过逐包均衡/多路径、算效优先调度、容损传输等技术，为AI智算网络提供高吞吐、高算效、高可扩展的传输能力，本次SIGCOMM文章的DCP技术是ANT若干特性之一。
