---
title: Multi-path Flow Scheduling Algorithm Based on SDN(一)
tag: programming
---
纪录一下科研近况

基于SDN的多路径大流调度问题主要分成两部分。
1.Selection Algorithm (large flow)
选流算法，有选择性的选出部分长流进行重新调度。并使用分段路由技术提高流的传输速度
2.Routing Algorithm
选路算法，动态规划路径，减少功耗

# Selection Part1(选流部分)
## Background 
传统的数据中心主要采用的是层次式的拓扑结构，这种结构在有大量数据出入数据中心时很有用，但是如果是是数据在数据中内部流动，那么这种逻辑结构将会让带宽分布不均匀。VL2和FatTree是Clos型的拓扑结构，这种结构采用在网络上的两台主机之间利用多核交换来提供足够的带宽。FatTree主要用在低速交换（1Gb/s），VL2用在稍高速交换（10Gb/s），而恰恰相反，BCube主要采用主机自己传播流量的方式.
在路由部分，由于密集型互联拓扑结构在两个主机之间会产生多条平行的路径，而主机无法知道那条路径是最低负载的，最简单的办法就是使用随机算法安排路径负载,最简单的办法就是使用随机算法安排路径负载。在实际问题中，有很多的办法在交换中实现随机负载平衡。
在路径选择方面，如果采用随机的办法，那么很容易产生部分点过热，也就是部分路径过载 。
解决这种问题的办法主要思路就是让大流量被安排到负担比较轻的路径上去，以及对已经存在的流进行从新安排，会使得整个网络的流最大化。

## Common Method
一、Transfer layer control based on end host(基于端主机的传输层控制)
主要选了两篇经典文章，之后会贴到后面的链接，正文会大概讲一下思想。
### 1.Dater Center TCP
2010，SIGCOMM 
通过控制端到端的传输速率以缓解网络拥塞,其利用交换机进行链路拥塞检测,当发现出口队列的长度超出阈值时,对进入队列的包进行标记,发送端根据被标记的包的比例估计链路拥塞程度调整发送窗口的大小。
Two key ideas:
* React in proportion to the extent of congestion, not its presence.
- Reduces variance in sending rates, lowering queuing requirements.
* Mark based on instantaneous queue length.
- Fast feedback to better deal with bursts.
![DCTCP](/images/dctcp.jpeg)

### 2.Multi-Path TCP(MPTCP)
2011，SIGCOMM
Improving Datacenter Performance and Robustness with Multipath TCP
他的实现在原来的OSI架构里面插入了一层MPTCP。
* Implement
- master subsocket
- multi-path control bock (mpcb)
- slave subsocket

master subsocket是一个标准的sock结构体用于TCP通信。
multi-path control bock(mpcb)提供开启或关闭子通道、选择发送数据的子通道以及重组报文段的功能。
slave subsocket对应用程序并不可见，他们都是被mpcb管理并用于发送数据。

![MPTCP1](/images/mptcp1.jpeg)
MPTCP通过在主机之间建立多条路径进行传输,其使用多网卡主机,将一条流分割为多条子流,均衡地分配到各个链路上,MPTCP不再使用传统TCP协议所要求的单个信道，而是支持冗余信道资源的反向多路复用，将整个数据传输速率提高到所有可用信道的总和,架构如下。
![MPTCP2](/images/mptcp2.jpeg)
![MPTCP3](/images/mptcp3.jpeg)
虽然与标准TCP工作方式很像，但是，MPTCP的核心思想是定义一种在两个主机之间建立连接的方式，而不是在两个接口之间（例如标准TCP）。在标准TCP中，连接应在两个IP地址之间建立。每个TCP连接由标志着源和目的地的地址和端口的四元组来标识。鉴于此限制，应用程序只能通过单个连接创建一个TCP连接，因此会出现两个主机之间虽然可能同时建立了多个连接，但同一时刻只有单个连接被某个应用利用，而 Multipath TCP则允许连接同时使用多个路径。为此，MultipathTCP在每个需要使用的路径上创建一个称为subflow的TCP连接。

