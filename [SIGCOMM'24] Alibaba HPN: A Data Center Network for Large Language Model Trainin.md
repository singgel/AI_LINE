https://juejin.cn/post/7559280369797152808

[SIGCOMM'24] Alibaba HPN: A Data Center Network for Large Language Model Training

<https://zhuanlan.zhihu.com/p/719350246>

阿里云这篇针对LLM设计的[HPN](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=HPN\&zhida_source=entity)paper真是干货满满，读起来也非常的丝滑。大语言模型（LLM）和传统云计算因流量特点以及容错需求上的差异，导致传统的数据中心网络架构并不适合LLM训练，所以阿里云针对LLM训练场景设计了一套新的基于[RoCEv2](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=RoCEv2\&zhida_source=entity)的组网架构，总结起来就是去堆叠双上联 + 双网络平面 + 大二层设计，使单Pod可以支持15K GPU，跨Pod可以支持100K GPU。

# 1、问题描述

LLM模型规模一直在快速增长，很多都超过了千亿（100B）参数规模，比如GPT3的参数规模为175B，GPT4的参数规模是1800B，百度“文心”为260B，等等，如图1所示。巨大的参数规模意味着需要大量的GPU做训练，以及意味着需要更大规模的高性能网络集群。LLM训练又因为流量模式和对故障容忍的高要求，导致传统针对通用云计算服务搭建的数据中心网络并不适用。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/3226a5b27e904a33a03fa4aaa7f88c17~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=LUABBuQV8bVJUqU2DMdGh%2BwBi2I%3D)

## 1.1 流量load imbalance问题

LLM训练的流量特点总结起来就是，1）周期性；2）连接数少；3）突发性。

一般云计算（VPC网络）连接数10K，100K都是有可能的，每个流都是连续的，带宽利用率低（例如，通常低于 NIC 容量的 20%），流量模式总体上相对连续和稳定，以小时为粒度缓慢变化，如图2所示。而LLM 训练产生的流很少，几十条流或者上百条流，但会周期性地突发，持续几秒或者几十秒，突发流量可以直接打到 NIC 带宽上限，例如打到 400 Gbps，如图 3 所示。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/95ca51be444f4a4090d542c3d59c810a~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=HV%2BQsK9rqkteuXqnfL8t1CKqKxs%3D)
传统数据中心网络采用[ECMP](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=ECMP\&zhida_source=entity)作为load balance方案。ECMP策略有效工作的前提是哈希算法可以有效地将流量均匀分发在所有等效路径上，此假设在一般的云计算场景下成立，通常会产生数百万个连接（如图 2所示）。然而，在只有少量大流量（也称为大象流）的 LLM 训练场景，这种假设不再有效。paper中指出，在使用传统数据中心进行 LLM 训练的实践中，遇到了由于哈希极化而导致的多个性能问题，严重影响了 LLM 训练效率。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/d8873621526e442382d971945cb52366~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=5u5N6N0LePpw8zp7hLZW9JZTI%2BQ%3D)
多八卦下，load balance一直是数据中心网络的一个研究热点，特别在[RDMA](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=RDMA\&zhida_source=entity)网络，由于硬件片上资源有限导致支持的QP连接数也有限，更容易造成ECMP hash不均。解法方法可以分别从端侧和网络侧思考，端侧比如flowlet，还有24年sigcomm上腾讯Turbo提出的DCT，还有很多啦，网络侧方向，比如DDC交换机。

## 1.2 LLM训练对故障高度敏感

LLM 训练是一个同步过程，对故障更敏感。在 LLM 训练中，多个 GPU 协作完成每次迭代，需要多次迭代（持续数十天）才能完成整个训练过程。因此，任何 GPU 或主机发生故障都可能直接减慢当前迭代速度，甚至导致整个 LLM 训练过程崩溃。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/95eee153ab3b484cac1166fe94955727~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=QyJB8SolAQeNQZXCIfaBYMWpqJE%3D)
paper中指出，对阿里 LLM 训练影响最大的是与 Top-of-Rack (ToR) 相关的单点故障，这种故障会影响到各种 GPU。tier2和tier3层具有丰富的冗余链路，但每个NIC都通过一条链路连接到ToR，存在单点故障风险。当访问链路发生故障时，会导致相应主机连接断开。LLM训练需要数千个GPU协同训练，涉及数十个ToR和数千个光模块和链路。在如此大的规模下，很难保证没有网络设备发生故障。监控和故障排除系统等工具可以被动地定位故障的根本原因，但无法防止训练崩溃。根据阿里云的统计数据，每月有0.057%的NIC-ToR链路发生故障，约0.051%的ToR交换机遇到严重错误和崩溃。在如此高的故障率下，单个 LLM 训练job每月会遇到 1-2 次崩溃。此外，每天会发生 5K-60K 的链路抖动情况，也会导致暂时的性能下降。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/bbe0e2929018476f96b4f0c70f37cfc8~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=mmvEI5CnLG5ZAEHABsCMbSBEpxQ%3D)
此外，LLM 训练中的故障代价高昂。发生故障时，LLM 训练通常会利用checkpoint从故障中恢复；但是，由于在 LLM 训练中生成checkpoint需要大量存储空间（例如，每个 GPU 30GB）和高开销（例如，100 秒），因此通常选择每隔几个小时才生成一次检查点。例如，在图 5 中，记录了四个代表性生产 LLM 中的checkpoint生成间隔，这些间隔通常在两到四个小时之间（即使在这些高间隔下，检查点带来的开销仍然在 5% 左右）。这意味着一旦发生故障，整个训练必须回滚到几个小时前并重新训练。

