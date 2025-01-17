[toc]

plethora of ML frameworks：NCCL, Horovod, BytePS, Mesh-TensorFlow, Gpipe, Ray, HugeCTR, DALI

### TensorFlow Internals

#### chpt1: 介绍

概念：数据流图、DAG、本地设备集

DistBelief: 异步SGD（model replicas），主从结构

TensorFlow: 延迟计算、原子OP、抽象设备（CPU、GPU、ASIC）、抽象任务（基于任务的PS）


#### chpt2: 编程环境

https://www.tensorflow.org/install/source#ubuntu

#### 附录A：代码阅读

* 发现领域模型
* 抛开细枝末节： `git checkout -b code-reading`
* 适可而止，BFS阅读




#### TensorFlow: Large-Scale Machine Learning on Heterogeneous Distributed Systems [2015]

node对应operation

edge对应tensor

* 也可是control dependencies，应用：控制算法、control the peak memory usage

operations and kernels

* attributes -> make operations polymorphic 
* kernel: a particular implementation of an operation that can be run on a particular type of device (e.g., CPU or GPU)
* 定义operations and kernels: registration mechanism

Sessions: 支持Extend和Run

Variables: a special kind of operation that returns a handle to a persistent mutable tensor that survives across executions of a graph. Handles to these persistent mutable tensors can be passed to a handful of special operations, such as `Assign` and `AssignAdd` (equivalent to +=) that mutate the referenced tensor. 

##### 3.Implementation

subgraph ~ devices  <img src="https://www.zhihu.com/equation?tex=%5Cstackrel%7B%5Cbf%7B%E5%A4%9A%E5%AF%B9%E4%B8%80%7D%7D%7B%5Clongrightarrow%7D" alt="\stackrel{\bf{多对一}}{\longrightarrow}" class="ee_img tr_noresize" eeimg="1"> workers  <img src="https://www.zhihu.com/equation?tex=%5Cstackrel%7B%5Cbf%7B%7D%7D%7B%5Clongrightarrow%7D" alt="\stackrel{\bf{}}{\longrightarrow}" class="ee_img tr_noresize" eeimg="1">  master  <img src="https://www.zhihu.com/equation?tex=%5Cstackrel%7B%5Cbf%7Bsession%7D%7D%7B%5Clongleftarrow%7D" alt="\stackrel{\bf{session}}{\longleftarrow}" class="ee_img tr_noresize" eeimg="1">  client 

device信息: the job of "the task of worker" or "localhost"

* `/job:localhost/device:cpu:0` or `/job:worker/task:17/device:gpu:3`

Tensor backing store buffers are reference counted

Execution

* Single-Device Execution: 每个 node 存未执行的依赖数，降为0进入ready queue
* Multi-Device Execution
  * Node Placement: cost model 估算 node 在特定 device 上执行用时，simulated execution, 贪心算法
  * Cross-Device Communication: Send and Receive Nodes, 给特定tensor、特定device限制下的所有users只分配一次空间; scheduling下放给节点执行，而非master

分布式实现：per subgraph per device, TCP or RDMA

* device层面自然地达成 CPU & GPU 并行计算

**4.Extensions**

4.1 Gradient Computation

* 如果 extend the TensorFlow graph，自动地加入 gradient tensors，那么关于 tensor 使用位置/先后顺序 预测的 heuristic 可能break down，最先使用的 tensor 到最后依然需要使用
* improvements to memory management, options include
  * using more sophisticated heuristics to determine the order of graph execution
  * recomputing tensors instead of retaining them in memory
  * swapping out long-lived tensors from GPU memory to more plentiful host CPU memory.

4.2 Partial Execution

* First, the Run call accepts inputs, an optional mapping of `name:port` names to “fed” tensors values. Second, the Run call accepts output names, a list of output `name[:port]` specifications indicating which nodes should be executed, and, if the port portion is present in a name, that that particular output tensor value for the node should be returned to the client if the Run call completes successfully.
* 根据feed node和fetch node决定partial graph

4.3 Device Constraints

* 限制的范畴：1）GPU/CPU 2)task 3)colocate with some variables
* 实现利用并查集先分析colocation，再缩小devices范围，输入到placement algorithm's simulator

4.4 Control Flow

