# Chapter 1: Reliability, Scalability, and Maintainability

# 本书为什么以数据系统为主题

**数据系统**（data system）是一种模糊的统称。在信息社会中，一切皆可信息化，或者，某种程度上来说——数字化。这些数据的采集、存储和使用，是构成信息社会的基础。我们常见的绝大部分应用背后都有一套数据系统支撑，比如微信、京东、微博等等。

![data-society.png](img/ch01-data-society.png)

因此，作为 IT 从业人员，有必要系统性的了解一下现代的、分布式的数据系统。学习本书，能够学习到数据系统的背后的原理、了解其常见的实践、进而将其应用到我们工作的系统设计中。

## 常见的数据系统有哪些

- 存储数据，以便之后再次使用——**数据库**(database)
- 记住一些非常“重”的操作结果，方便之后加快读取速度——**缓存**(cache)
- 允许用户以各种关键字搜索、以各种条件过滤数据——**搜索引擎**(search index)
- 源源不断的产生数据、并发送给其他进程进行处理——**流式处理**(stream processing)
- 定期处理累积的大量数据——**批处理**(batch processing)
- 进行消息的传送与分发——**消息队列**(message queue)

这些概念如此耳熟能详以至于我们在设计系统时拿来就用，而不用去想其实现细节，更不用从头进行实现。当然，这也侧面说明这些概念抽象的多么成功。

## 数据系统的日益复杂化

但这些年来，随着应用需求的进一步复杂化，出现了很多新型的数据采集、存储和处理系统，它们不拘泥于单一的功能，也难以生硬的归到某个类别。随便举几个例子：

1. **Kafka**：可以作为存储持久化(durability)一段时间日志数据、可以作为消息队列(message queue)对数据进行分发、可以作为流式处理(stream processing)组件对数据反复蒸馏等等。
2. **Spark**：可以对数据进行批处理(batch processing)、也可以化小批为流，对数据进行流式处理(stream processing)。
3. **Redis**：可以作为缓存加速对数据库的访问、也可以作为事件中心对消息的发布订阅。

我们面临一个新的场景，以某种组合使用这些组件时，在某种程度上，便是创立了一个新的数据系统。书中给了一个常见的对用户数据进行采集、存储、查询、旁路等操作的数据系统示例。从其示意图中可以看到各种 Web Services 的影子。

![data-system.png](img/ch01-fig01.png)

但就这么一个小系统，在设计时，就可以有很多取舍：

1. 使用何种缓存策略？是旁路(cache-aside)还是写穿透(write-through)？
2. 部分组件机器出现问题时，是保证可用性(availability)还是保证一致性(consistency)？
3. 当机器一时难以恢复，如何保证数据的正确性(correctness)和完整性(integrity)？
4. 当负载增加时，是增加机器还是提升单机性能？
5. 设计对外的 API 时，是力求简洁还是追求强大？

因此，有必要从根本上思考下如何评价一个好数据系统，如何构建一个好的数据系统，有哪些可以遵循的设计模式？有哪些通常需要考虑的方面？

书中用了三个词来回答：**可靠性（Reliability）、可扩展性（Scalability）、可维护性（Maintainability）**

# 可靠性 - Reliability

如何衡量可靠性？

- **功能上**(functional)
  - 正常情况下，应用行为满足 API 给出的行为
  - 在用户误输入/误操作时，能够正常处理
  
- **性能上**(performance)
  - 在给定硬件和数据量下，能够满足承诺的性能指标。

- **安全上**(security)
  - 能够阻止未授权、恶意破坏。


可用性也是可靠性的一个侧面，云服务通常以多少个 9 来衡量可用性。

---

两个易混淆的概念：**Fault（系统出现问题）** 和 **Failure（系统不能提供服务）**

不能进行 Fault-tolerance 的系统，积累的 fault 多了，就很容易 failure。

