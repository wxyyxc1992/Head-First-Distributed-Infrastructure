# 分布式事务

在[《数据库导论](https://github.com/wx-chevalier/Database-Series)》中我们讨论了数据库中著名的 ACID 特性，以及事务与并发控制的相关内容。随着互联网快速发展，微服务，SOA 等服务架构模式正在被大规模的使用，现在分布式系统一般由多个独立的子系统组成，多个子系统通过网络通信互相协作配合完成各个功能。而且这个过程中会涉及到事务的概念，即保证交易系统和支付系统的数据一致性，此处我们称这种跨系统的事务为分布式事务。具体一点而言，分布式事务是指事务的参与者、支持事务的服务器、资源服务器以及事务管理器分别位于不同的分布式系统的不同节点之上。分布式事务处理的关键是必须有一种方法可以知道事务在任何地方所做的所有动作，提交或回滚事务的决定必须产生统一的结果(全部提交或全部回滚)。

有很多用例会跨多个子系统才能完成，比较典型的是电子商务网站的下单支付流程，至少会涉及交易系统和支付系统；在用户下单之后，需要在库存子系统减少库存，然后在订单子系统添加订单。如果多个数据库之间的数据更新没有保证事务，将会导致出现子系统数据不一致，业务出现问题。

在单一数据节点中，事务仅限于对单一数据库资源的访问控制，称之为本地事务。几乎所有的成熟的关系型数据库都提供了对本地事务的原生支持。本地事务即在不开启任何分布式事务管理器的前提下，让每个数据节点各自管理自己的事务。它们之间没有协调以及通信的能力，也并不互相知晓其他数据节点事务的成功与否。本地事务在性能方面无任何损耗，但在强一致性以及最终一致性方面则力不从心。

但是在基于微服务的分布式应用环境下，越来越多的应用场景要求对多个服务的访问及其相对应的多个数据库资源能纳入到同一个事务当中，分布式事务应运而生。关系型数据库虽然对本地事务提供了完美的 ACID 原生支持。但在分布式的场景下，它却成为系统性能的桎梏。如何让数据库在分布式场景下满足 ACID 的特性或找寻相应的替代方案，是分布式事务的重点工作。

分布式事务同样需要满足原子性，一致性，隔离性等特征：

- 事务的原子性：事务操作跨不同节点，当多个节点某一节点操作失败时，需要保证多节点操作的要么不做，要么全做（All or Nothing）的原子性。在《一致性与共识》章节中我们也讨论过，共识算法的典型应用场景之一就是原子提交（atomic commit）。

- 事务的一致性：当发生网络传输故障或者节点故障，节点间数据复制通道中断，在进行事务操作时需要保证数据一致性，保证事务的任何操作都不会使得数据违反数据库定义的约束、触发器等规则。

- 事务的隔离性：事务隔离性的本质就是如何正确处理多个并发事务的读写冲突和写写冲突，因为在分布式事务控制中，可能会出现提交不同步的现象，这个时候就有可能出现“部分已经提交”的事务。此时并发应用访问数据如果没有加以控制，有可能出现“脏读”问题。

事务是为了保证数据的一致性，在[分布式系统理论]()中我们也讨论了 CAP 理论，对于分布式的应用而言，不可能同时满足 C（一致性），A（可用性），P（分区容错性），由于网络分区是分布式应用的基本要素，因此开发者需要在 C 和 A 上做出平衡。由于 C 和 A 互斥性，其权衡的结果就是 BASE 理论。对于大部分的分布式应用而言，只要数据在规定的时间内达到最终一致性即可。我们可以把符合传统的 ACID 叫做刚性事务，把满足 BASE 理论的最终一致性事务叫做柔性事务。

# 事务机制

# 多阶段提交

XA 协议最早的分布式事务模型是由 X/Open 国际联盟提出的 X/Open Distributed Transaction Processing（DTP）模型，简称 XA 协议。

基于 XA 协议实现的分布式事务对业务侵入很小。它最大的优势就是对使用方透明，用户可以像使用本地事务一样使用基于 XA 协议的分布式事务。XA 协议能够严格保障事务 ACID 特性。

严格保障事务 ACID 特性是一把双刃剑。事务执行在过程中需要将所需资源全部锁定，它更加适用于执行时间确定的短事务。对于长事务来说，整个事务进行期间对数据的独占，将导致对热点数据依赖的业务系统并发性能衰退明显。因此，在高并发的性能至上场景中，基于 XA 协议两阶段提交类型的分布式事务并不是最佳选择。

# 实践中的分布式事务

分布式事务的名声毁誉参半，尤其是那些通过两阶段提交实现的。一方面，它被视作提供了一个难以实现的重要的安全性保证；另一方面，它们因为导致运维问题，造成性能下降，做出超过能力范围的承诺而饱受批评。许多云服务由于其导致的运维问题，而选择不实现分布式事务。分布式事务的某些实现会带来严重的性能损失，例如据报告称，MySQL 中的分布式事务比单节点事务慢 10 倍以上，所以当人们建议不要使用它们时就不足为奇了。两阶段提交所固有的性能成本，大部分是由于崩溃恢复所需的额外强制刷盘（fsync）以及额外的网络往返。

但我们不应该直接忽视分布式事务，而应当更加仔细地审视这些事务，因为从中可以汲取重要的经验教训。首先，我们应该精确地说明“分布式事务”的含义。两种截然不同的分布式事务类型经常被混淆：

- 数据库内部的分布式事务：一些分布式数据库（即在其标准配置中使用复制和分区的数据库）支持数据库节点之间的内部事务。例如，VoltDB 和 MySQL Cluster 的 NDB 存储引擎就有这样的内部事务支持。在这种情况下，所有参与事务的节点都运行相同的数据库软件。

- 异构分布式事务：在异构（Heterogeneous）事务中，参与者是两种或以上不同技术：例如来自不同供应商的两个数据库，甚至是非数据库系统（如消息代理）。跨系统的分布式事务必须确保原子提交，尽管系统可能完全不同。

数据库内部事务不必与任何其他系统兼容，因此它们可以使用任何协议，并能针对特定技术进行特定的优化。因此数据库内部的分布式事务通常工作地很好。另一方面，跨异构技术的事务则更有挑战性。

## 恰好一次的消息处理

异构的分布式事务处理能够以强大的方式集成不同的系统。例如：消息队列中的一条消息可以被确认为已处理，当且仅当用于处理消息的数据库事务成功提交。这是通过在同一个事务中原子提交消息确认和数据库写入两个操作来实现的。藉由分布式事务的支持，即使消息代理和数据库是在不同机器上运行的两种不相关的技术，这种操作也是可能的。

如果消息传递或数据库事务任意一者失败，两者都会中止，因此消息代理可能会在稍后安全地重传消息。因此，通过原子提交消息处理及其副作用，即使在成功之前需要几次重试，也可以确保消息被有效地（effectively）恰好处理一次。中止会抛弃部分完成事务所导致的任何副作用。

然而，只有当所有受事务影响的系统都使用同样的原子提交协议（atomic commit protocl）时，这样的分布式事务才是可能的。例如，假设处理消息的副作用是发送一封邮件，而邮件服务器并不支持两阶段提交：如果消息处理失败并重试，则可能会发送两次或更多次的邮件。但如果处理消息的所有副作用都可以在事务中止时回滚，那么这样的处理流程就可以安全地重试，就好像什么都没有发生过一样。

## XA 事务

X/Open XA（扩展架构（eXtended Architecture）的缩写）是跨异构技术实现两阶段提交的标准。它于 1991 年推出并得到了广泛的实现：许多传统关系数据库（包括 PostgreSQL，MySQL，DB2，SQL Server 和 Oracle）和消息代理（包括 ActiveMQ，HornetQ，MSMQ 和 IBM MQ）都支持 XA。XA 不是一个网络协议——它只是一个用来与事务协调者连接的 C API。其他语言也有这种 API 的绑定；例如在 Java EE 应用的世界中，XA 事务是使用 Java 事务 API（JTA, Java Transaction API）实现的，而许多使用 Java 数据库连接（JDBC, Java Database Connectivity）的数据库驱动，以及许多使用 Java 消息服务（JMS）API 的消息代理都支持 Java 事务 API（JTA）。

XA 假定你的应用使用网络驱动或客户端库来与参与者进行通信（数据库或消息服务）。如果驱动支持 XA，则意味着它会调用 XA API 以查明操作是否为分布式事务的一部分；如果是，则将必要的信息发往数据库服务器。驱动还会向协调者暴露回调接口，协调者可以通过回调来要求参与者准备，提交或中止。事务协调者需要实现 XA API。标准没有指明应该如何实现，但实际上协调者通常只是一个库，被加载到发起事务的应用的同一个进程中（而不是单独的服务）。它在事务中个跟踪所有的参与者，并在要求它们准备之后收集参与者的响应（通过驱动回调），并使用本地磁盘上的日志记录每次事务的决定（提交/中止）。

如果应用进程崩溃，或者运行应用的机器报销了，协调者也随之往生极乐。然后任何带有准备了但未提交事务的参与者都会在疑虑中卡死。由于协调程序的日志位于应用服务器的本地磁盘上，因此必须重启该服务器，且协调程序库必须读取日志以恢复每个事务的提交/中止结果。只有这样，协调者才能使用数据库驱动的 XA 回调来要求参与者提交或中止。数据库服务器不能直接联系协调者，因为所有通信都必须通过客户端库。

X/Open 组织(即现在的 Open Group）定义了分布式事务处理模型。X/Open DTP 模型（1994）包括应用程序（AP）、事务管理器（TM）、资源管理器（RM）、通信资源管理器（CRM）四部分。一般，常见的事务管理器（TM）是交易中间 件，常见的资源管理器（RM）是数据库，常见的通信资源管理器（CRM）是消息中间件。通常把一个数据库内部的事务处理，如对多个表的操作，作为本地事务看待。数据库的事务处理对象是本地事务，而分布式事务处理的对象是全局事务。所谓全局事务，是指分布式事务处理环境中，多个数据库可能需要共同完成一个工作，这个工作即是一个全局事务，例如，一个事务中可能更新几个不同的数据库。对数据库的操作发生在系统的各处但必须全部被提交或回滚。此时一个数据库对自己内部所做操作的提交不仅依赖本身操作是否成功，还要依赖与全局事务相关的其 它数据库的操作是否成功，如果任一数据库的任一操作失败，则参与此事务的所有数据库所做的所有操作都必须回滚。一般情况下，某一数据库无法知道其它数据库在做什么，因此，在一个 DTP 环境中，交易中间件是必需的，由它通知和协调相关数据库的提交或回滚。而 一个数据库只将其自己所做的操作(可恢复)影射到全局事务中。

XA 就是 X/Open DTP 定义的交易中间件与数据库之间的接口规范(即接口函数)，交易中间件用它来通知数据库事务的开始、结束以及提交、回滚等。XA 接口函数由数据库厂商提供。

二阶提交协议和三阶提交协议就是根据这一思想衍生出来的。可以说二阶段提交其实就是实现 XA 分布式事务的关键(确切地说：两阶段提交主要保证了分布式事务的原子性：即所有结点要么全做要么全不做)。

## 柔性事务与补偿

在电商等互联网场景下，传统的事务在数据库性能和处理能力上都暴露出了瓶颈。在分布式领域基于 CAP 理论以及 BASE 理论，有人就提出了柔性事务的概念。基于 BASE 理论的设计思想，柔性事务下，在不影响系统整体可用性的情况下(Basically Available 基本可用)，允许系统存在数据不一致的中间状态(Soft State 软状态)，在经过数据同步的延时之后，最终数据能够达到一致。并不是完全放弃了 ACID，而是通过放宽一致性要求，借助本地事务来实现最终分布式事务一致性的同时也保证系统的吞吐。

柔性事务往往需要具备以下特性：

- 可见性(对外可查询)：在分布式事务执行过程中，如果某一个步骤执行出错，就需要明确的知道其他几个操作的处理情况，这就需要其他的服务都能够提供查询接口，保证可以通过查询来判断操作的处理情况。为了保证操作的可查询，需要对于每一个服务的每一次调用都有一个全局唯一的标识，可以是业务单据号（如订单号）、也可以是系统分配的操作流水号（如支付记录流水号）。除此之外，操作的时间信息也要有完整的记录。

- 操作幂等性：幂等性，其实是一个数学概念。幂等函数，或幂等方法，是指可以使用相同参数重复执行，并能获得相同结果的函数。幂等操作的特点是其任意多次执行所产生的影响均与一次执行的影响相同。也就是说，同一个方法，使用同样的参数，调用多次产生的业务结果与调用一次产生的业务结果相同。

之所以需要操作幂等性，是因为为了保证数据的最终一致性，很多事务协议都会有很多重试的操作，如果一个方法不保证幂等，那么将无法被重试。幂等操作的实现方式有多种，如在系统中缓存所有的请求与处理结果、检测到重复操作后，直接返回上一次的处理结果等。

事务补偿符合我们最终一致性的理念。补偿事务不一定会将系统中的数据返回到原始操作开始时其所处的状态。相反，它补偿操作失败前由已成功完成的步骤所执行的工作。补偿事务中步骤的顺序不一定与原始操作中步骤的顺序完全相反。例如，一个数据存储可能比另一个数据存储对不一致性更加敏感，因而补偿事务中撤销对此存储的更改的步骤应该会首先发生。对完成操作所需的每个资源采用短期的基于超时的锁并预先获取这些资源，这样有助于增加总体活动成功的可能性。仅在获取所有资源后才应执行工作。锁过期之前必须完成所有操作。

如果将实现了 ACID 的事务要素的事务称为刚性事务的话，那么基于 BASE 事务要素的事务则称为柔性事务。BASE 是基本可用、柔性状态和最终一致性这 3 个要素的缩写。

在 ACID 事务中对一致性和隔离性的要求很高，在事务执行过程中，必须将所有的资源占用。柔性事务的理念则是通过业务逻辑将互斥锁操作从资源层面上移至业务层面。通过放宽对强一致性和隔离性的要求，只要求当整个事务最终结束的时候，数据是一致的。而在事务执行期间，任何读取操作得到的数据都有可能被改变。这种弱一致性的设计可以用来换取系统吞吐量的提升。Saga 和 TCC 都是典型的柔性事务实现方案。

## 分布式事务的限制

XA 事务解决了保持多个参与者（数据系统）相互一致的现实的重要问题，但正如我们所看到的那样，它也引入了严重的运维问题。特别来讲，这里的核心认识是：事务协调者本身就是一种数据库（存储了事务的结果），因此需要像其他重要数据库一样小心地打交道：

- 如果协调者没有复制，而是只在单台机器上运行，那么它是整个系统的失效单点（因为它的失效会导致其他应用服务器阻塞在存疑事务持有的锁上）。令人惊讶的是，许多协调者实现默认情况下并不是高可用的，或者只有基本的复制支持。

- 许多服务器端应用都是使用无状态模式开发的（受 HTTP 的青睐），所有持久状态都存储在数据库中，因此具有应用服务器可随意按需添加删除的优点。但是，当协调者成为应用服务器的一部分时，它会改变部署的性质。突然间，协调者的日志成为持久系统状态的关键部分，与数据库本身一样重要，因为协调者日志是为了在崩溃后恢复存疑事务所必需的。这样的应用服务器不再是无状态的了。

- 由于 XA 需要兼容各种数据系统，因此它必须是所有系统的最小公分母。例如，它不能检测不同系统间的死锁（因为这将需要一个标准协议来让系统交换每个事务正在等待的锁的信息），而且它无法与 SSI 协同工作，因为这需要一个跨系统定位冲突的协议。

- 对于数据库内部的分布式事务（不是 XA），限制没有这么大，例如，分布式版本的 SSI 是可能的。然而仍然存在问题：2PC 成功提交一个事务需要所有参与者的响应。因此，如果系统的任何部分损坏，事务也会失败。因此，分布式事务又有**扩大失效（amplifying failures）**的趋势，这又与我们构建容错系统的目标背道而驰。

# 分布式事务方案

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g29eo85f8sj30u00ek75x.jpg)

![](https://ww1.sinaimg.cn/large/007rAy9hgy1g29eo837q8j30pk0b9wez.jpg)

![](http://dbaplus.cn/uploadfile/2018/0801/20180801105319120.jpg)

![](http://www.primeton.com/uploads/image/20160907/20160907103229_58941.png)

- 2PC/3PC：依赖于数据库，能够很好的提供强一致性和强事务性，但相对来说延迟比较高，比较适合传统的单体应用，在同一个方法中存在跨库操作的情况，不适合高并发和高性能要求的场景。

- TCC：适用于执行时间确定且较短，实时性要求高，对数据一致性要求高，比如互联网金融企业最核心的三个服务：交易、支付、账务。

- 消息驱动：都适用于事务中参与方支持操作幂等，对一致性要求不高，业务上能容忍数据不一致到一个人工检查周期，事务涉及的参与方、参与环节较少，业务上有对账/校验系统兜底。

- Saga 事务：由于 Saga 事务不能保证隔离性，需要在业务层控制并发，适合于业务场景事务并发操作同一资源较少的情况。Saga 相比缺少预提交动作，导致补偿动作的实现比较麻烦，例如业务是发送短信，补偿动作则得再发送一次短信说明撤销，用户体验比较差。Saga 事务较适用于补偿动作容易处理的场景。

# TBD

- https://mp.weixin.qq.com/s/vgohXl1LxYk3CyDI8WHxwA
- https://dbaplus.cn/news-159-2152-1.html?utm_source=oschina-app
- [分布式事务笔记](http://www.yangguo.info/2016/05/23/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%E7%AC%94%E8%AE%B0/)
