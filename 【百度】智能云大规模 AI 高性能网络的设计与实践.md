https://juejin.cn/post/7559399994307837952

【百度】智能云大规模 AI 高性能网络的设计与实践

## 1. 大模型训练对网络的要求

我们先来聊聊大模型训练对网络的需求。

最近半年以来大模型持续火爆。虽然关于大模型的发展与应用还有很多的争论，但可以肯定的是，大模型能力已经成为了接下来人工智能发展的基础。

和以前的小模型相比，大模型对大规模的分布式并行训练有更强的诉求。

这一方面是因为模型本身非常大。受制于今天的 GPU 显存限制，我们不得不把一个模型分拆到很多个 GPU 上来存储。比如说，百度的<https://qianfan.cloud.baidu.com/>[文心大模型](https://cloud.baidu.com/product/wenxinworkshop)有 2600 亿个参数，但是实际上一个 80G 显存的 A800，算上训练中间的计算状态，也只能存放大概 10 亿-20 亿参数。那显然光是存放 2600 亿的模型本身，就需要一两百块 GPU。这已经是一个比较大的规模了。

另一方面，因为训练更多的参数需要更多的计算量，因此我们必须得引入更大规模的 GPU 来进行加速，所以我们需要的 GPU 又要有一个数量级的提升。

在百度我们根据一个任务的 GPU 卡数来命名训练的规模。比如百卡以下我们叫小规模，百卡到千卡我们叫中规模，千卡以上我们叫大规模，超过万卡我们则以超大规模进行命名。依照这个命名方式，我们可以说，千卡以上的大规模并行训练是大模型成功的基础。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/af9b8b01b7244a718a7998118485b6d8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=mjmcKK%2BOy8dJUScolnELfGY4PQE%3D)
分布式并行训练有多种策略，我们这里列举出常用的三种。

*   应用最广泛的是数据并行。在数据并行里，每个 GPU 都拥有同样的模型副本，数据集则拆分成多份给到不同的 GPU 进行训练。每一轮迭代训练完成后，各个 GPU 需要把各自反向计算得到的梯度做全局同步，然后各个 GPU 计算出下一轮迭代用到的参数。在数据并行中，网络上需要对各个 GPU 上的梯度做一次 Allreduce，通信的数据量规模和模型参数规模成正比，对于千亿规模参数的大模型来说数据通信量都是很大的。
*   第二种并行策略就是流水线并行。[神经网络](https://cloud.baidu.com/product/wenxinworkshop)模型通常都是多层神经元的组合，包括大模型底层的 Transformer 模型也是这样。我们可以把模型按照神经元的层次进行拆分，不同层放到不同的 GPU 上去。这种并行策略需要不同 GPU 之间做层间点到点数据传递，传输的内容包括正向计算里的激活值和反向计算里的梯度值。这种通信在一个迭代里至少会发生几十次，但通信量一般不大，对网络的性能要求相对较低。
*   第三种并行策略就是张量并行，也就是联合多个 GPU 同时做一个张量计算，比如说矩阵乘法。这种并行策略需要多个 GPU 间对局部的张量计算结果做全局的 Allreduce 同步。这里的张量计算结果的大小，不仅和模型有关，也和训练使用的 batchsize 相关，通常都非常大，并且在一次迭代里会发生很多次这样的 Allreduce。因此张量并行对网络带宽的需求是最大的。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/95147db9cba942e98fd17b8fd71a5a66~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=QEHjQNyKv44XnuG6itDN4KV5s74%3D)
考虑到三种并行策略的特点，我们在训练大模型时，通常混合采用了三种并行策略。

首先在单机内部的多 GPU 卡间，我们采用张量并行，充分利用单机内部 NVLink 的高带宽特性。

其次，由于模型过大，单机 8 卡肯定还是放不下，因此在多机间继续采用流水线并行的策略，多路流水线并行构成一个模型训练的最小单元。

最后，为了进一步加速模型训练，我们再使用多路数据并行。注意这里数据并行的单元我们叫做一个 DP 组，每个 DP 组内部都是张量并行和流水线并行共存。