如何预防？混沌测试：如 Netflix 的 [chaosmonkey](https://netflix.github.io/chaosmonkey/)。

## 硬件故障 - Hardware Faults

在一个大型数据中心中，这是常态：

1. 网络抖动、不通(Network jitter or outages)
2. 硬盘老化坏道(Aging hard drives with bad sectors)
3. 内存故障(Memory failures)
4. 机器过热导致 CPU 出问题(CPU issues due to overheating)
5. 机房断电(Power outages in the data center)

数据系统中常见的需要考虑的硬件指标：

**MTTF mean time to failure** - 单块盘 平均故障时间 5 ~10 年，如果你有 1w+ 硬盘，则均匀期望下，每天都有坏盘出现。当然事实是硬盘会一波一波坏。

解决办法，增加冗余度(redundancy)：机房多路供电，双网络等等。

对于数据：

* **单机**：可以做 RAID 冗余。如：EC 编码。
* **多机**：多副本 或 EC 编码。

## 软件错误 - Software Errors

相比硬件故障的随机性，软件错误的相关性更高：

1. 不能处理特定输入(particular bad input)，导致系统崩溃。
2. 失控进程(runaway process)，如循环未释放资源，耗尽 CPU、内存、网络资源。
3. 系统依赖组件变慢甚至无响应(unresponsive)。
4. 级联故障(cascading failures)，由一个小的fault发展成大范围的fault，持续散播蔓延。

在设计软件时，我们通常有一些**环境假设**，和一些**隐性约束**。随着时间的推移、系统的持续运行，如果这些假设不能够继续被满足；如果这些约束被后面维护者增加功能时所破坏；都有可能让一开始正常运行的系统，突然崩溃。

## 人为问题 - Human Errors

系统中最不稳定的是人，因此要在设计层面尽可能消除人对系统影响。依据软件的生命周期，分几个阶段来考虑：

- **设计编码**
  1. 尽可能消除所有不必要的假设，提供合理的抽象，仔细设计 API
  2. 进程间进行隔离(decouple)，对尤其容易出错的模块使用沙箱机制
  3. 对服务依赖进行熔断设计
- **测试阶段**
  1. 尽可能引入第三方成员测试，尽量将测试平台自动化
  2. 单元测试、集成测试、e2e 测试、混沌测试(chaos testing)
- **运行阶段**
  1. 详细的仪表盘(dashboard)
  2. 持续自检(self-checks)
  3. 报警(Alert)机制
  4. 问题预案(Incident response plans)
- **针对组织**
  1. 科学的培训和管理

## 可靠性有多重要？

事关用户数据安全，事关企业声誉，企业存活和做大的基石。

# 可扩展性 - Scalability

可扩展性，即系统应对负载增长的能力。它很重要，但在实践中又很难做好，因为存在一个基本矛盾：**只有能存活下来的产品才有资格谈扩展，而过早为扩展设计往往活不下去**。

但仍是可以了解一些基本的概念，来应对**可能会**暴增的负载。

## 衡量负载 - Measuring Load

应对负载之前，要先找到合适的方法来衡量负载，如**负载参数（load parameters）**：

- 应用日活月活(DAU)
- 每秒向 Web 服务器发出的请求
- 数据库中的读写比率
- 聊天室中同时活跃的用户数量

书中以 Twitter 2012 年 11 月披露的信息为例进行了说明：

1. 识别主营业务：发布推文、首页 Feed 流。
2. 确定其请求量级：发布推文（平均 4.6k 请求/秒，峰值超过 12k 请求/秒），查看其他人推文（300k 请求/秒）

![twitter-table.png](img/ch01-fig02.png)

单就这个数据量级来说，无论怎么设计都问题不大。但 Twitter 需要根据用户之间的关注与被关注关系来对数据进行多次处理。常见的有推拉两种方式：

1. **拉**。每个人查看其首页 Feed 流时，从数据库现**拉取**所有关注用户推文，合并后呈现。
2. **推**。为每个用户保存一个 Feed 流视图，当用户发推文时，将其插入所有关注者 Feed 流视图中。

![twitter-push.png](img/ch01-fig03.png)

前者是 Lazy 的，用户只有查看时才会去拉取，不会有无效计算和请求，但每次需要现算，呈现速度较慢。而且流量一大也扛不住。

后者事先算出视图，而不管用户看不看，呈现速度较快，但会引入很多无效请求。

最终，使用的是一种推拉结合的方式(Use **push** for users with a **smaller follower base** and **pull** for users with **large follower counts**)

## 描述性能 - Describing Performance

注意和系统负载(system load)区分，系统负载是从用户视角来审视系统，是一种**客观指标**。而系统性能(system performance)则是描述的系统的一种**实际能力**。比如：

1. **吞吐量（throughput）**：每秒可以处理的单位数据量，通常记为 QPS。
2. **响应时间（response time）**：从用户侧观察到的发出请求到收到回复的时间。
3. **延迟（latency）**：日常中，延迟经常和响应时间混用指代响应时间；但严格来说，延迟只是指请求过程中排队等休眠时间，虽然其在响应时间中一般占大头；但只有我们把请求真正处理耗时认为是瞬时，延迟才能等同于响应时间。

响应时间通常以百分位点来衡量，比如 p95，p99 和 p999，它们意味着 95％，99％或 99.9％ 的请求都能在该阈值内完成。在实际中，通常使用滑动窗口滚动计算最近一段时间的响应时间分布，并通常以折线图或者柱状图进行呈现。

**Response time** not as a single number, but as a **distribution of values** that you can measure

**Even in a scenario where you’d think all requests should take the same time, you get variation**: random additional latency could be introduced by 后台进程的上下文切换(a context switch to a background process), 网络丢包(the loss of a network packet) and TCP重传(TCP retransmission), 垃圾回收暂停(a garbage collection pause), 触发磁盘读取的缺页中断(a page fault forcing a read from disk), 服务器机架的物理振动(mechanical vibrations in the server rack), or many other causes.

High percentiles of response times, also known as **tail latencies**, are important because they directly affect users’ experience of the service.

百分位数(**Percentiles**) are often used in 服务等级目标(service level objectives (**SLOs**)) and 服务等级协议(service level agreements (**SLAs**)), contracts that define the **expected performance** and **availability of a service**.

**Queueing delays** often account for a large part of the response time at high percentiles.

## 应对负载 - Handling Load

在有了描述和定义负载、性能的手段之后，终于来到正题，如何应对负载的不断增长，即使系统具有可扩展性。

1. **纵向扩展（scaling up）或 垂直扩展（vertical scaling）**：换具有更强大性能的机器。e.g. 大型机机器学习训练。
2. **横向扩展（scaling out）或 水平扩展（horizontal scaling）**：“并联”很多廉价机，分摊负载。e.g. 马斯克造火箭。

负载扩展的两种方式：

- **自动**
  - 如果负载不好预测且多变，则自动较好。坏处在于不易跟踪负载，容易抖动，造成资源浪费。
  
- **手动**
  - 如果负载容易预测且不长变化，最好手动。设计简单，且不容易出错。


针对不同应用场景：

首先，如果规模很小，尽量还是用性能好一点的机器，可以省去很多麻烦。

其次，可以上云，利用云的可扩展性。甚至如 Snowflake 等基础服务提供商也是 All In 云原生。

最后，实在不行再考虑自行设计可扩展的分布式架构。

两种服务类型：

- **无状态服务**
  - 比较简单，多台机器，外层罩一个 gateway 就行。
  
- **有状态服务**
  - 根据需求场景，如读写负载、存储量级、数据复杂度、响应时间、访问模式，来进行取舍，设计合乎需求的架构。


**不可能啥都要，没有万金油架构**！但同时：万变不离其宗，组成不同架构的原子设计模式是有限的，这也是本书稍后要论述的重点。

# 可维护性 - Maintainability

从软件的整个生命周期来看，维护阶段绝对占大头。

但大部分人都喜欢挖坑，不喜欢填坑。因此有必要，在刚开就把坑开的足够好。有三个原则：

- **_可维护性（Operability）_**
  - 便于运维团队无痛接手。
  
- **_简洁性（Simplicity）_**
  - 便于新手开发平滑上手：这需要一个合理的抽象，并尽量消除各种复杂度。如，层次化抽象。

- **_可演化性（Evolvability）_**
  - 便于后面需求快速适配：避免耦合过紧，将代码绑定到某种实现上。也称为**可扩展性（extensibility）**，**可修改性（modifiability）**  或**可塑性（plasticity）**。


## 可维护性 - Operability

有效的运维绝对是个高技术活：

1. 紧盯系统状态，出问题时快速恢复。
2. 恢复后，复盘问题，定位原因。
3. 定期对平台、库、组件进行更新升级。
4. 了解组件间相互关系，避免级联故障。
5. 建立自动化配置管理、服务管理、更新升级机制。
6. 执行复杂维护任务，如将存储系统从一个数据中心搬到另外一个数据中心。
7. 配置变更时，保证系统安全性。

系统具有良好的可维护性，意味着将**可定义**的维护过程编写**文档和工具**以自动化，从而解放出人力关注更高价值事情：

1. 友好的文档和一致的运维规范。
2. 细致的监控仪表盘(dashboards)、自检(self-checks)和报警(alerts)。
3. 通用的缺省配置(default configurations)。
4. 出问题时的自愈机制，无法自愈时允许管理员手动介入。
5. 将维护过程尽可能的自动化。
6. 避免单点依赖(single points of failure)，无论是机器还是人。

## 简洁性 - Simplicity

![recommand-book.png](img/ch01-book-software-design.jpeg)

推荐一本书：[A Philosophy of Software Design](https://book.douban.com/subject/30218046/) ，讲述在软件设计中如何定义、识别和降低复杂度。

复杂度表现：

1. 状态空间的膨胀explosion of the state space)。
2. 组件间的强耦合。
3. 不一致的术语和[命名](https://www.qtmuniao.com/2021/12/12/how-to-write-code-scrutinize-names/)。
4. 为了提升性能的 hack。
5. 随处可见的补丁（workaround）。

需求很简单，但不妨碍你实现的很复杂 ：过多的引入了**额外复杂度**（_accidental_ complexity
）——非问题本身决定的，而由实现所引入的复杂度。

通常是问题理解的不够本质，写出了“**流水账**”（没有任何**抽象，abstraction**）式的代码。

如果你为一个问题找到了合适的抽象，那么问题就解决了一半，如：

1. 高级语言隐藏了机器码、CPU 和系统调用细节。
2. SQL 隐藏了存储体系、索引结构、查询优化实现细节。

如何找到合适的抽象？

1. 从计算机领域常见的抽象中找。
2. 从日常生活中常接触的概念找。

总之，一个合适的抽象，要么是**符合直觉**的；要么是和你的读者**共享上下文**的。

## 可演化性 - Evolvability

系统需求没有变化，说明这个行业死了。

否则，需求一定是不断在变，引起变化的原因多种多样：

1. 对问题阈了解更全面
2. 出现了之前未考虑到的用例
3. 商业策略的改变
4. 客户要求新功能
5. 依赖平台的更迭
6. 合规性(compliance)要求
7. 体量的改变

应对之道：

- 项目管理上
  - 敏捷开发
- 系统设计上
  - 依赖前两点。合理抽象，合理封装，对修改关闭，对扩展开放。

# Chapter 2: Data Models and Query Languages

# 概要

本节围绕两个主要概念来展开。

如何分析一个**数据模型**：

1. 基本考察点：数据基本元素，和元素之间的对应关系（一对多，多对多）
2. 利用几种常用模型来比较：（最为流行的）关系模型，（树状的）文档模型，（极大自由度的）图模型。
3. schema 模式：强 Schema（写时约束）；弱 Schema（读时解析）

如何考量**查询语言**：

1. 如何与数据模型关联、匹配
2. 声明式（declarative）和命令式（imperative）

## 数据模型

> A **data model** is an [abstract model](https://en.wikipedia.org/wiki/Abstract_model) that organizes elements of [data](https://en.wikipedia.org/wiki/Data) and standardizes how they relate to one another and to the properties of real-world entities.
> —[https://en.wikipedia.org/wiki/Data_model](https://en.wikipedia.org/wiki/Data_model)

**数据模型**：如何组织数据，如何标准化关系，如何关联现实。

它既决定了我们构建软件的方式（**实现**），也左右了我们看待问题的角度（**认知**）。

作者开篇以计算机的不同抽象层次来让大家对**泛化的**数据模型有个整体观感。

大多数应用都是通过不同的数据模型层级累进构建的。

![ddia2-layered-data-models.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-layered-data-models.png)

每层模型核心问题：如何用下一层的接口来对本层进行建模？

1. 作为**应用开发者，** 你将现实中的具体问题抽象为一组对象、**数据结构（data structure）** 以及作用于其上的 API。
2. 作为**数据库管理员（DBA）**，为了持久化上述数据结构，你需要将他们表达为通用的**数据模型（data model）**，如文档数据库中的 XML/JSON、关系数据库中的表、图数据库中的图。
3. 作为**数据库系统开发者**，你需要将上述数据模型组织为内存中、硬盘中或者网络中的**字节（Bytes）** 流，并提供多种操作数据集合的方法。
4. 作为**硬件工程师**，你需要将字节流表示为二极管的电位（内存）、磁场中的磁极（磁盘）、光纤中的光信号（网络）。

> 在每一层，通过对外暴露简洁的**数据模型**，我们**隔离**和**分解**了现实世界的**复杂度**。

这也反过来说明了，好的数据模型需有两个特点：

1. 简洁直观
2. 具有组合性

第二章首先探讨了关系模型、文档模型及其对比，其次是相关查询语言，最后探讨了图模型。

# 关系模型与文档模型

## 关系模型

关系模型无疑是当今最流行的数据库模型。

关系模型是 [埃德加·科德（](https://zh.wikipedia.org/wiki/%E5%9F%83%E5%BE%B7%E5%8A%A0%C2%B7%E7%A7%91%E5%BE%B7)[E. F. Codd](https://en.wikipedia.org/wiki/E._F._Codd)）于 1969 年首先提出，并用“[科德十二定律](https://zh.wikipedia.org/wiki/%E7%A7%91%E5%BE%B7%E5%8D%81%E4%BA%8C%E5%AE%9A%E5%BE%8B)”来解释。但是商业落地的数据库基本没有能完全遵循的，因此关系模型后来通指这一类数据库。特点如下：

1. 将数据以**关系**呈现给用户（比如：一组包含行列的二维表）。
2. 提供操作数据集合的**关系算子**(operators such as Join, Agg, etc)。

**常见分类**

1. 事务型（TP）：银行交易、火车票
2. 分析型（AP）：数据报表、监控表盘
3. 混合型（HTAP）：

关系模型诞生很多年后，虽有不时有各种挑战者（比如上世纪七八十年代的**网状模型** network model 和**层次模型** hierarchical model），但始终仍未有根本的能撼动其地位的新模型。

直到近十年来，随着移动互联网的普及，数据爆炸性增长，各种处理需求越来越精细化，催生了数据模型的百花齐放。

## NoSQL 的诞生

NoSQL（最初表示 Non-SQL，后来有人转解为 Not only SQL），是对不同于传统的关系数据库的数据库管理系统的统称。根据 [DB-Engines 排名](https://db-engines.com/en/ranking)，现在最受欢迎的 NoSQL 前几名为：MongoDB，Redis，ElasticSearch，Cassandra。

其催动因素有：

1. 处理更大数据集(very large datasets)：更强伸缩性(scalability)、更高吞吐量(write throughput)
2. 开源免费的兴起(open source)：冲击了原来把握在厂商的标准
3. 特化的查询操作(Specialized query operations)：关系数据库难以支持的，比如图中的多跳分析
4. 表达能力更强(dynamic and expressive data model)：关系模型约束太严，限制太多

## 面向对象和关系模型的不匹配

核心冲突在于面向对象的**嵌套性**和关系模型的**平铺性**，当然有 ORM 框架可以帮我们搞定这些事情，但仍是不太方便。

![ddia2-bill-resume.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-fig01.png)

换另一个角度来说，关系模型很难直观的表示**一对多**的关系。比如简历上，一个人可能有多段教育经历和多段工作经历。

**文档模型**：使用 Json 和 XML 的天然嵌套。

**关系模型**：使用 SQL 模型就得将职位、教育单拎一张表，然后在用户表中使用外键(foreign key)关联。

在简历的例子中，文档模型还有几个优势：

1. **模式灵活**：可以动态增删字段，如工作经历。
2. **更好的局部性**：一个人的所有属性被集中访问的同时，也被集中存储。
3. **结构表达语义**：简历与联系信息、教育经历、职业信息等隐含一对多的树状关系可以被 JSON 的树状结构明确表达出来。

## 多对一和多对多

是一个对比各种数据模型的切入角度。

region 在存储时，为什么不直接存储纯字符串：“Greater Seattle Area”，而是先存为 region_id → region name，其他地方都引用 region_id？

1. **统一样式**：所有用到相同概念的地方都有相同的拼写和样式
2. **避免歧义**：可能有同名地区
3. **易于修改**：如果一个地区改名了，我们不用去逐一修改所有引用他的地方
4. **本地化支持**：如果翻译成其他语言，可以只翻译名字表。
5. **更好搜索**：列表可以关联地区，进行树形组织

类似的概念还有：面向抽象编程，而非面向细节。

关于用 ID 还是文本，作者提到了一点：ID 对人类是**无意义**的，无意义的意味着不会随着现实世界的将来的改变而改动。

这在关系数据库表设计时需要考虑，即如何控制**冗余（duplication）**。会有几种**范式（normalization）** 来消除冗余。

文档型数据库很擅长处理一对多的树形关系，却不擅长处理多对多的图形关系。如果其不支持 Join，则处理多对多关系的复杂度就从数据库侧移动到了应用侧。

如，多个用户可能在同一个组织工作过。如果我们想找出在同一个学校和组织工作过的人，如果数据库不支持 Join，则需要在应用侧进行循环遍历来 Join。

![ddia2-mul-to-mul.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-fig02.png)

文档 vs 关系

1. 对于一对多关系，文档型数据库将嵌套数据放在父节点中，而非单拎出来放另外一张表。
2. 对于多对一和多对多关系，本质上，两者都是使用外键（文档引用）进行索引。查询时需要进行 join 或者动态跟随。

## 文档模型是否在重复历史？

### 层次模型 **（hierarchical model）**

20 世纪 70 年代，IBM 的信息管理系统 IMS。

> A **hierarchical database model** is a [data model](https://en.wikipedia.org/wiki/Data_model) in which the data are organized into a [tree](https://en.wikipedia.org/wiki/Tree_data_structure)-like structure. The data are stored as **records** which are connected to one another through **links.** A record is a collection of fields, with each field containing only one value. The **type** of a record defines which fields the record contains. — wikipedia

几个要点：

1. 树形组织，每个子节点只允许有一个父节点
2. 节点存储数据，节点有类型
3. 节点间使用类似指针方式连接

可以看出，它跟文档模型很像，也因此很难解决多对多的关系，并且不支持 Join。

为了解决层次模型的局限，人们提出了各种解决方案，最突出的是：

1. 关系模型
2. 网状模型

### 网状模型（network model）

network model 是 hierarchical model 的一种扩展：允许一个节点有多个父节点。它被数据系统语言会议（CODASYL）的委员会进行了标准化，因此也被称为 CODASYL 模型。

多对一和多对多都可以由路径来表示。访问记录的唯一方式是顺着元素和链接组成的链路进行访问，这个链路叫**访问路径** （access path）。难度犹如在 n-维空间中进行导航。

内存有限，因此需要严格控制遍历路径。并且需要事先知道数据库的拓扑结构，这就意味着得针对不同应用写大量的专用代码。

### 关系模型

在关系模型中，数据被组织成**元组（tuples）**，进而集合成**关系（relations）**；在 SQL 中分别对应行（rows）和表（tables）。

不知道大家好奇过没，明明看起来更像表模型，为什叫**关系模型**？
表只是一种实现。
关系（relation）的说法来自集合论，指的是几个集合的笛卡尔积的子集。
R ⊆ （D1×D2×D3 ··· ×Dn）（关系用符号 R 表示，属性用符号 Ai 表示，属性的定义域用符号 Di 表示）

其主要目的和贡献在于提供了一种**声明式**的描述数据和构建查询的方法。

即，相比网络模型，关系模型的查询语句和执行路径相解耦，**查询优化器**（Query Optimizer 自动决定执行顺序、要使用的索引），即将逻辑和实现解耦。

举个例子：如果想使用新的方式对你的数据集进行查询，你只需要在新的字段上建立一个索引。那么在查询时，你并不需要改变的你用户代码，查询优化器便会动态的选择可用索引。

## 文档型 vs 关系型

根据数据类型来选择数据模型

|            | 文档型                                                       | 关系型                                      |
| ---------- | ------------------------------------------------------------ | ------------------------------------------- |
| 对应关系   | 数据有天然的一对多、树形嵌套关系，如简历。                   | 通过外键 + Join 可以处理 多对一，多对多关系 |
| 代码简化   | 数据具有文档结构，则文档模型天然合适，用关系模型会使得建模繁琐、访问复杂。但不宜嵌套太深，因为只能手动指定访问路径，或者范围遍历 | 主键，索引，条件过滤                        |
| Join 支持  | 对 Join 支持的不太好                                         | 支持的还可以，但 Join 的实现会有很多难点    |
| 模式灵活性 | 弱 schema，支持动态增加字段                                  | 强 schema，修改 schema 代价很大             |
| 访问局部性 | 1. 一次性访问整个文档，较优 <br/>2. 只访问文档一部分，较差   | 分散在多个表中                              |

对于高度关联的数据集，使用文档型表达比较奇怪，使用关系型可以接受，使用图模型最自然。

### 文档模型中 Schema 的灵活性

说文档型数据库是 schemaless 不太准确，更贴切的应该是 **schema-on-read。**

| 数据模型        |                                        | 编程语言 |                    | 性能 & 空间                                                  |
| --------------- | -------------------------------------- | -------- | ------------------ | ------------------------------------------------------------ |
| schema-on-read  | 写入时不校验，而在读取时进行动态解析。 | 弱类型   | 动态，在运行时解析 | 读取时动态解析，性能较差。写入时无法确定类型，无法对齐，空间利用率较差。 |
| schema-on-write | 写入时校验，数据对齐到 schema          | 强类型   | 静态，编译时确定   | 性能和空间使用都较优。                                       |

文档型数据库使用场景特点：

1. 有多种类型的数据，但每个放一张表又不合适。
2. 数据类型和结构由外部决定，你没办法控制数据的变化。

### 查询时的数据局部性

如果你同时需要文档中所有内容，把文档顺序存储，访问会效率比较高。

但如果你只需要访问文档中的某些字段，则文档仍需要将文档全部加载出。

但运用这种局部性不局限于文档型数据库。不同的数据库，会针对不同场景，调整数据物理分布以适应常用访问模式的局部性。

- Spanner 中允许表被声明为嵌入到父表中——常用关联内嵌（获得类似文档模型的结构）
- HBase 和 Cassandra 使用列族(column-family)来聚集数据——分析型
- 图数据库中，将点和出边存在一个机器上——图遍历

### 关系型和文档型的融合

- MySQL 和 PostgreSQL 开始支持 JSON
  - 原生支持 JSON 可以理解为，MySQL 可以理解 JSON 类型。如 Date 这种复杂格式一样，可以让某个字段为 JSON 类型、可以修改 Join 字段的某个属性、可以在 Json 字段中某个属性建立索引。

- RethinkDB 在查询中支持 relational-link Joins

科德（Codd）：**nonsimple domains**，记录中的值除了简单类型（数字、字符串），还可以一个嵌套关系（表）。这很像 SQL 对 XML、JSON 的支持。

# 数据查询语言

获取动物表中所有鲨鱼类动物。

```jsx
function getSharks() {
  var sharks = [];
  for (var i = 0; i < animals.length; i++) {
    if (animals[i].family === 'Sharks') {
      sharks.push(animals[i]);
    }
  }
  return sharks;
}
```

```sql
SELECT * FROM animals WHERE family = 'Sharks';
```

|          | 声明式（declarative）语言                                    | 命令式（imperative）语言                                     |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 概念     | 描述控制逻辑而非执行流程                                     | 描述命令的执行过程，用一系列语句来不断改变状态               |
| 举例     | SQL，CSS，XSL                                                | IMS，CODASYL，通用语言如 C，C++，JS                          |
| 抽象程度 | 高                                                           | 低                                                           |
| 解耦程度 | 与实现解耦。 <br/>可以持续优化查询引擎性能；                 | 与实现耦合较深。                                             |
| 解析执行 | 词法分析 → 语法分析 → 语义分析 <br/>生成执行计划 → 执行计划优化 | 词法分析 → 语法分析 → 语义分析 <br/>中间代码生成 → 代码优化 → 目标代码生成 |
| 多核并行 | 声明式更具多核潜力，给了更多运行时优化空间                   | 命令式由于指定了代码执行顺序，编译时优化空间较小。           |

> Q：相对声明式语言，命令式语言有什么优点？
>
> 1.  当描述的目标变得复杂时，声明式表达能力不够。
> 2.  实现命令式的语言往往不会和声明式那么泾渭分明，通过合理抽象，通过一些编程范式（函数式），可以让代码兼顾表达力和清晰性。

A declarative query language is attractive because it is typically more concise and easier to work with than an imperative API. But more importantly, it also hides implementation details of the database engine, which makes it possible for the database system to introduce performance improvements without requiring any changes to queries.

## 数据库以外：Web 中的声明式

**需求**：选中页背景变蓝。

```html
<ul>
  <li class="selected">
    <p>Sharks</p>
    <ul>
      <li>Great White Shark</li>
      <li>Tiger Shark</li>
      <li>Hammerhead Shark</li>
    </ul>
  </li>
  <li>
    <p>Whales</p>
    <ul>
      <li>Blue Whale</li>
      <li>Humpback Whale</li>
      <li>Fin Whale</li>
    </ul>
  </li>
</ul>
```

如果使用 CSS，则只需（CSS selector）：

```css
li.selected > p {
  background-color: blue;
}
```

如果使用 XSL，则只需（XPath selector）：

```css
<xsl:template match="li[@class='selected']/p">
	<fo:block background-color="blue">
		<xsl:apply-templates/>
	</fo:block>
</xsl:template>
```

但如果使用 JavaScript（而不借助上述 selector 库）：

```jsx
var liElements = document.getElementsByTagName('li');
for (var i = 0; i < liElements.length; i++) {
  if (liElements[i].className === 'selected') {
    var children = liElements[i].childNodes;
    for (var j = 0; j < children.length; j++) {
      var child = children[j];
      if (child.nodeType === Node.ELEMENT_NODE && child.tagName === 'P') {
        child.setAttribute('style', 'background-color: blue');
      }
    }
  }
}
```

## MapReduce 查询

**Google 的 MapReduce 模型**

1. 借鉴自函数式编程(functional programming)。
2. 一种相当简单的编程模型，或者说原子的抽象，现在不太够用。
3. 但在大数据处理工具匮乏的蛮荒时代（03 年以前），谷歌提出的这套框架相当有开创性。

![how maprduce works](/Users/zhuoshuyang/Desktop/ddia/img/ch02-how-mr-works.png)

**MongoDB 的 MapReduce 模型**

MongoDB 使用的 MapReduce 是一种介于

1. **声明式**：用户不必显式定义数据集的遍历方式、shuffle 过程等执行过程。
2. **命令式**：用户又需要定义针对单条数据的执行过程。

两者间的混合数据模型。

**需求**：统计每月观察到鲨类鱼的次数。

**查询语句**：

**PostgresSQL**

```sql
SELECT date_trunc('month', observation_timestamp) AS observation_month,
	sum(num_animals) AS total_animals
FROM observations
WHERE family = 'Sharks' GROUP BY observation_month;
```

**MongoDB**

```jsx
db.observations.mapReduce(
  function map() {
    // 2. 对所有符合条件 doc 执行 map
    var year = this.observationTimestamp.getFullYear();
    var month = this.observationTimestamp.getMonth() + 1;
    emit(year + '-' + month, this.numAnimals); // 3. 输出一个 kv pair
  },
  function reduce(key, values) {
    // 4. 按 key 聚集
    return Array.sum(values); // 5. 相同 key 加和
  },
  {
    query: { family: 'Sharks' }, // 1. 筛选
    out: 'monthlySharkReport', // 6. reduce 结果集
  }
);
```

上述语句在执行时，经历了：筛选（filter） → 遍历并执行 map → 对输出按 key 聚集（shuffle）→ 对聚集的数据逐一 reduce → 输出结果集。

MapReduce 一些特点：

1. **要求 Map 和 Reduce 是纯函数**。即无任何副作用，在任意地点、以任意次序执行任何多次，对相同的输入都能得到相同的输出。因此容易并发调度。
2. **非常底层、但表达力强大的编程模型**。可基于其实现 SQL 等高级查询语言，如 Hive。

但要注意：

1. 不是所有的分布式 SQL 都基于 MapReduce 实现。
2. 不是只有 MapReduce 才允许嵌入通用语言（如 js）模块。
3. MapReduce 是有一定**理解成本**的，需要非常熟悉其执行原理才能让两个函数紧密配合。

MongoDB 2.2+ 进化版，_aggregation pipeline:_

```jsx
db.observations.aggregate([
  { $match: { family: 'Sharks' } },
  {
    $group: {
      _id: {
        year: { $year: '$observationTimestamp' },
        month: { $month: '$observationTimestamp' },
      },
      totalAnimals: { $sum: '$numAnimals' },
    },
  },
]);
```

# 图模型

- 文档模型的适用场景？
  - 你的建模场景中存在着大量**一对多**（one-to-many）的关系。

- 图模型的适用场景？
  - 你的建模场景中存在大量的**多对多**（many-to-many）的关系。


## 基本概念

图数据模型（属性图）的基本概念一般有三个：**点**，**边**和附着于两者之上的**属性**。

常见的可以用图建模的场景：

| 例子     | 建模                        | 应用                                            |
| -------- | --------------------------- | ----------------------------------------------- |
| 社交图谱 | 人是点，follow 关系是边     | 六度分隔(Six Degrees of Separation)，信息流推荐 |
| 互联网   | 网页是点，链接关系是边      | PageRank                                        |
| 路网     | 交通枢纽是点，铁路/公路是边 | 路径规划，导航最短路径                          |
| 洗钱     | 账户是点，转账关系是边      | 判断是否有环                                    |
| 知识图谱 | 概念是点，关联关系是边      | 启发式问答                                      |

> 同构（homogeneous）数据和异构数据
> 图模型中的点变可以像关系中的表一样都具有相同类型；但是，一张图中的点和变也可以具有不同类型，能够容纳异构数据是图模型善于处理多对多关系的一大原因。

本节都会以下图为例，它表示了一对夫妇，来自美国爱达荷州的 Lucy 和来自法国 的 Alain：他们已婚，住在伦敦。

![example](/Users/zhuoshuyang/Desktop/ddia/img/ch02-fig05.png)

有多种对图的建模方式：

1. **属性图（property graph）**：比较主流，如 Neo4j、Titan、InfiniteGraph
2. **三元组（triple-store）**：如 Datomic、AllegroGraph

## 属性图（PG，Property Graphs）

| 点 (vertices, nodes, entities) | 边 (edges, relations, arcs) |
| ------------------------------ | --------------------------- |
| 全局唯一 ID                    | 全局唯一 ID                 |
| 出边集合                       | 起始点                      |
| 入边集合                       | 终止点                      |
| 属性集（kv 对表示）            | 属性集（kv 对表示）         |
| 表示点类型的 type？            | 表示边类型的 label          |

> Q：有一个疑惑点，为什么书中对于 PG 点的定义中没有 Type？
> 因为属性图具体实现时也可以分为强类型和弱类型，NebulaGraph 是强类型，好处在于效率高，但灵活性差；Neo4j 是弱类型。书中应该是用的弱类型，此时每个点都是一组属性集，不需要 type。

如果感觉不直观，可以使用我们熟悉的 SQL 语义来构建一个图模型，如下图。（Facebook TAO 论文中的单机存储引擎便是 MySQL）

```sql
// 点表
CREATE TABLE vertices (
	vertex_id integer PRIMARYKEY, properties json
);

// 边表
CREATE TABLE edges (
	edge_id integer PRIMARY KEY,
	tail_vertex integer REFERENCES vertices (vertex_id),
	head_vertex integer REFERENCES vertices (vertex_id),
	label text,
	properties json
);

// 对点的反向索引，图遍历时用。给定点，找出点的所有入边和出边。
CREATE INDEX edges_tails ON edges (tail_vertex);
CREATE INDEX edges_heads ON edges (head_vertex);
```

图是一种很灵活的建模方式：

1. 任何两点间都可以插入边，没有任何模式限制。
2. 对于任何顶点都可以高效（思考：如何高效？）找到其入边和出边，从而进行图遍历。
3. 使用多种**标签**来标记不同类型边（关系）。

相对于关系型数据来说，**可以在同一个图中保存异构类型的数据和关系，给了图极大的表达能力！**

这种表达能力，根据图中的例子，包括：

1. 对同样的概念，可以用不同结构表示。如不同国家的行政划分。
2. 对同样的概念，可以用不同粒度(granularity)表示。比如 Lucy 的现居住地和诞生地。
3. 可以很自然的进行演化。

将异构的数据容纳在一张图中，可以通过**图遍历**，轻松完成关系型数据库中需要**多次 Join** 的操作。

## Cypher 查询语言

Cypher 是 Neo4j 创造的一种查询语言。

Cypher 和 Neo 名字应该都是来自《黑客帝国》（The Matrix）。想想 Oracle。

Cypher 的一大特点是可读性强，尤其在表达路径模式（Path Pattern）时。

结合前图，看一个 Cypher 插入语句的例子：

```sql
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country'  }),
  (Idaho:Location    {name:'Idaho',         type:'state'    }),
  (Lucy:Person       {name:'Lucy' }),
  (Idaho) -[:WITHIN]->  (USA)  -[:WITHIN]-> (NAmerica),
  (Lucy)  -[:BORN_IN]-> (Idaho)
```

如果我们要进行一个这样的查询：找出所有从美国移居到欧洲的人名。

转化为图语言，即为：给定条件，BORN_IN 指向美国的地点，并且 LIVING_IN 指向欧洲的地点，找到所有符合上述条件的点，并且返回其名字属性。

用 Cypher 语句可表示为：

```sql
MATCH
  (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
  (person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'})
RETURN person.name
```

注意到：

1. 点 `()`，边 `-[]→`，标签\类型 `：`，属性 `{}`。
2. 名字绑定或者说变量：`person`
3. 0 到多次通配符： `*0...`

正如声明式查询语言的一贯特点，你只需描述问题，不必担心执行过程。但与 SQL 的区别在于，SQL 基于**关系代数**，Cypher 类似**正则表达式**。

无论是 BFS、DFS 还是剪枝等实现细节，一般来说（但是不同厂商通常都会有不同的最佳实践），用户都不需要关心。

## 使用 SQL 进行图查询

前面看到可以用 SQL 存储点和边，以表示图。

那可以用 SQL 进行图查询吗？

Oracle 的 [PGQL](https://docs.oracle.com/en/database/oracle/property-graph/20.4/spgdg/property-graph-query-language-pgql.html)：

```sql
CREATE PROPERTY GRAPH bank_transfers
     VERTEX TABLES (persons KEY(account_number))
     EDGE TABLES(
                  transactions KEY (from_acct, to_acct, date, amount)
                  SOURCE KEY (from_account) REFERENCES persons
                  DESTINATION KEY (to_account) REFERENCES persons
                  PROPERTIES (date, amount)
       )
```

其中有一个难点，就是如何表达图中的路径模式（graph pattern），如**多跳查询**，对应到 SQL 中，就是不确定次数的 Join：

```sql
() -[:WITHIN*0..]-> ()
```

使用 SQL:1999 中 recursive common table expressions（PostgreSQL, IBM DB2, Oracle, and SQL Server 支持）的可以满足。但是，相当冗长和笨拙。

## **Triple-Stores and SPARQL**

**Triple-Stores**，可以理解为三元组存储，即用三元组存储图。

![ddia2-triple-store.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-spo.png)

其含义如下：

| Subject   | 对应图中的一个点                                             |
| --------- | ------------------------------------------------------------ |
| Object    | 1. 一个原子数据(primitive datatype)，如 string 或者 number。<br/>2. 另一个 Subject。 |
| Predicate | 1. 如果 Object 是原子数据，则 <Predicate, Object> 对应点附带的 KV 对。<br/>2. 如果 Object 是另一个 Object，则 Predicate 对应图中的边。 |

仍是上边例子，用 Turtle triples (一种 **Triple-Stores** 语法）**表达为**：

```scheme
@prefix : <urn:example:>.
_:lucy     a       :Person.
_:lucy     :name   "Lucy".
_:lucy     :bornIn _:idaho.
_:idaho    a       :Location.
_:idaho    :name   "Idaho".
_:idaho    :type   "state".
_:idaho    :within _:usa.
_:usa      a       :Location
_:usa      :name   "United States"
_:usa      :type   "country".
_:usa      :within _:namerica.
_:namerica a       :Location.
_:namerica :name   "North America".
_:namerica :type   "continent".
```

一种更紧凑的写法：

```scheme
@prefix : <urn:example:>.
_:lucy     a: Person;   :name "Lucy";          :bornIn _:idaho
_:idaho    a: Location; :name "Idaho";         :type "state";     :within _:usa.
_:usa      a: Location; :name "United States"; :type "country";   :within _:namerica.
_:namerica a :Location; :name "North America"; :type "continent".
```

### 语义网（The **Semantic Web**）

万维网之父 Tim Berners Lee 于 1998 年提出，知识图谱前身。其目的在于对网络中的资源进行结构化，从而让计算机能够**理解**网络中的数据。即不是以文本、二进制流等非结构数据呈现内容，而是以某种标准结构化互联网上通过超链接而关联的数据。

**语义**：提供一种统一的方式对所有资源进行描述和**结构化**（机器可读）。

**网**：将所有资源勾连起来。

下面是**语义网技术栈**（Semantic Web Stack）：

![ddia2-rdf.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-semantic-web-stack.png)

其中 **RDF** （ *ResourceDescription Framework，资源描述框架* ）提供了一种结构化网络中数据的标准。使发布到网络中的任何资源（文字、图片、视频、网页），都能以统一的形式被计算机理解。从另一个角度来理解，即，不需要资源使用方通过深度学习等方式来抽取语义，而是靠资源提供方通过 RDF 主动提供其资源语义。

感觉有点理想主义，但互联网、开源社区都是靠这种理想主义、分享精神发展起来的！

虽然语义网没有发展起来，但是其**中间数据交换**格式 RDF 所定义的 SPO 三元组 (Subject-Predicate-Object) 却是一种很好用的数据模型，也就是上面提到的 **Triple-Stores。**

### RDF 数据模型

上面提到的 Turtle 语言（SPO 三元组）是一种简单易读的描述 RDF 数据的方式，RDF 也可以基于 XML 表示，但是要冗余难读的多（嵌套太深）：

```xml
<rdf:RDF xmlns="urn:example:"
	xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
	<Location rdf:nodeID="idaho">
		<name>Idaho</name>
		<type>state</type>
		<within>
			<Location rdf:nodeID="usa">
				<name>United States</name>
				<type>country</type>
				<within>
					<Location rdf:nodeID="namerica">
						<name>North America</name>
						<type>continent</type>
			    </Location>
		    </within>
      </Location>
    </within>
	</Location>
	<Person rdf:nodeID="lucy">
		<name>Lucy</name>
		<bornIn rdf:nodeID="idaho"/>
	</Person>
</rdf:RDF>
```

为了标准化和去除二义性，一些看起来比较奇怪的点是：无论 subject，predicate 还是 object 都是由 URI 定义，如

```json
lives_in 会表示为 <http://my-company.com/namespace#lives_in>
```

其前缀只是一个 namespace，让定义唯一化，并且在网络上可访问。当然，一个简化的方法是可以在文件头声明一个公共前缀。

### **SPARQL 查询语言**

有了语义网，自然需要在语义网中进行遍历查询，于是有了 RDF 的查询语言：SPARQL Protocol and RDF Query Language, pronounced“sparkle.”

```
PREFIX : <urn:example:>
SELECT ?personName WHERE {
  ?person :name ?personName.
  ?person :bornIn  / :within* / :name "United States".
  ?person :livesIn / :within* / :name "Europe".
}
```

他是 Cypher 的前驱，因此结构看起来很像：

```
(person) -[:BORN_IN]-> () -[:WITHIN*0..]-> (location)   # Cypher
?person   :bornIn /        :within*        ?location.   # SPARQL
```

但 **SPARQL** 没有区分边和属性的关系，都用了 Predicates。

```
(usa {name:'United States'})   # Cypher
?usa :name "United States".    # SPARQL
```

虽然语义网没有成功落地，但其技术栈影响了后来的知识图谱和图查询语言。

### 图模型和网络模型

图模型是网络模型旧瓶装新酒吗？

否，他们在很多重要的方面都不一样。

| 模型     | 图模型（Graph Model）                                     | 网络模型（Network Model）                              |
| -------- | --------------------------------------------------------- | ------------------------------------------------------ |
| 连接方式 | 任意两个点之间都有可以有边                                | 指定了嵌套约束                                         |
| 记录查找 | 1. 使用全局 ID <br/>2. 使用属性索引。<br/>3. 使用图遍历。 | 只能使用路径查询                                       |
| 有序性   | 点和边都是无序的                                          | 记录的孩子们是有序集合，在插入时需要考虑维持有序的开销 |
| 查询语言 | 即可命令式，也可以声明式                                  | 命令式的                                               |

## 查询语言前驱：Datalog

有点像 triple-store，但是变了下次序：(_subject_, _predicate_, _object_) → _predicate_(_subject_, _object_).
之前数据用 Datalog 表示为：

```
name(namerica, 'North America').
type(namerica, continent).

name(usa, 'United States').
type(usa, country).
within(usa, namerica).

name(idaho, 'Idaho').
type(idaho, state).
within(idaho, usa).

name(lucy, 'Lucy').
born_in(lucy, idaho).
```

查询从*美国迁移到欧洲的人*可以表示为：

```
within_recursive(Location, Name) :- name(Location, Name). /* Rule 1 */
within_recursive(Location, Name) :- within(Location, Via), /* Rule 2 */
                                    within_recursive(Via, Name).

migrated(Name, BornIn, LivingIn) :- name(Person, Name), /* Rule 3 */
                                    born_in(Person, BornLoc),
                                    within_recursive(BornLoc, BornIn),
                                    lives_in(Person, LivingLoc),
                                    within_recursive(LivingLoc, LivingIn).
?- migrated(Who, 'United States', 'Europe'). /* Who = 'Lucy'. */
```

1. 代码中以大写字母开头的元素是**变量**，字符串、数字或以小写字母开头的元素是**常量**。下划线（\_）被称为匿名变量
2. 可以使用基本 Predicate 自定义 Predicate，类似于使用基本函数自定义函数。
3. 逗号连接的多个谓词表达式为且的关系。

![ddia2-triple-store-query.png](/Users/zhuoshuyang/Desktop/ddia/img/ch02-06.png)

基于集合的逻辑运算：

1. 根据基本数据子集选出符合条件集合。
2. 应用规则，扩充原集合。
3. 如果可以递归，则递归穷尽所有可能性。

Prolog（Programming in Logic 的缩写）是一种逻辑编程语言。它创建在逻辑学的理论基础之上。

## 参考

1. 声明式 (declarative) vs 命令式 (imperative)**：**[https://lotabout.me/2020/Declarative-vs-Imperative-language/](https://lotabout.me/2020/Declarative-vs-Imperative-language/)
2. **[SimmerChan](https://www.zhihu.com/people/simmerchan)** 知乎专栏，知识图谱，语义网，RDF：[https://www.zhihu.com/column/knowledgegraph](https://www.zhihu.com/column/knowledgegraph)
3. MySQL 为什么叫“关系”模型：[https://zhuanlan.zhihu.com/p/64731206](https://zhuanlan.zhihu.com/p/64731206)

# DDIA 逐章精读（三）：存储和查询

第二章讲了上层抽象：数据模型和查询语言。
本章下沉一些，聚焦数据库底层如何处理查询和存储。这其中，有个**逻辑链条**：

> 使用场景 → 查询类型 → 存储格式。

查询类型主要分为两大类：

| 引擎类型 | 请求数量               | 数据量                   | 瓶颈           | 存储格式     | 用户                         | 场景举例 | 产品举例   |
| -------- | ---------------------- | ------------------------ | -------------- | ------------ | ---------------------------- | -------- | ---------- |
| OLTP     | 相对频繁，侧重在线交易 | 总体和单次查询都相对较小 | Disk Seek      | 多用行存     | 比较普遍，一般应用用的比较多 | 银行交易 | MySQL      |
| OLAP     | 相对较少，侧重离线分析 | 总体和单次查询都相对巨大 | Disk Bandwidth | 列存逐渐流行 | 多为商业用户                 | 商业分析 | ClickHouse |

其中，OLTP 侧，常用的存储引擎又有两种流派：

| 流派               | 主要特点                                             | 基本思想         | 代表                                             |
| ------------------ | ---------------------------------------------------- | ---------------- | ------------------------------------------------ |
| log-structured 流  | 只允许追加，所有修改都表现为文件的追加和文件整体增删 | 变随机写为顺序写 | Bitcask、LevelDB、RocksDB、Cassandra、Lucene     |
| update-in-place 流 | 以页（page）为粒度对磁盘数据进行修改                 | 面向页、查找树   | B 族树，所有主流关系型数据库和一些非关系型数据库 |

此外，针对 OLTP，还探索了常见的建索引的方法，以及一种特殊的数据库——全内存数据库。

对于数据仓库，本章分析了它与 OLTP 的主要不同之处。数据仓库主要侧重于聚合查询，需要扫描很大量的数据，此时，索引就相对不太有用。需要考虑的是存储成本、带宽优化等，由此引出列式存储。

# 驱动数据库的底层数据结构

本节由一个 shell 脚本出发，到一个相当简单但可用的存储引擎 Bitcask，然后引出 LSM-tree，他们都属于日志流范畴。之后转向存储引擎另一流派——B 族树，之后对其做了简单对比。最后探讨了存储中离不开的结构——索引。

首先来看，世界上“最简单”的数据库，由两个 Bash 函数构成：

```bash
#!/bin/bash
db_set () {
	echo "$1,$2" >> database
}

db_get () {
	grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

这两个函数实现了一个基于字符串的 KV 存储（只支持 get/set，不支持 delete）：

```bash
$ db_set 123456 '{"name":"London","attractions":["Big Ben","London Eye"]}'
$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'
$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
```

来分析下它为什么 work，也反映了日志结构存储的最基本原理：

1. set：在文件末尾追加一个 KV 对。
2. get：匹配所有 Key，返回最后（也即最新）一条 KV 对中的 Value。

可以看出：写很快，但是读需要全文逐行扫描，会慢很多。典型的以读换写。为了加快读，我们需要构建**索引**：一种允许基于某些字段查找的额外数据结构。

索引从原数据中构建，只为加快查找。因此索引会耗费一定额外空间，和插入时间（每次插入要更新索引），即，重新以空间和写换读取。

这便是数据库存储引擎设计和选择时最常见的**权衡（trade off）**：

1. 恰当的**存储格式**能加快写（日志结构），但是会让读取很慢；也可以加快读（查找树、B 族树），但会让写入较慢。
2. 为了弥补读性能，可以构建索引。但是会牺牲写入性能和耗费额外空间。

存储格式一般不好动，但是索引构建与否，一般交予用户选择。

## 哈希索引

本节主要基于最基础的 KV 索引。

依上小节的例子，所有数据顺序追加到磁盘上。为了加快查询，我们在内存中构建一个哈希索引：

1. Key 是查询 Key
2. Value 是 KV 条目的起始位置和长度。

![ddia-3-1-hash-map-csv.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig01.png)

看来很简单，但这正是 [Bitcask](https://docs.riak.com/riak/kv/2.2.3/setup/planning/backend/bitcask/index.html 'Bitcask') 的基本设计，但关键是，他 Work（在小数据量时，即所有 key 都能存到内存中时）：能提供很高的读写性能：

1. 写：文件追加写。
2. 读：一次内存查询，一次磁盘 seek；如果数据已经被缓存，则 seek 也可以省掉。

如果你的 key 集合很小（意味着能全放内存），但是每个 key 更新很频繁，那么 Bitcask 便是你的菜。举个栗子：频繁更新的视频播放量，key 是视频 url，value 是视频播放量。

> 但有个很重要问题，单个文件越来越大，磁盘空间不够怎么办？
>
> 在文件到达一定尺寸后，就新建一个文件，将原文件变为只读。同时为了回收多个 key 多次写入的造成的空间浪费，可以将只读文件进行紧缩（compact），将旧文件进行重写，挤出“水分”（被覆写的数据）以进行垃圾回收。

![ddia-3-3-compaction-sim.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig03.png)

当然，如果我们想让其**工业可用**，还有很多问题需要解决：

1. **文件格式**。对于**日志**来说，CSV 不是一种紧凑的数据格式，有很多空间浪费。比如，可以用 length + record bytes。
2. **记录删除**。之前只支持 put\get，但实际还需要支持 delete。但日志结构又不支持更新，怎么办呢？一般是写一个特殊标记（比如墓碑记录，tombstone）以表示该记录已删除。之后 compact 时真正删除即可。
3. **宕机恢复**。在机器重启时，内存中的哈希索引将会丢失。当然，可以全盘扫描以重建，但通常一个小优化是，对于每个 segment file，将其索引条目和数据文件一块持久化，重启时只需加载索引条目即可。
4. **记录写坏、少写**。系统任何时候都有可能宕机，由此会造成记录写坏、少写。为了识别错误记录，我们需要增加一些校验字段，以识别并跳过这种数据。为了跳过写了部分的数据，还要用一些特殊字符来标识记录间的边界。
5. **并发控制**。由于只有一个活动（追加）文件，因此写只有一个天然并发度。但其他的文件都是不可变的（compact 时会读取然后生成新的），因此读取和紧缩可以并发执行。

乍一看，基于日志的存储结构存在折不少浪费：需要以追加进行更新和删除。但日志结构有几个原地更新结构无法做的优点：

1. **以顺序写代替随机写**。对于磁盘和 SSD，顺序写都要比随机写快几个数量级。
2. **简易的并发控制**。由于大部分的文件都是**不可变（immutable）** 的，因此更容易做并发读取和紧缩。也不用担心原地更新会造成新老数据交替。
3. **更少的内部碎片**。每次紧缩会将垃圾完全挤出。但是原地更新就会在 page 中留下一些不可用空间。

当然，基于内存的哈希索引也有其局限：

1. **所有 Key 必须放内存**。一旦 Key 的数据量超过内存大小，这种方案便不再 work。当然你可以设计基于磁盘的哈希表，但那又会带来大量的随机写。
2. **不支持范围查询**。由于 key 是无序的，要进行范围查询必须全表扫描。

后面讲的 LSM-Tree 和 B+ 树，都能部分规避上述问题。

- 想想，会如何进行规避？

## SSTables 和 LSM-Trees

这一节层层递进，步步做引，从 SSTables 格式出发，牵出 LSM-Trees 全貌。

对于 KV 数据，前面的 BitCask 存储结构是：

1. 外存上日志片段
2. 内存中的哈希表

其中外存上的数据是简单追加写而形成的，并没有按照某个字段有序。

假设加一个限制，让这些文件按 key 有序。我们称这种格式为：SSTable（Sorted String Table）。

这种文件格式有什么优点呢？

**高效的数据文件合并**。即有序文件的归并外排，顺序读，顺序写。不同文件出现相同 Key 怎么办？

![ddia-3-4-merge-sst.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig04.png)

**不需要在内存中保存所有数据的索引**。仅需要记录下每个文件界限（以区间表示：[startKey, endKey]，当然实际会记录的更细）即可。查找某个 Key 时，去所有包含该 Key 的区间对应的文件二分查找即可。

![ddia-3-5-sst-index.png](https://s2.loli.net/2022/04/16/j8tM6IUk1QrJXuw.png)

**分块压缩，节省空间，减少 IO**。相邻 Key 共享前缀，既然每次都要批量取，那正好一组 key batch 到一块，称为 block，且只记录 block 的索引。

### 构建和维护 SSTables

SSTables 格式听起来很美好，但须知数据是乱序的来的，我们如何得到有序的数据文件呢？

这可以拆解为两个小问题：

1. 如何构建。
2. 如何维护。

**构建 SSTable 文件**。将乱序数据在外存（磁盘 or SSD）中上整理为有序文件，是比较难的。但是在内存就方便的多。于是一个大胆的想法就形成了：

1. 在内存中维护一个有序结构（称为 **MemTable**）。红黑树、AVL 树、跳表。
2. 到达一定阈值之后全量 dump 到外存。

**维护 SSTable 文件**。为什么需要维护呢？首先要问，对于上述复合结构，我们怎么进行查询：

1. 先去 MemTable 中查找，如果命中则返回。
2. 再去 SSTable 按时间顺序由新到旧逐一查找。

如果 SSTable 文件越来越多，则查找代价会越来越大。因此需要将多个 SSTable 文件合并，以减少文件数量，同时进行 GC，我们称之为**紧缩**（ Compaction）。

**该方案的问题**：如果出现宕机，内存中的数据结构将会消失。解决方法也很经典：WAL。

### 从 SSTables 到 LSM-Tree

将前面几节的一些碎片有机的组织起来，便是时下流行的存储引擎 LevelDB 和 RocksDB 后面的存储结构：LSM-Tree：

![ddia-3-leveldb-architecture.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig05.png)

这种数据结构是 Patrick O’Neil 等人，在 1996 年提出的：[The Log-Structured Merge-Tree](https://www.cs.umb.edu/~poneil/lsmtree.pdf 'The Log-Structured Merge-Tree')。

Elasticsearch 和 Solr 的索引引擎 Lucene，也使用类似 LSM-Tree 存储结构。但其数据模型不是 KV，但类似：word → document list。

### 性能优化

如果想让一个引擎工程上可用，还会做大量的性能优化。对于 LSM-Tree 来说，包括：

**优化 SSTable 的查找**。常用 [**Bloom Filter**](https://www.qtmuniao.com/2020/11/18/leveldb-data-structures-bloom-filter/)。该数据结构可以使用较少的内存为每个 SSTable 做一些指纹，起到一些初筛的作用。

**层级化组织 SSTable**。以控制 Compaction 的顺序和时间。常见的有 size-tiered 和 leveled compaction。LevelDB 便是支持后者而得名。前者比较简单粗暴，后者性能更好，也因此更为常见。

![ddia-sized-tierd-compact.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-sized-tiered.png)

对于 RocksDB 来说，工程上的优化和使用上的优化就更多了。在其 [Wiki](https://github.com/facebook/rocksdb/wiki 'rocksdb wiki') 上随便摘录几点：

1. Column Family
2. 前缀压缩和过滤
3. 键值分离，BlobDB

但无论有多少变种和优化，LSM-Tree 的核心思想——**保存一组合理组织、后台合并的 SSTables** ——简约而强大。可以方便的进行范围遍历，可以变大量随机为少量顺序。

## B 族树

虽然先讲的 LSM-Tree，但是它要比 B+ 树新的多。

B 树于 1970 年被 R. Bayer and E. McCreight [提出](https://dl.acm.org/doi/10.1145/1734663.1734671 'b tree paper')后，便迅速流行了起来。现在几乎所有的关系型数据中，它都是数据索引标准一般的实现。

与 LSM-Tree 一样，它也支持高效的**点查**和**范围查**。但却使用了完全不同的组织方式。

其特点有：

1. 以页（在磁盘上叫 page，在内存中叫 block，通常为 4k）为单位进行组织。
2. 页之间以页 ID 来进行逻辑引用，从而组织成一颗磁盘上的树。

![ddia-3-6-b-tree-lookup.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig06.png)

**查找**。从根节点出发，进行二分查找，然后加载新的页到内存中，继续二分，直到命中或者到叶子节点。查找复杂度，树的高度—— O(lgn)，影响树高度的因素：分支因子（分叉数，通常是几百个）。

![ddia-3-7-b-tree-grow-by-split.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig07.png)

**插入 or 更新**。和查找过程一样，定位到原 Key 所在页，插入或者更新后，将页完整写回。如果页剩余空间不够，则分裂后写入。

**分裂 or 合并**。级联分裂和合并。

- 一个记录大于一个 page 怎么办？
  树的节点是逻辑概念，page or block 是物理概念。一个逻辑节点可以对应多个物理 page。

### 让 B 树更可靠

B 树不像 LSM-Tree，会在原地修改数据文件。

在树结构调整时，可能会级联修改很多 Page。比如叶子节点分裂后，就需要写入两个新的叶子节点，和一个父节点（更新叶子指针）。

1. 增加预写日志（WAL），将所有修改操作记录下来，预防宕机时中断树结构调整而产生的混乱现场。
2. 使用 latch 对树结构进行并发控制。

### B 树的优化

B 树出来了这么久，因此有很多优化：

1. 不使用 WAL，而在写入时利用 Copy On Write 技术。同时，也方便了并发控制。如 LMDB、BoltDB。
2. 对中间节点的 Key 做压缩，保留足够的路由信息即可。以此，可以节省空间，增大分支因子。
3. 为了优化范围查询，有的 B 族树将叶子节点存储时物理连续。但当数据不断插入时，维护此有序性的代价非常大。
4. 为叶子节点增加兄弟指针，以避免顺序遍历时的回溯。即 B+ 树的做法，但远不局限于 B+ 树。
5. B 树的变种，分形树，从 LSM-tree 借鉴了一些思想以优化 seek。

## B-Trees 和 LSM-Trees 对比

| 存储引擎 | B-Tree                                                   | LSM-Tree                                                     | 备注                                                         |
| -------- | -------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 优势     | 读取更快                                                 | 写入更快                                                     |                                                              |
| 写放大   | 1. 数据和 WAL<br/>2. 更改数据时多次覆盖整个 Page         | 1. 数据和 WAL<br/>2. Compaction                              | SSD 不能过多擦除。因此 SSD 内部的固件中也多用日志结构来减少随机小写。 |
| 写吞吐   | 相对较低：<br/>1. 大量随机写。                           | 相对较高：<br/>1. 较低的写放大（取决于数据和配置）<br/>2. 顺序写入。<br/>3. 更为紧凑。 |                                                              |
| 压缩率   | 1. 存在较多内部碎片。                                    | 1. 更加紧凑，没有内部碎片。<br/>2. 压缩潜力更大（共享前缀）。 | 但紧缩不及时会造成 LSM-Tree 存在很多垃圾                     |
| 后台流量 | 1. 更稳定可预测，不会受后台 compaction 突发流量影响。    | 1. 写吞吐过高，compaction 跟不上，会进一步加重读放大。<br/>2. 由于外存总带宽有限，compaction 会影响读写吞吐。<br/>3. 随着数据越来越多，compaction 对正常写影响越来越大。 | RocksDB 写入太过快会引起 write stall，即限制写入，以期尽快 compaction 将数据下沉。 |
| 存储放大 | 1. 有些 Page 没有用满                                    | 1. 同一个 Key 存多遍                                         |                                                              |
| 并发控制 | 1. 同一个 Key 只存在一个地方<br/>2. 树结构容易加范围锁。 | 同一个 Key 会存多遍，一般使用 MVCC 进行控制。                |                                                              |

## 其他索引结构

**次级索引（secondary indexes）**。即，非主键的其他属性到该元素（SQL 中的行，MongoDB 中的文档和图数据库中的点和边）的映射。

### **聚集索引和非聚集索引（cluster indexes and non-cluster indexes）**

对于存储数据和组织索引，我们可以有多种选择：

1. 数据本身**无序**的存在文件中，称为 **堆文件（heap file）**，索引的值指向对应数据在 heap file 中的位置。这样可以避免多个索引时的数据拷贝。
2. 数据本身按某个字段有序存储，该字段通常是主键。则称基于此字段的索引为**聚集索引**（clustered index），从另外一个角度理解，即将索引和数据存在一块。则基于其他字段的索引为**非聚集索引**，在索引中仅存数据的引用。
3. 一部分列内嵌到索引中存储，一部分列数据额外存储。称为**覆盖索引（covering index）**
   或  **包含列的索引（index with included columns）**。

索引可以加快查询速度，但需要占用额外空间，并且牺牲了部分更新开销，且需要维持某种一致性。

### **多列索引**（**Multi-column indexes**）

现实生活中，多个字段联合查询更为常见。比如查询某个用户周边一定范围内的商户，需要经度和纬度二维查询。

```sql
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079
                            AND longitude > -0.1162 AND longitude < -0.1004;
```

可以：

1. 将二维编码为一维，然后按普通索引存储。
2. 使用特殊数据结构，如 R 树。

### **全文索引和模糊索引（Full-text search and fuzzy indexes）**

前述索引只提供全字段的精确匹配，而不提供类似搜索引擎的功能。比如，按字符串中包含的单词查询，针对笔误的单词查询。

在工程中常用 [Apace Lucene](https://lucene.apache.org/ 'Apace Lucene') 库，和其包装出来的服务：[Elasticsearch](https://www.elastic.co/cn/ 'Elasticsearch')。他也使用类似 LSM-tree 的日志存储结构，但使用其索引进行模糊匹配的过程，本质上是一个有限状态自动机，在行为上类似 Trie 树。

### 全内存数据结构

随着单位内存成本下降，甚至支持持久化（_non-volatile memory_，NVM，如 Intel 的 [傲腾](https://www.intel.cn/content/www/cn/zh/products/details/memory-storage/optane-dc-persistent-memory.html '傲腾')），全内存数据库也逐渐开始流行。

根据是否需要持久化，内存数据大概可以分为两类：

1. **不需要持久化**。如只用于缓存的 Memcached。
2. **需要持久化**。通过 WAL、定期 snapshot、远程备份等等来对数据进行持久化。但使用内存处理全部读写，因此仍是内存数据库。

> VoltDB, MemSQL, and Oracle TimesTen 是提供关系模型的内存数据库。RAMCloud 是提供持久化保证的 KV 数据库。Redis and Couchbase 仅提供弱持久化保证。

内存数据库存在优势的原因不仅在于不需要读取磁盘，而在更于不需要对数据结构进行**序列化、编码**后以适应磁盘所带来的**额外开销**。

当然，内存数据库还有以下优点：

1. **提供更丰富的数据抽象**。如 set 和 queue 这种只存在于内存中的数据抽象。
2. **实现相对简单**。因为所有数据都在内存中。

此外，内存数据库还可以通过类似操作系统 swap 的方式，提供比物理机内存更大的存储空间，但由于其有更多数据库相关信息，可以将换入换出的粒度做的更细、性能做的更好。

基于**非易失性存储器**（non-volatile memory，NVM）的存储引擎也是这些年研究的一个热点。

# 事务型还是分析型

术语 **OL**（Online）主要是指交互式的查询。

术语**事务**（transaction）由来有一些历史原因。早期的数据库使用方多为商业交易（commercial），比如买卖、发工资等等。但是随着数据库应用不断扩大，交易\事务作为名词保留了下来。

> 事务不一定具有 ACID 特性，事务型处理多是随机的以较低的延迟进行读写，与之相反，分析型处理多为定期的批处理，延迟较高。

下表是一个对比：

| 属性         | OLTP                            | OLAP                                   |
| ------------ | ------------------------------- | -------------------------------------- |
| 主要读取模式 | 小数据量的随机读，通过 key 查询 | 大数据量的聚合（max,min,sum, avg）查询 |
| 主要写入模式 | 随机访问，低延迟写入            | 批量导入（ETL）或者流式写入            |
| 主要应用场景 | 通过 web 方式使用的最终用户     | 互联网分析，为了辅助决策               |
| 如何看待数据 | 当前时间点的最新状态            | 随着时间推移的                         |
| 数据尺寸     | 通常 GB 到 TB                   | 通常 TB 到 PB                          |

一开始对于 AP 场景，仍然使用的传统数据库。在模型层面来说，SQL 足够灵活，能够基本满足 AP 查询需求。但在实现层面，传统数据库在 AP 负载中的表现（大数据量吞吐较低）不尽如人意，因此大家开始转向在专门设计的数据库中进行 AP 查询，我们称之为**数据仓库**（Data Warehouse）。

## 数据仓库

对于一个企业来说，一般都会有很多偏交易型的系统，如用户网站、收银系统、仓库管理、供应链管理、员工管理等等。通常要求**高可用**与**低延迟**，因此直接在原库进行业务分析，会极大影响正常负载。因此需要一种手段将数据从原库导入到专门的**数仓**。

我们称之为 **ETL：extract-transform-load**。

![ddia-3-8-etl.png](https://s2.loli.net/2022/04/16/HAq8ekcmz65xGlO.png)

一般企业的数据量达到一定的量级才会需要进行 AP 分析，毕竟在小数据量尺度下，用 Excel 进行聚合查询都够了。当然，现在一个趋势是，随着移动互联网、物联网的普及，接入终端的类型和数量越来越多，产生的数据增量也越来越大，哪怕初创不久的公司可能也会积存大量数据，进而也需要 AP 支持。

AP 场景下的**聚合查询**分析和传统 TP 型有所不同。因此，需要构建索引的方式也多有不同。

### 同样接口后的不同

TP 和 AP 都可以使用 SQL 模型进行查询分析。但是由于其负载类型完全不同，在查询引擎实现和存储格式优化时，做出的设计决策也就大相径庭。因此，在同一套 SQL 接口的表面下，两者对应的数据库实现结构差别很大。

虽然有的数据库系统号称两者都支持，比如之前的 Microsoft SQL Server 和 SAP HANA，但是也正日益发展成两种独立的查询引擎。近年来提的较多的 HTAP 系统也是类似，其为了 serve 不同类型负载底层其实有两套不同的存储，只不过系统内部会自动的做数据的冗余和重新组织，对用户透明。

## AP 建模：星状型和雪花型

AP 中的处理模型相对较少，比较常用的有**星状模型**，也称为**维度模型**。

![ddia3-9-star-schema.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig09.png)

如上图所示，星状模型通常包含一张**事件表（_fact table_）** 和多张**维度表（_dimension tables_）**。事件表以事件流的方式将数据组织起来，然后通过外键指向不同的维度。

星状模型的一个变种是雪花模型，可以类比雪花（❄️）图案，其特点是在维度表中会进一步进行二次细分，讲一个维度分解为几个子维度。比如品牌和产品类别可能有单独的表格。星状模型更简单，雪花模型更精细，具体应用中会做不同取舍。

在典型的数仓中，事件表可能会非常宽，即有很多的列：一百到数百列。

# 列存

前一小节提到的**分维度表**和**事实表**，对于后者来说，有可能达到数十亿行和数 PB 大。虽然事实表可能通常有几十上百列，但是单次查询通常只关注其中几个维度（列）。

如查询**人们是否更倾向于在一周的某一天购买新鲜水果或糖果**：

```sql
SELECT
  dim_date.weekday,
  dim_product.category,
  SUM(fact_sales.quantity) AS quantity_sold
FROM fact_sales
  JOIN dim_date ON fact_sales.date_key = dim_date.date_key
  JOIN dim_product ON fact_sales.product_sk = dim_product.product_sk
WHERE
  dim_date.year = 2013 AND
  dim_product.category IN ('Fresh fruit', 'Candy')
GROUP BY
  dim_date.weekday, dim_product.category;
```

由于传统数据库通常是按行存储的，这意味着对于属性（列）很多的表，哪怕只查询一个属性，也必须从磁盘上取出很多属性，无疑浪费了 IO 带宽、增大了读放大。

于是一个很自然的想法呼之欲出：每一个列分开存储好不好？

![ddia-3-10-store-by-column.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig10.png)

不同列之间同一个行的字段可以通过下标来对应。当然也可以内嵌主键来对应，但那样存储成本就太高了。

## 列压缩

将所有数据分列存储在一块，带来了一个意外的好处，由于同一属性的数据相似度高，因此更易压缩。

如果每一列中值阈相比行数要小的多，可以用**位图编码（_[bitmap encoding](https://en.wikipedia.org/wiki/Bitmap_index 'bitmap encoding')_）**。举个例子，零售商可能有数十亿的销售交易，但只有 100,000 个不同的产品。

![ddia-3-11-compress.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig11.png)

上图中，是一个列分片中的数据，可以看出只有 {29, 30, 31, 68, 69, 74} 六个离散值。针对每个值出现的位置，我们使用一个 bit array 来表示：

1. bit map 下标对应列的下标
2. 值为 0 则表示该下标没有出现该值
3. 值为 1 则表示该下标出现了该值

如果 bit array 是稀疏的，即大量的都是 0，只要少量的 1。其实还可以使用 **[游程编码](https://zh.wikipedia.org/zh/%E6%B8%B8%E7%A8%8B%E7%BC%96%E7%A0%81 '游程编码')（RLE，Run-length encoding）** 进一步压缩：

1. 将连续的 0 和 1，改写成 `数量+值`，比如 `product_sk = 29` 是 `9 个 0，1 个 1，8 个 0`。
2. 使用一个小技巧，将信息进一步压缩。比如将同值项合并后，肯定是 0 1 交错出现，固定第一个值为 0，则交错出现的 0 和 1 的值也不用写了。则 `product_sk = 29` 编码变成 `9，1，8`
3. 由于我们知道 bit array 长度，则最后一个数字也可以省掉，因为它可以通过 `array len - sum(other lens)` 得到，则 `product_sk = 29` 的编码最后变成：`9，1`

位图索引很适合应对查询中的逻辑运算条件，比如：

```sql
WHERE product_sk IN（30，68，69）
```

可以转换为 `product_sk = 30`、`product_sk = 68`和  `product_sk = 69`这三个 bit array 按位或（OR）。

```sql
WHERE product_sk = 31 AND store_sk = 3
```

可以转换为 `product_sk = 31`和  `store_sk = 3`的 bit array 的按位与，就可以得到所有需要的位置。

### 列族

书中特别提到**列族（column families）**。它是 Cassandra 和 HBase 中的的概念，他们都起源于自谷歌的 [BigTable](https://en.wikipedia.org/wiki/Bigtable 'BigTable') 。注意到他们和**列式（column-oriented）存储**有相似之处，但绝不完全相同：

1. 同一个列族中多个列是一块存储的，并且内嵌行键（row key）。
2. 并且列不压缩（存疑？）

因此 BigTable 在用的时候主要还是面向行的，可以理解为每一个列族都是一个子表。

### 内存带宽和向量化处理

数仓的超大规模数据量带来了以下瓶颈：

1. 内存处理带宽
2. CPU 分支预测错误和[流水线停顿](https://zh.wikipedia.org/wiki/%E6%B5%81%E6%B0%B4%E7%BA%BF%E5%81%9C%E9%A1%BF '流水线停顿')

关于内存的瓶颈可已通过前述的数据压缩来缓解。对于 CPU 的瓶颈可以使用：

1. 列式存储和压缩可以让数据尽可能多地缓存在 L1 中，结合位图存储进行快速处理。
2. 使用 SIMD 用更少的时钟周期处理更多的数据。

## 列式存储的排序

由于数仓查询多集中于聚合算子（比如 sum，avg，min，max），列式存储中的存储顺序相对不重要。但也免不了需要对某些列利用条件进行筛选，为此我们可以如 LSM-Tree 一样，对所有行按某一列进行排序后存储。

> 注意，不可能同时对多列进行排序。因为我们需要维护多列间的下标间的对应关系，才可能按行取数据。

同时，排序后的那一列，压缩效果会更好。

### 不同副本，不同排序

在分布式数据库（数仓这么大，通常是分布式的）中，同一份数据我们会存储多份。对于每一份数据，我们可以按不同列有序存储。这样，针对不同的查询需求，便可以路由到不同的副本上做处理。当然，这样也最多只能建立副本数（通常是 3 个左右）列索引。

这一想法由 C-Store 引入，并且为商业数据仓库 Vertica 采用。

## 列式存储的写入

上述针对数仓的优化（列式存储、数据压缩和按列排序）都是为了解决数仓中常见的读写负载，读多写少，且读取都是超大规模的数据。

> 我们针对读做了优化，就让写入变得相对困难。

比如 B 树的**原地更新流**是不太行的。举个例子，要在中间某行插入一个数据，**纵向**来说，会影响所有的列文件（如果不做 segment 的话）；为了保证多列间按下标对应，**横向**来说，又得更新该行不同列的所有列文件。

所幸我们有 LSM-Tree 的追加流。

1. 将新写入的数据在**内存**中 Batch 好，按行按列，选什么数据结构可以看需求。
2. 然后达到一定阈值后，批量刷到**外存**，并与老数据合并。

数仓 Vertica 就是这么做的。

## 聚合：数据立方和物化视图

不一定所有的数仓都是列式存储，但列式存储的种种好处让其变得流行了起来。

其中一个值得一提的是**物化聚合（materialized aggregates，或者物化汇总）**。

> 物化，可以简单理解为持久化。本质上是一种空间换时间的 tradeoff。

数据仓库查询通常涉及聚合函数，如 SQL 中的 COUNT、SUM、AVG、MIN 或 MAX。如果这些函数被多次用到，每次都即时计算显然存在巨大浪费。因此一个想法就是，能不能将其缓存起来。

其与关系数据库中的**视图**（View）区别在于，视图是**虚拟的、逻辑存在**的，只是对用户提供的一种抽象，是一个查询的中间结果，并没有进行持久化（有没有缓存就不知道了）。

物化视图本质上是对数据的一个摘要存储，如果原数据发生了变动，该视图要被重新生成。因此，如果**写多读少**，则维持物化视图的代价很大。但在数仓中往往反过来，因此物化视图才能较好的起作用。

物化视图一个特化的例子，是**数据立方**（data cube，或者 OLAP cube）：按不同维度对量化数据进行聚合。

![ddia-3-12-data-cube.png](/Users/zhuoshuyang/Desktop/ddia/img/ch03-fig12.png)

上图是一个按**日期和产品分类**两个维度进行加和的数据立方，当针对日期和产品进行汇总查询时，由于该表的存在，就会变得非常快。

当然，现实中，一个表中常常有多个维度，比如 3-9 中有日期、产品、商店、促销和客户五个维度。但构建数据立方的意义和方法都是相似的。

但这种构建出来的视图只能针对固定的查询进行优化，如果有的查询不在此列，则这些优化就不再起作用。

> 在实际中，需要针对性的识别（或者预估）每个场景查询分布，针对性的构建物化视图。