* The Switch
  and Merge operators allow us to skip the execution of
  an entire subgraph based on the value of a boolean ten-sor. The Enter, Leave, and NextIteration operators allow
  us to express iteration.

4.5 Input Operations

* input nodes，通过文件feed，client到worker需要一个额外的network hop

4.6 Queues

* 存下数据，意义是为了prefetch from disks，或者收集梯度进行更复杂的操作

* FIFO / Shuffling Queues

4.7 Containers

* Using containers, it is possible to share state even across
  completely disjoint computation graphs associated with
  different Sessions.

**5.Optimizations**

5.1 Common Subexpression Elimination

5.2 Controlling Data Communication and Memory Usage

* e.g. 分析 critical path，用 control edge 来 delay Receive Nodes

5.3 Asynchronous Kernels

5.4 Optimized Libraries for Kernel Implementations

5.5 Lossy Compression

* 参考"Hitchhiker"论文Arithmetic Precision这节

**7.Common Programming Idioms**

同步/异步SGD

**9.Tools**

9.1 TensorBoard

9.2 Performance Tracing: EEG



#### TensorFlow: A system for large-scale machine learning [OSDI, 2016]

**Introduction**
* TensorFlow allows vertices to represent computations that own or update mutable state.
* synchronous replication

While MXNet partially fulfills our extensibility requirements, the parameter server is “privileged” code, which makes it difficult for researchers to customize the handling of large models

**3.TensorFlow execution model**

Dataflow with mutable state 是tf吸取PS架构的经验 

几种训练方式的讨论
* 同步：大并发、无gradient staleness、scalable
* 异步：资源利用率高 (maintain high throughput in the presence of
  stragglers)；可以只使用一部分worker的梯度做更新，虽然损失了信息，但减少了异步带来的冲突
* 半同步：dense同步、sparse异步



**4.3 Fault tolerance**

Having checkpoint and parameter management as programmable operations in the graph gives users the flexibility to implement schemes like these and others that we have not anticipated.



**4.4 Synchronous replica coordination**

synchronous with backup workers，和MapReduce的backup方案对比，更 proactive



原生tensorflow架构分析：

* 优点：
  * 无需开发PS
    * 实现需要额外存储变量的op在原生tf更为简单
    * 新optimizer的探索不需要单独部署PS

* 缺点：
  * distributed runtime有通信问题，每个slot产生一对send/recv op，对于大规模embedding的场景基本训不动模型



### TensorFlow Serving

#### 《TensorFlow-Serving: Flexible, High-Performance ML Serving》



load模型（尤其是对模型进行warmup）导致延迟spike的问题，确实不容易解决。特别复杂的模型warm up引起服务cpu抖动，可能是因为线程不够了

2.Library

2.1 Model Lifecycle Management

* Source, Source Adapters, Source Routers
* Canary and Rollback
* Aspired Versions Manager
  * RCU

2.2 Inference

* Inter-Request Batching（图内/外）

3.Canonical Binary and Hosted Service