数据并行中的 Allreduce 实际上是每个 DP 组的同号卡之间进行的。比如这个图里，8 路张量并行，4 路流水线并行，3 路数据并行。在数据并行里，实际上有 32 个 Allreduce 组，每个组里有 3 张 GPU 做梯度同步。

数据并行里每张 GPU 卡都需要对 10GB 级别的数据做 Allreduce，这个 Allreduce 是大模型训练对网络的主要需求。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4ef7b41aac6944bea7c7775e9f348203~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=XyTDBeE2ZJrBsoUA7%2FkOZ3ldwGU%3D)
![]()
![]()
正是由于大模型训练里这些需求，我们提出了 AI 高性能网络的三大目标：超大规模、超高带宽以及超长稳定。

首先，规模的大小直接决定了模型训练的快慢。这张图里可以看到，对于一个 1750 亿的模型，如果采用 2 千张 GPU，仍然需要训练 100 天以上。采用 8 千卡则可以把时间压缩到 30 天左右。这对于今天快速迭代的大模型而言是非常重要的。

其次，Allreduce 的带宽直接决定了大规模分布式下的整体效率。我们可以看下面这个图，平均单 GPU 的 Allreduce 带宽有 5GB/s 的时候，大规模分布式的整体加速比只有 70%。想要获得 90% 的加速比，单 GPU 的 AllReduce 带宽则需要做到 20GB/s，相当于单 GPU 跑满 400G 网卡。

最后是稳定的问题，由于训练时长至少是几个星期，长时间下的稳定性显得非常重要。这个图里我们以 GPU 可用性为例来做个简化讨论，假定单 GPU 的月可用性是 99.9%，那么在千卡规模下模型训练一月内遇到故障发生中断的概率是 60%，而如果采用 8 千卡中断概率就有 99%。即使 GPU 的可用性提升到 99.99%，8 千卡下的中断概率仍然在 50% 左右。

我们这里以 GPU 可用性举例，是因为大模型训练中碰到的主要可用性问题来自于 GPU。当然网络必须保证更高的可用性，才能尽可能减少模型的训练中断，降低模型做 checkpoint 的频率以及开销。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/69222e04fa3543b7b3e8240920f71429~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=uNVXpg2hQuvXuglwe6K5WyMRcWw%3D)
![]()
![]()

## 2. AIPod 高性能网络设计

基于这样的目标，我们有针对性地设计了 AI 大底座里面的 AI 高性能网络—— AIPod。

这张图是关于 AIPod 高性能网络的点线图。注意这是一张完全图，我们把每一个网卡、每一个交换机、每一条线缆都画了出来，充分展现出这张网络的复杂性。这张网络里面有约 400 台交换机、3000 张网卡、10000 根线缆和 20000 个光模块。其中仅线缆的总长度就相当于北京到青岛的距离。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/f97c3f69e6dc4f1f88f0a21203cf6e26~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=xo9G532L%2FlZyYd1RUBC%2B8wqa%2FOI%3D)
![]()
![]()
当然上一个图只是一个感性的认识，接下来我们要聊聊理性的设计。

为了支撑超大规模的这张 AIPod 网络，我们选择了 3 层无收敛的 CLOS 组网结构。所谓的 CLOS 组网就跟这张图展示的一样，服务器在最下面，连接到 Leaf 层交换机，也就是图里的 LF，然后 Leaf 交换再通过 Spine 交换机连接起来，就是图里的 SP。最后 Spine 交换机再通过 SuperSpine，也就是 SSP 互联起来。

我们前面说到，在大模型训练的时候，主要的通信来自于同号 GPU 卡之间，也就是一台服务器的 1 号卡和另一台服务器的 1 号卡之间通信，一台服务器的 2 号卡和另一台服务器的 2 号卡之间通信，以此类推。较少的情况下才会发生跨卡号通信，比方说一台服务器的 1 号卡和另一台服务器的 2 号卡通信。所以，AIPod 网络在设计的时候，采用了 8 通道的架构。每个服务器上的 8 个网口，对应 8 个 GPU，分别连接 8 个不同的 Leaf 交换机。这 8 个 Leaf 交换机一组，构成了一个汇聚组 Group。这样的一个汇聚组下最大可以有 512 张 GPU。进一步，8 个 Leaf 交换机再往上连入不同的 8 个通道，每个通道内 Spine 交换机和 Leaf 交换机之间做 fullmesh 全互联。这样的一个集群最大可以支持超过 16K GPU。