新加入的 MPTCP 层功能主要包含2个:
1. 分流,把传统 的 TCP 数据进行分流,分别在不同的子流上传输;
2. 路径管理,用来探测和管理通信双方的可用路径。具体来说,MPTCP 按功能可以分 为路径管理(PM)和包调度(PS),如图 figure.2 所示。路径管理负责通信双方的路径发 现;包调度的功能包括:包的调度、子流接口和拥塞控制。当包调度接收到来自应用层的数据后,首先对数据进行处理,然后发送给子流;子流对数据添加序列号和确认号传给网络层。目的端子流收到数据后,将其交付给MPTCP,MPTCP对数据进行处理后上传给应用层。作为包调度的组成部分, 拥塞控制用来控制各个子流的发送速率.

当子流接收到发送端发来的数据后,子流将数据交付给 MPTCP,MPTCP 对数据重组后再向上提交给应用层。 为了实现数据的按序接收,在 TCP 里采用序列号的机制对数据进行编号。MPTCP也采用类似对数据编号的机制,不过数据编号采用分层的两级模式——子流层序列号和 数据层序列号。子流层序列号即是 TCP 中的序列号,而 数据层序列号被封装在选项字段.

MPTCP在所有多路径流控文章里面基本上都会对比，实际上弊端也很多，比如这么多链路产生的ACK，握手信息之类，抑或是之后队列接收时候的重新排序，都导致性能下降，而且拥塞机制也不完善。下面是一篇inform对其的评价。
offer greater parallelism but cannot predict down- stream collisions, forcing them to continuously react to congestion。

二、Local load balancing based on the switch(基于交换机的本地负载均衡)
### Local Flow
2010,ACM，CoNEXT
Solved locally at each switch by LocalFlow
LocalFlow利用交换机 对网络中的大流进行分割,将数据包分配到不同的端口上,实现链路利用率的最大化.
要了解Localflow机制就要先了解PacketScatter.
PacketScatter:
* Essentially round-robins packets over a switch’s outgoing links.
* PacketScatter is load agnostic: it splits every flow, which causes packet reorder- ing and increases flow completion times
![LOCALFLOW](/images/localflow.jpeg)
引入参数δ，δ∈ [0,1]
－ δ = 1,no split 
－ δ = 0,PacketScatter
图上很清楚了，没什么好说的。

### FlowBlender
2014, ACM，CoNEXT
FlowBlender对交换机内的哈希函数 进行了改进,函数的输入参数新增了一个灵活域,当链路发生 拥塞时,交换机通过调整该灵活域的值改变 ECMP 的哈希结果 从而改变包的输出端口 
（1）以流量的粒度分配负载均衡，而不是分组，避免过多的数据包重新排序。 
（2）使用终端主机驱动的rehashing来触发动态流到路径分配。 
（3）从重传超时（RTO）中的链路故障恢复。 
（4）量少于50线的关键内核代码，并且可以在今天的商品数据中心轻松部署。

还有篇类似的DDFR，只看了个introduction.参考广义负载共享模型,设计了一种自适应算法,将数据流公平的分配到各条上行链路上。 基于交换机的本地负载均衡可以提高链路利用率,但仅考虑了局部优化的效果,没有从全网整体的角度进行优化设计.

三、Controller - based centralized flow scheduling
### Hedera
2010 NSDI’
提出了两种经典式算法，simulated annealing(SA，模拟退火算法) & global first fit(GFF)。
Hedera在全局网络视图下利用模拟退火算法(simulated annealing,SA)周期性地调整网络中大流的传输路径。
![HEDERA](/images/hedera.jpeg)