# 2、解决方法

## 2.1 去堆叠双上联

去堆叠双上联在国内并不是什么新创意，很多互联网公司都在用，paper中花了很大篇幅介绍单上联->堆叠双上联->去堆叠双上联的优缺点。网上有很多资料，这里不做过多介绍，感兴趣的朋友可以参看这篇博客[堆叠与去堆叠](https://link.zhihu.com/?target=https%3A//cloud.tencent.com/developer/article/2076426)。

## 2.2 双网络平面

由于传统数据中心网络的三层架构特性，大流量的转发需要经过三遍哈希运算（即 ToR、聚合和核心层）。由于每次哈希运算的输入（即流量的五元组）保持不变，这种“级联”哈希运算的效果可能会导致更严重的负载不平衡。而去堆叠双上联并不能解决hash极化问题，如图6（a）所示，在server1上发出去的流量经过聚合交换机最后都hash到了ToR3，然后从server2的NIC的某个port出去，导致server2 NIC的两个端口负载极度不均衡，影响训练性能。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/a41ca39336034a2ab01bb9acdfca8743~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=SQg2Lq%2BCvhzNgudVNGiQs%2B3ypEk%3D)
引入双平面可以有效解决hash不均的问题，双平面就是双上联ToR1和ToR2的流量是隔离的，如图6（b）所示，ToR1的流量只会走Agg1 -> ToR3这条路径，ToR2的流量只会走Agg2 -> ToR4这条路径，只要保证server1到双上联的流量是hash均衡的行。也就是在LLM训练场景下，因为连接数不多，且多是大象流，所以尽量减少hash次数，如图7所示，[Clos架构](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=Clos%E6%9E%B6%E6%9E%84\&zhida_source=entity)和双平面架构ToR下联口队列长度的比较，可以明显看到采用双平面架构的端口队列长度更均衡。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/42bec33a5f4e4a3196be3b07bb9aa2e2~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=wUS7b7PViT%2BYCaZM6HQzk4Egro0%3D)

## 2.3 组网分析

根据前文的分析，针对LLM训练场景这些独特的特征，阿里云设计了一套新的网络架构。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/35b5de994c364294bb9869f3e2d76bcc~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=%2FS17Qm9EU%2FWK%2FZlHb%2BRDahl6X3Q%3D)
HPN网络架构分为前端网络和后端网络，后端网络主要承载LLM训练过程中的流量，前端网络承载其他流量（如管理、推理、存储等流量）。后端网络每台host插有8张GPU和9张NIC，同一台 host 内的 GPU 通过[NVLink](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=NVLink\&zhida_source=entity)以full-mesh方式（类似 spine-leaf）互联，参考图9，图9的主机内拓扑是`2-2-4-6-8-8`结构，2 片 CPU（及两边的内存，NUMA），2 张存储网卡（访问分布式存储，带内管理等），4 个 PCIe Gen4 Switch 芯片，6 个 NVSwitch 芯片，8 个 GPU，8 个GPU 专属网卡。强调下，图9仅作为参考，且使用的PCIe4，而阿里云的HPN使用的[PCIe5](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=PCIe5\&zhida_source=entity)。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/90efeca74c8b469aa710d770277ea0bd~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=kidAf%2FUUI%2FBh6ZefNvfbCWzI8DM%3D)
阿里云HPN组网架构每个host内有9张2x200G CX7网卡，其中8个GPU专属网卡和1个存储网卡。8个GPU专属2x200G CX7网卡，也就是16个port，分别连接到不同的上联ToR，所以每个host会上联16个ToR，如图8所示。然后每个ToR有136（128 active + 8 backup）个200G下联口和60个400G上联口，136个下联口意为着可以接136个host，60个上联口，意为着可以接60个聚合交换机。16个ToR，奇数ToR接入的是一个平面，偶数ToR接入另一个平面。

一个Pod有两个网络平面，15个segment。每个网络平面有60台聚合交换机，每个segment有136台host，16个ToR，每个host有8张GPU，每个ToR有136个下联口，和60个上联口。那么每个segment内GPU数量为（），每个Pod包含GPU（）。另外，ToR的下联口是200G，而上联口是400G，（）。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/78bfdbf0f782483fbb79753774e55755~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=QZOStLlwIyeDyAwxkGNGEA8M53c%3D)
为什么一个Pod内有15个segment，因为一个segment内会有8个ToR上联同一个平面，而聚合交换机的下联口是120个，所以一个聚合交换机可以支持个segment。