虽然主要的通信发生在同一个通道内，但总还是会存在跨通道的通信。所以我们通过 SuperSpine 再把不同的通道的 Spine 交换机连接起来，打通各个通道。这就是 AIPod 的组网方式。AIPod 的网络我们采用了无收敛，或者说收敛比为 1:1 的方案，也就是指交换机的上联带宽等于下联带宽，确保集群内互通带宽充足。

为了尽可能支撑更大的规模，我们在选择交换机的时候，会选用当前顶级容量的交换芯片，比如曾经的 12.8T 或者 25.6T 芯片，那么现在已经演进到了单芯片 51.2T 的交换机。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/eae4c3e4ee804072b2945f0e157dd3fd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=jYFk%2F%2BXJsjzhdPXUJi0hACuuIRA%3D)
![]()

上面我们主要讲了大规模的 AIPod 怎么构建。接下来我们谈谈带宽的问题。

百度智能云选择单机最大 8x400G 的接入规格，网络上选择了无收敛比的 CLOS 架构，同时也支持 RDMA 和 GDR，理论上带宽可以做到很高的水平。但是实际上，当规模一大，就会产生很多的问题。其中一个最主要的问题，就是跨交换机的选路冲突。

从技术上说，几乎所有的网络传输都有一个固有的问题，就是同一条连接在网络内要避免乱序，因为一旦发生乱序，在接收端就会触发重传逻辑导致降速。所以网络内交换机转发报文的时候，会把同一条连接的报文会向一条路径转发，而这条路径的选择就依赖哈希算法。

我们知道哈希算法总是会有冲突的，像是左边这个图里展示的，2 个跨交换机的连接如果同时选择了左边这个链路，那么就会导致左边链路拥挤而右边的链路空闲，这样两个连接的带宽都会减半。这种问题在大规模训练中太常见了。

为了缓解这类问题的影响，我们通常会在 NCCL 通信库里面设置两个 GPU 间采用多个连接的方式，比方说右边这个图，连接数一多，出现严重不均衡的概率也就越小。这种方法可以增加网络内的路由熵，减小哈希选路冲突所带来的影响，但是问题并没有彻底解决。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/f5d7a5159b794249b16f08fabb122c61~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=UszMnG6NhY4BTYdulv66Z3RexqU%3D)
![]()

我们可以看到，这类问题只发生在跨交换机通信的场景。所以为了进一步减小这个问题的影响，我们应该尽可能让通信发生在一个交换机内。而同一个汇聚组内的同号 GPU 通信，是不会跨交换机的，也就没有哈希选路冲突问题。这也是为什么我们尽可能扩大一个汇聚组规模的原因。

为了减少跨交换机的通信，在 AIPod 里我们提供了网络架构感知的方法。网络架构感知，就是允许上层感知到当前 GPU 在网络架构的什么位置，归属于哪一个汇聚组，它的 GroupID 是多少。

AIPod 可以把这个信息透给上层的任务调度系统，让训练任务调度的时候，把同一个任务尽可能调度在同一个汇聚组下，这样通信肯定也就只在一个汇聚组内了。

但是大模型的任务通常很大，不可能全在一个汇聚组下。这时我们就需要通过汇聚组信息对全局 GPU 做有序化处理，让通信库在构建 Allreduce 拓扑图的时候，减少跨交换机的互通流量。我们用右下角这个图来说明，4 个 GPU 做 AllReduce，采用 ring 算法，两种不同的 ring 的构建顺序，会使得跨交换机的带宽有很大的差别。显然左边的这种建 ring 的方法是更高效的，而右边则是低效的。这就是 AIPod 里面网络架构感知所带来的收益。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/55902757020e4586bdd942f6b0d545a2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=xsPBZ%2FiOAZmUcPvKDuIUyQ%2Bb8m8%3D)
网络架构感知可以明显缓解跨交换机通信的数量，从而减少哈希选路冲突的影响。但是问题并没有彻底解决，冲突仍然存在。