这篇也是现有思路的最好方向。
将整个网络分为3个模块。
* Large flow detection module(大流检测模块)
* Network state maintenance module(网络状态维护模块)
* Flow scheduling module(流调度模块)
OpenFlow 交换机默 认采用 ECMP 的方式对网络中的流进行路由,且优先传输小流; 控制器通过轮询操作周期性地收集当前网络状态信息,包括拓扑结构、链路的可用带宽、大流的吞吐量和路由路径等,信息的管理和更新由网络状态维护模块完成。

大流检测模块检测出需要规划的大流。
网络状态维护模块更新过全网视图后,将当前网络状态信息转发给流调度模块。
流调度模块首先筛选出按照当前路径传输,保证网络吞吐量最大的大流集合,对于集合内的大流,维持原路径传输;对于不属于集合的大流,则为其计算出更优的传输路径。
OFswitch:
利用主机对进入网络的流进行区分,OpenFlow 交换机默认以 ECMP 的方式路由且优先传输小流,控制器周期 性地从网络中收集大流信息,并调整其路径。
# Selection Part2(选路部分)
常用的选路算法分为5类。
* Dynamic routing
* Global first fit
* Simulated annealing
* Multipath routing (parallel)
* Hot spot avoidance
下面是简单介绍。
1. Dynamic routing
－ 在所有路徑滿載前給每個flow分配最空閒的路徑
－ 滿載後使用round-robin

2. Global First Fit
－ 遍歷所有路徑, 有位置就放, 沒位置則用ECMP的模式
－ 速度快但結果容易被flow次序影響

3. Simulated annealing(模擬淬火)
初始化: 使用一個可能的路徑規劃與一個溫度, 進行數次iteration

每次迭代:
    1. 為隨機source/destination組合隨機交換路徑
    2. 當新路徑的cost小於原cost則有P機率使用新路徑
    3. 當溫度降到0時結束

－ 利用隨機變換路徑的方式尋找全局最優解
－ 由於變換路徑的機制完全隨機, 優化結果可能不固定

4. Multipath routing (parallel)
－ flow通過大於一條link同時傳輸
－ 速度可能大幅上升但可能需要軟/硬體支持

5. Hot spot avoidance
－ flow經常試圖以全部頻寬進行傳輸 → 當多條流使用同一路徑就會競爭資源(hot spot)
－ 偵測到hot spot則為原有規劃換路


# Resource
要理解本文我觉得你可能需要了解：
1.[等价多路由协议-wiki](https://zh.wikipedia.org/wiki/%E7%AD%89%E5%83%B9%E5%A4%9A%E8%B7%AF%E5%BE%91%E8%B7%AF%E7%94%B1).注意等价和多路径(最多不超过3条路径).ECMP不接受相差带宽很大的链路的原因你要了解.
2.[DCTCP](http://dl.acm.org/citation.cfm?id=1851192).
3.[MPTCP](http://queue.acm.org/detail.cfm?id=2591369).这是MPTCP，本文引的事基于MPTCP的链路负载.
4.[Improving datacenter performance and robustness with multipath TCP](http://dl.acm.org/citation.cfm?id=2018467).
5.[Scalable, Optimal Flow Routing in Datacenters via Local Link Balancing](http://dl.acm.org/citation.cfm?id=2535397).LocalFlow
6.[FlowBender: Flow-level Adaptive Routing for Improved Latency and Throughput in Datacenter Networks](http://dl.acm.org/citation.cfm?id=2674985).FlowBender.
7.[Hedera: Dynamic Flow Scheduling for Data Center Networks](https://www.usenix.org/legacy/event/nsdi10/tech/full_papers/al-fares.pdf).Hedera.
8.[选流部分ppt By Xuanlong](https://drive.google.com/open?id=0B2GNLDTkTqeKd3FUdFhOLU0tZUE).
9.[选路部分ppt By ChingHsuan](https://docs.google.com/presentation/d/12K91hzOCOBGn4GsNQ76vLzfdaKTjglMDij8rw1pdHLE/edit?usp=sharing).