[How Zendesk Serves TensorFlow Models in Production](https://medium.com/zendesk-engineering/how-zendesk-serves-tensorflow-models-in-production-751ee22f0f4b)

[美团：基于TensorFlow Serving的深度学习在线预估](https://tech.meituan.com/2018/10/11/tfserving-improve.html)



#### [Jeff Dean: Achieving Rapid Response Times in Large Online Services](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/44875.pdf)

讨论了分布式服务的通用优化思想，很值得学习！

shared environment 提升资源利用率的同时，也带来不可预测的因素（比如network congestion、background activities、bursts of foreground activity、not just your jobs, but everyone else’s jobs, too），影响服务长尾延时，并且会 exacerbated by large fanout systems

Conclusion

* Tolerating variability
  * important for large-scale online services
  * large fanout magnifies importance
  * makes services more responsive
  * saves significant computing resources
* Collection of techniques
  * general good engineering practices
    * prioritized server queues, careful management of background activities
  * cross-request adaptation
    * load balancing, micro-partitioning
  * within-request adaptation
    * backup requests, backup requests w/ cancellation, tainted results



### Go+Torch

https://github.com/wangkuiyi/gotorch

Q: TensorFlow为什么需要引入图这个概念

A: 

1.backward自动求导，需要定义前向的数据结构

2.python执行速度慢，决定执行效率的是图的解释器。图是python代码的另一种表示形式，开始包括前向计算过程，通过调用TensorFlow API，加入其它op包括反向计算过程和模型更新过程。构造图本质上是在编译。

* [TFRT](https://github.com/tensorflow/runtime)

调用libtorch内部的native function类比tf的op，但native function是函数，而不是一个class，每一个function可以用HLO（一种古老的适用于数值计算的语言）写一遍。gotorch调libtorch调pytorch XLA里的HLO程序，翻译成特定设备优化的代码

* native function有YAML描述，可自动生成Go Wrapper
* torchscripts：用到的python语法的子集 => python高层api可翻译成torchscripts再翻译

如果 Go+Torch 在未来一年里孕育成熟，有望优化以下核心 应用场景:

1. 统一训练和预测系统(目前训练用 Python 写，预测用 C++)
2. 统一云和端系统(目前云上用 TensorFlow，端上比如 xNN 调用 TensorFlow Lite)
3. 统一训练和预测时的数据处理流程(目前需要用 Python和C++分别做两套，开销大，而且容易出错)
4. 统一搜索、推荐、广告、金融核心、移动智能和端智能、无人驾驶等多个领域的基础架构
5. 能支持新的机器学习模式——online learning、GAN、reinforcement learning、imitation learning等。

### OneFlow: 大规模分布式深度学习框架

数据并行：allreduce + PS

模型并行：参数如何划分？复杂的通信模式

![platforms](https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/MLSys/platforms.jpg)

横向拓展：片间高速互联，e.g. TPU

纵向拓展：单个芯片从通用到专用



静态调度与流式执行系统![layers](https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/MLSys/layers.jpg)



OneFlow架构

* actor及流水线
  * 内存槽，用类似rust的ownership解决内存冲突问题，ownership随状态转移

![memory-pipeline](https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/MLSys/memory-pipeline.jpg)

* node placement: consistent view
  * SBP, 在op层面实现数据和模型并行 
![SBP](https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/MLSys/SBP.jpg)


### GPU 相关知识

共享卡 —— 如何实现算力和显存隔离
* 隔离方式：时间片 vs 空间
* 隔离级别：不隔离 vs 强隔离 vs 弹性
* 几种隔离技术对比：
  * vGPU(Grid)(Nvidia)：虚拟化；容器化支持不好，license
  * vCuda(腾讯)：cuda hook；性能损耗严重
  * cGPU(Alibaba)：ioctl；损耗小，硬隔离，侵入内核（机器容易坏）
  * MPS(Nvidia)：thread；显存隔离，故障隔离不好
  * MIG(~A100)：sm/global memory；硬件层面隔离



### 论文阅读

已读，待整理：

#### 《MLSys: The New Frontier of Machine Learning Systems》

#### 《Deep Neural Networks for Youtube Recommendations, RecSys 16》

#### 《Wide & Deep learning for Recommender Systems, RecSys 17》

1.Introduction
* Wide ~ Memorization: 模型直接学习并利用历史数据中物品或者特征的“共现频率”的能力
* Deep ~ Generalization: 模型传递特征的相关性，以及发掘稀疏甚至从未出现过的稀有特征与最终标签相关性的能力
* Generalized linear models with nonlinear feature transformations
* cross-product transformations: 特征工程的概念，交叉积变换，缺点是无法generalize没出现过的query-item feature pairs

问题：输入是稀疏高秩矩阵，缺少interactions，难以利用它学到合适的低维embedding

3.WIDE&DEEP Learning

3.1 The Wide Component

利用cross-product transformation提供多个特征的非线性

<->  对比：[Deep Neural Networks for YouTube Recommendations ] 用平方和平方根项提供单个特征的非线性

3.3 Joint Training of Wide & Deep Model

注意辨析joint training和ensemble的区别
* 前者是共同训练，后者不是
* 后者模型可以更大
* 前者，Wide只需要给Deep补少量cross-product feature transformations

4.System Implementation 4.2 Model Training
* warm-starting system
* dry run
* sanity check

Appendix

* 概念：AUC：ROC曲线下方的面积，ROC横坐标FPR，纵坐标TPR
* 资源：
  * 这个大佬的专栏很实用，讲解tensorflow和推荐系统，https://zhuanlan.zhihu.com/learningdeep
* 思考：可否联系到IRLS方法，最优化稀疏矩阵的秩，用一个类似的矩阵学习秩的表示



**改进：Deep&Cross模型**

* 多层交叉层:  <img src="https://www.zhihu.com/equation?tex=x_%7Bl%2B1%7D%3Dx_0x_l%5ETw_l%2Bb_l%2Bx_l" alt="x_{l+1}=x_0x_l^Tw_l+b_l+x_l" class="ee_img tr_noresize" eeimg="1">  
  * 参数引入较为克制，增强模型的非线性学习能力
  * 解决了Wide&Deep模型人工组合特征的问题

#### 《A Hitchhiker's Guide On Distributed Training Of Deep Neural Networks, JPDC 18》

#### 《TFX: A TensorFlow-based production-scale machine learning platform》

#### 《TensorFlow: A system for large-scale machine learning, OSDI 16》

#### 《Clipper: A Low-Latency Online Prediction Serving System, NSDI 17》

low latencies, high throughputs, and improved accuracy

prediction cache, batching queue

##### Model abstraction layer

用object store存模型，减少初始化开销

prediction cache：本质上类似SHARED属性（同一batch内的某一特征用相同的预估结果）。两者的区别在于，前者的输入更简单，以模型和req id为标识，易于做cache操作；后者是feature层面，更精细。推荐系统入图的特征输入很难做到完全一致，因此做prediction cache操作难度较大。

batching：动态选batch size的方式
* additive-increase-multiplicative-decrease (AIMD) scheme 
* quantile regression
* delayed batching：按攒batch的timeout来delay，适合并行优化明显的模型

model container: 无状态服务
* Clipper performs adaptive batching independently for each replica

##### Model selection layer

动态调整选用模型的策略，推荐系统采用这类技术比CV/NLP难度更大

* Single Model Selection Policy
  * address the trade-off between exploring possible actions and exploiting the estimated best action. 
* Ensemble Model Selection Policies
  * Robust Prediction 
    * agreement衡量prediction confidence 
    * 有针对degraded模型的降级机制
  * Straggler Mitigation
* Contextualization: instantiate a unique model selection state for each user, context, or session.



#### 《Hidden Technical Debt in Machine Learning Systems, NIPS 15》

boundary erosion, entanglement, hidden feedback loops, undeclared consumers, data dependencies, configuration issues, changes in the external world, and a variety of system-level anti-patterns.

2. Complex Models Erode Boundaries
* Entanglement: 即使多模型/超参的配置独立，效果也会互相影响
* Correction Cascade: 模型级联是hidden debt
* Undeclared Consumers: 需要SLA(service-level agreement)

3. Data Dependencies Cost More than Code Dependencies
* Underutilized dependencies: legacy/bundled/epsilon/correlated, use exhaustive leave-one-feature-out evaluations to detect

4. Feedback Loops
* direct: related to bandit algorithms, costly
* hidden: two independent systems may interact

5. ML-System Anti-Patterns
* Glue Code: hard to achieve a domain-specific goal
* Pipeline Jungle: 特征工程的意义所在，thinking holistically about data collection and feature ex traction
* Dead Experimental Codepaths
* Abstraction Debt
* Common Smells

6. Configuration Debts
* Feature A was incorrectly logged from 9/14 to 9/17
* Feature B is not available on data before 10/7
* The code used to compute feature C has to change for data before and after 11/1 because of changes to the logging format
* Feature D is not available in production, so a substitute features D′ and D′′ must be used when querying the model in a live setting
* If feature Z is used, then jobs for training must be given extra memory due to lookup tables or they will train inefficient
* Feature Q precludes the use of feature R because of latency constraints.

7. Dealing with Changes in the External World




#### 《Ad Click Prediction: a View from the Trenches, KDD 13》
* a high-dimensional visualization tool was used to allow researchers to quickly see effects across many dimensions and slicings
* enables data sources and features to be annotated. Automated checks can then be run to ensure that all dependencies have the appropriate annotations, and dependency trees can be fully resolved.

#### 《XDL: An industrial deep learning framework for high-dimensional sparse data, KDD 19》

MPI(All Reduce)和PS，两种分布式计算方向

Sparse + Dense

* SparseNet: Representa-tion learning which captures information from high-dimensional sparse input and embeds them into a low-dimensional space

* DenseNet: Function fitting which models the relationship between dense em- bedding representation and supervised label

In order to facilitate deployment on various computing platforms,
XDL can be scheduled by multiple resource management platform, like Yarn, and provides data I/O interfaces to various data storage systems, like HDFS and Kafka.



* I/O
  * Hierarchical sample compression: prefix tree

![prefix-tree](https://raw.githubusercontent.com/huangrt01/Markdown-Transformer-and-Uploader/mynote/Notes/MLSys/prefix-tree.png)

* Workflow pipeline

  * I/O: read sample and group mini-batch -> prefetch (maybe cudaMemcpyAsync()) -> pull/forward/backward/push
  * SparseNet and DenseNet

* Optimization for Advanced Model Server

  * Network: [Seastar](https://github.com/scylladb/seastar) + zero-copy/CPU-binding

* Online Learning with XDL

  * Feature Entry Filter
  * Incremental Model Export
  * Feature Expire

#### 《Ethane: Taking control of the enterprise, SIGCOMM 2007》

make networks more manageable and more secure，一种思路是全方位的增加控制，相当于新增一层，只是hide了复杂度；于是提出ethane

ethane的思想：
* The network should be governed by policies declared over high-
level names
* Policy should determine the path that packets follow
* The network should enforce a strong binding between a packet
and its origin.

Ethane的优势：
* Security follows management.

* Incremental deployability.

* Significant deployment experience.
  
  
#### 《Scaling distributed machine learning with the parameter server, OSDI 2014》

PS架构的优势主要还是高可用(system efficiency)

2.2
* distributed subgradient descent

3.6 User-defined Filters
* signifi-cantly modified filter
* KKT(见5.1)：特征重要性筛选

4.Implementation

4.2 Messages
* key-caching and value-compression can be used jointly.
* key-cache让sender只需要传key lists的hash
* 用snappy压缩 zero value

4.3 Consistent Hashing
一致性hash和 key-range 的概念紧密相连，论文 Chord: A scalable peer-to-peer lookup protocol for Internet applications

4.5 Server Management
* 计算节点分为server node和worker node
* server共同维持全局共享的模型参数
* workers保留一部分的训练数据，并且执行计算
* worker只和server有通信，互相之间没有通信

examples
* CountMin Sketch Algo 有点像 bloom filter

PS运维：
* expectation - current_situation = operations
* 服务发现、数据发现

性能优化：
* 双buffer + RCU，读不被锁阻碍
* 简化版读写锁，优化系统态开销

#### 《Serving DNNs like Clockwork: Performance Predictability from the BottomUp, OSDI 2020》

[presentation](https://www.usenix.org/conference/osdi20/presentation/gujarati) 挺有意思

model serving: ML system's "narrow waist"

这篇文章尝试解决服务化请求长尾问题

首先分析产生长尾的原因：out-of-order scheduling, interference from concurrency, power saving modes, and network queuing delays.
然后基于以下两个假设：
1) “DNN inference is predictable.”
2) 能限制系统到应用层面的决策能力（减少worker内部的并行）

提出解决方案：
分布式系统常用的思路，request打到worker之前，先过一个中心controller，中心controller掌握全局信息（模型是否load、worker是否pending等），预测latency是否会超过SLA，以决定将请求打到哪个worker

感觉这一系统难以直接应用于大公司的场景，因为：

1.需要和rpc框架做更深的结合

* 长尾问题本身有一部分是来自于服务化带来的网络传输开销，比如thrift worker负担，只有rpc框架能掌握更多信息
* 如果要落地到生产场景，自制的简陋 controller 不易推广

2.自身的优势不明显

* 分业务服务化部署、并且是online learning的场景，显存不是瓶颈，模型本身已经是preload了
* scalable能力未经过验证 (6.6)，controller成为瓶颈

有启发的地方
* 框架内的page cache可以借鉴一下 (https://gitlab.mpi-sws.org/cld/ml/clockwork/-/blob/master/src/clockwork/cache.h)
  
  