如果想彻底解决这个问题，就需要用到网络的多路径转发能力。也就是允许报文的乱序接收，从而打破单连接报文只能选择一个路径这种假设。Infiniband 网络里面引入了这种多路径转发能力，叫做自适应路由。而在 AIPod 里面，我们依托百度的自研交换机，在以太网上通过动态[负载均衡](https://cloud.baidu.com/product/blb.html)DLB 技术也实现了类似的能力。

就以下图为例，首先在发送端，网卡要通过标记允许报文被乱序处理，而在交换机上会根据当前各个路径上的队列深度、利用率等信息，计算出每个报文当前的最佳路径进行转发。这样肯定会引入同连接报文的乱序问题，所以需要在接收端对乱序的报文做重排处理。

经过这一系列的组合机制，就可以彻底解决跨交换机通信的哈希选路冲突问题。我们相信这种底层的技术能力增强，才是解决大规模训练的终极方案。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/ea470843eaab40ac97cc8625858083cf~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=RYJj5PMNPE5S9wK09OJWpUTQT0Y%3D)
![]()

接下来我们再来聊聊稳定性相关的问题。

保持任务长时间不中断对于大模型训练是很重要的，但是硬件终究会有故障。举个例子，对于一个可以承载 16000 卡的集群而言，里面会有将近 10 万个光模块。考虑到真实硬件的质量水平，假定一个模块的 MTBF 是 1 千万小时。这里 MTBF 指的是一个硬件设备在故障前的平均使用时长。因为模块基数太大，哪怕是 1000 万小时的 MTBF，也会导致平均下来 4 天左右就会发生一个故障发生。在这么大的基数下，单体的小概率事件也会演变成总体的大概率事件。

所以，AIPod 网络着重构建快速从硬件故障中恢复的能力。比如网络里面的某条链路发生了故障，通过这个链路的报文都会发生丢包。我们必须让这个丢包的时长低于通信库通常设置的超时时间，这样任务才不会中断。

在 AIPod 里面，对于上行方向的这一类丢包，通过动态负载均衡技术，可以在毫秒级的时间尺度上面得到修复，也就是选择新的可用链路。而对于下行方向的这一类丢包，需要触发网络内的路由更新和路由收敛，通过优化路由更新策略以及路由下发效率，来把下行方向的丢包时长控制在秒级水平。这样我们就可以做到网络内硬件故障对训练业务基本透明。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/00f941c51de44aabbeefaf898b88857b~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=VL0Meh3t%2FJM57%2BJd3SBwJNS9wPg%3D)
![]()

当然网络的复杂度在于，有一些硬件故障是不能被显式直接感知到的。比如某些交换机芯片缺陷所带来的比特翻转问题，会导致报文在传输过程中某个比特位被修改从而丢包。这种问题不像设备故障那样可以被直接发现，非常难以排查。

为了充分发现这些隐藏的问题，我们基于百度自研交换机设计了 AIPod 网络的黑盒探测机制，确保每条链路上每秒都有黑盒探测报文。探测一旦发现连通性问题就会触发自动的定位和隔离机制，把问题链路、问题设备隔离掉，并且触发告警，让运维人员快速介入进行修复。

AIPod 里面的黑盒探测机制，是保障各种网络问题被第一时间感知到的关键。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/160c516629d741fbb3cedcdf9417d432~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=UEeSbPon%2FZUci2UOVhfPj%2BzHvCI%3D)
![]()

对于 AIPod 高性能网络而言，除了连通性故障以外，还有一类故障与连通性无关，而是与无损网络有关。

AIPod 高性能网络是无损网络，也就是说网络内不会产生拥塞丢包。这是依赖网络内的 PFC 技术实现的。但是无损网络也可能会有异常。

常见的一个异常是 PFC 死锁，这个技术上比较复杂，原理我们就不展开了，从效果上看会导致网络永久性挂死。因此我们在网络内一定会启用死锁检测。但问题在于，来自于交换芯片的死锁检测总是有假阳性判断的存在，从而导致瞬时误丢包。第二个常见异常通常是来自于芯片故障，尤其是网卡芯片故障所带来的持续性 PFC 风暴，这会导致集群整体的传输效率下降。