为了满足长期支持更多 GPU（例如 O(100K) GPU）的规划，阿里云HPN还设计了连接多个 Pod 的 tier3 网络。为了增加Pod的规模，Aggregation-Core 之间的oversubscription是15:1。

抛出一个问题，为什么单上联没法在PoD内做到15K？之前我也没仔细想，最近和同事聊到了这个问题。

[TH5交换机芯片](https://zhida.zhihu.com/search?content_id=247985621\&content_type=Article\&match_order=1\&q=TH5%E4%BA%A4%E6%8D%A2%E6%9C%BA%E8%8A%AF%E7%89%87\&zhida_source=entity)最高带宽51.2T，图8中S0（136*200 + 60 \* 400）= 51.2T，S1 （120*400 + 8*400）= 51.2T，S2 (128 \* 400) = 51.2T。如果是单上联，同样一个segment要支持1K GPU，那么ToR需要128个400G下联口，按收敛比1:1计算，那么ToR的上联口也需要128个，S0（128*400 + 128 \* 400）= 102.4T，显然交换机支持不了。

# 3、效果

图10（a）中，端到端的训练性能提升了14.9%以上，这种端到端的性能提升在生产中其实是很大的价值，考虑到整个训练集群的建设可能要耗费数十亿美元，14.9%的性能提升可以带来显著的成本节约。如图10（b）所示，跨网段流量平均减少了37%。跨网段流量越少，网络拥塞就越少。图15（c）说明了聚合交换机下行链路的队列长度分布。在DCN+中，高流量和哈希碰撞不断形成队列，而在HPN中，这个问题得到了大大缓解。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/4223658684e54d288704f41079c7561d~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=M1z8mNmOXTEUrM5j7gBuYdVDt1c%3D)
如图 11（a） 所示，在单 ToR 拓扑中，链路故障发生在 10 秒时，训练立即停止。如果可以在 1 分钟内找到并修复故障，训练可以恢复。但是，如果修复需要两分钟以上，则训练无法恢复。相反，对于双 ToR，单个链路故障仅会导致 6.25% 的性能下降。在修复故障的同时，训练吞吐量立即恢复正常。图 11（b） 显示了链路抖动的影响，在单 ToR 中，临时链路抖动使训练停止超过九秒。在双 ToR 中，性能下降可以忽略不计。

![]()

![image.png](https://p0-xtjj-private.juejin.cn/tos-cn-i-73owjymdk6/882af369adcc4f7db526e7fe6fcd2c74~tplv-73owjymdk6-jj-mark-v1:0:0:0:0:5o6Y6YeR5oqA5pyv56S-5Yy6IEAg6Zi_5ouJ5pav5Yqg5aSn6Ze46J-5:q75.awebp?policy=eyJ2bSI6MywidWlkIjoiNDA1NTQ0Mjg4NzAyMjExOSJ9&rk3s=f64ab15b&x-orig-authkey=f32326d3454f2ac7e96d3d06cdbb035152127018&x-orig-expires=1760695848&x-orig-sign=bRpjD6Kg156e6KNhHhksK8rak4g%3D)

## 4、总结

为了解决load imbalance和单点故障问题，HPN提出了去堆叠双上联双平面的设计，大大提高了大规模部署中的可靠性问题。HPN 双平面架构单Pod中可以容纳 15K GPU（这是训练集群的常规规模），而不需要 3 层 Clos 架构。大二层双平面的设计还大大减少了 ECMP 的发生，避免了聚合层中的哈希极化。

CX7两种类型的卡，一是单卡单口400G，另一种是单卡双口2 x 200G，两种类型卡的选择看组网需求吧。

百度和腾讯星脉的组网方案和阿里云的HPN不一样，百度采用的单上联多平面，腾讯的星脉2.0也是单上联多平面，连接我都放在参考文献里了。

## 5、参考文献

【1】[https://www.youtube.com/watch?v=s-3VLs9sd10](https://link.zhihu.com/?target=https%3A//www.youtube.com/watch%3Fv%3Ds-3VLs9sd10)

【2】[https://ennanzhai.github.io/pub/sigcomm24-hpn.pdf](https://link.zhihu.com/?target=https%3A//ennanzhai.github.io/pub/sigcomm24-hpn.pdf)

【3】[https://arthurchiao.art/blog/gpu-advanced-notes-1-zh/](https://link.zhihu.com/?target=https%3A//arthurchiao.art/blog/gpu-advanced-notes-1-zh/)

【4】[大规模 AI 高性能网络的设计与实践](https://link.zhihu.com/?target=https%3A//cloud.baidu.com/article/364290)

【5】[面向大模型，如何打造云上最强算力集群？ | Tencent Digital Ecosystem Summit 2023 | NVIDIA On-Demand](https://link.zhihu.com/?target=https%3A//www.nvidia.com/en-in/on-demand/session/tencentdes2023-tdes2303/)