这两类与无损网络相关的异常是固有问题，我们目前还不能完全消除。但是，通过基于百度自研交换机的 Telemetry 遥测技术，我们搭建了无损网络的性能透视平台，确保网络内的任一丢包信息和 PFC、缓存的异常变化都能被迅速感知到。通过这样的性能透视平台，这类问题可以被第一时间发现并解决掉，最终并不会影响到大模型的稳定训练。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/0973ff7de3ea4cd79d17de98ad3a6324~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=7ACwYiszqoZwE17GkCrvkApj8A0%3D)
![]()

讲完了 AIPod 的大规模、高带宽和长稳定设的计，接下来作为一个番外篇，我们聊聊超低延迟的 AIPod 网络。

之所以把这个内容放到番外篇来讲，是因为对于大模型训练而言，延迟并不是一个核心考虑，带宽才是核心。只要能做到微秒级别的网络延迟，在网络上进一步降低几个微秒的延迟在端到端的性能上几乎看不出任何的影响。但是从技术上说，仍然存在一些特定的 AI 业务是延迟敏感的。AIPod 网络也一样能满足低延迟的诉求。

我们先来拆解一下网络延迟的构成，除了总线、网卡、光模块、交换机的硬件延迟以外，我们真正有机会优化的只有两个延迟，一个是光纤延迟，另一个是交换机排队时延。

光纤时延之所以重要，是因为光速是有限的。光在玻璃里的传输速度是每秒 20 万公里，换句话说，两个 GPU 间每增加 200m 光纤，就会增加 1 微秒的时延。为了让集群内任意两个 GPU 之间的延迟尽可能低，AIPod 里我们优化了整个集群在机房内的布局，使得服务器和交换机以及交换机和交换机之间的距离尽可能近，从而能够采用更短的光纤连接。

另一方面，交换机排队延迟与交换机的缓存占用有关。一个交换机端口上每有 1MB 的缓存，就会导致 25us 延迟的增加。AIPod 内部我们优化了的拥塞控制参数，尽可能保证在不影响带宽的前提下，降低交换机的缓存占用，以此实现更低的延迟。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/cd0b0c186afb4b2fb264d5b93092c0a0~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=EpNwJtipN89CNhhhCX9HgpozRNE%3D)
![]()

大规模、高带宽、长稳定、低延迟，我们关于 AIPod 高性能网络的设计要点基本讲完了。接下来再看一个番外篇。

刚才讲的 AIPod 网络主要指的是连接 GPU 的训练网络。但其实存储的读写性能也对大模型的训练非常重要，这里面就包括了数据集的读取以及训练过程中 checkpoint 的读写。

存储主要依赖的还是云上的 VPC 网络，百度智能云在 VPC 网络上也做了相应的高性能支持。比如如果要访问并行文件系统 PFS，我们有高性能的弹性 RDMA 技术进行支撑，单客户端能够做到 200G 的带宽。

而如果要访问普通的[文件存储](https://cloud.baidu.com/product/pfs.html)CFS 或者是[对象存储](https://cloud.baidu.com/product/bos.html)BOS，也可以通过挂载我们的高性能硬件负载均衡实例，单客户端获得超过 10G 的稳定带宽。

存储底层的这些高性能技术的使用，对于大模型的计算效率也有很大的贡献。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/e7f3fa7bd7b841bea5f4171aa91e752e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=GsJ0%2B5YAsoXV%2BnfFa8ZQVfh0kU4%3D)
![]()
![]()

## 3. AIPod 大模型训练实践

以上就是我们关于 AIPod 高性能网络设计的分享。接下来我们再看看百度基于 AIPod 网络所做的一些大模型训练的实践。

这里展示的是在百度百舸平台上的两个千卡规模的训练流量图，分别来自于 RoCE 集群和 IB 集群。

两个集群上的模型稍有不同，但都达到了单卡百 G 水平的通信带宽并且可以长期稳定运行。

搞过大模型训练的同学应该都有切身感受，真正把千卡任务跑起来并且顺利的跑完并不是一件容易的事情。过程中会遇到很多很多的问题，我们需要一些辅助工具来帮助我们快速判断问题的源头是什么。

这里我想着重讲两个工具，一个是任务可视化工具，一个是故障诊断工具。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a5526946c4a845abbbf0b211bcb26396~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=cdhhfnPURbYFNlEhlKSPmvDAqcU%3D)
![]()

任务维度的高精度可视化工具对于判断任务是否正常训练非常的重要。这里有两个重点，一个是任务维度，另一个是高精度。

所谓任务维度，指的是我们要能够把归属于一个任务的几百上千个实例的监控数据合并到一起来看。因为并行训练的特点，这些实例理论上展现出来的网络流量信息是高度一致的。一旦有不一致的存在，我们就很容易发现异常。

所谓高精度，是指我们的监控采集必须是秒级甚至是亚秒级的。我们前面提到，一次迭代的 Allreduce 通信量是 10GB 级别的，听上去很大，但是放到一个 100G 网卡上也不过只需要 2 秒的时间就传完了，如果是 400G 网卡只需要半秒的时间。所以，如果还是采用传统的 10 秒采集甚至是分钟级采集，那么我们对网络流量的观测一定是扭曲的，会干扰我们对于任务状态的判断。比如上面这个图展示的就是早期我们做的 10 秒采集，看到的峰值带宽只有 20G，这显然是不对的。等到我们做了 1 秒采集后，就可以精确的看到脉冲式的流量图，流量峰值也达到了 100G。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/b3b0cbadffa24a5a8eed62ff0763695e~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=GuZGWnEAfOYiI9mVqqkhPW4rEdU%3D)
第二个关键工具是故障诊断工具。

大模型训练经常会碰到各种各样的异常，很多异常会表现为通信库通信异常。但是这些异常的根本原因多数情况下并不是网络问题。举个例子，左边这个是我们一个客户的 case，在框架[日志](https://cloud.baidu.com/product/bls.html)里面看到了 NCCL 相关的 Timeout。但是该 case 的真实原因其实是 GPU 的故障。针对这类问题，我们需要故障的一键定位，从各个节点的相关日志里提取信息并且自动分析，让故障的根因更快的被找到。

有些软硬件的问题并不一定导致任务完全卡死，而是会有慢节点的存在。大模型的训练会因为一个慢节点而拖累全局速度，也就是一点慢，全局慢，所以有的时候并不能直接通过任务维度的可视化工具找到问题所在。针对这个问题，我们也研发了对应的慢节点搜索工具，能够在一个集群内自动以二分的方式测试集合通信带宽，从而找到或者排除掉慢节点的存在。这对于真实问题的定位也是有很大帮助的。

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/c6ea4d0199ef431792b72276b9f9dbe4~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695913&x-orig-sign=j1Id0QaFvPNJ%2BGTolngToPpvao0%3D)
![]()

以上就是我今天想跟大家分享的全部内容了。AIPod 高性能网络是百度智能云 AI 大底座中百度百舸的底层关键技术，决定了大模型训练的能力和效率。大规模、高带宽、长稳定的 AIPod 高性能网络能够帮助用户更高效率、更低成本的训练自己的大模型，在这个大模型的时代里面保持持续领先。

**Q\&A**

Q：支持乱序+多路径，底层的网络协议还是用的原生 RoCE 吗？乱序组包是由硬件还是软件实现？对性能有多大的影响？

A：用的是 NV 的新版本的 RoCE 实现，乱序组包使用网卡硬件来支持的，乱序重组仍然可以达到网卡限速。

Q：VPC 也支持 RDMA 么？

A：百度智能云提供了弹性 RDMA 产品 ERI，可以支持 VPC 上 200G 带宽、5us 延迟的 RDMA 通信。

Q：如果整个 AIPod 是无收敛的，那对交换机的数量或者交换机光口的数量要求会比较多，百度在部署成本上是怎么考虑的？

A：百度智能云考虑的是整个集群端到端的性价比。网络设备成本在集群整体成本里面占相对较小的比例，通过提供一定的冗余，所带来的性能增益是高于成本提升的。
