# 第九章：一致性与共识

## 可线性化

可线性化 [6]（也称为原子一致性 [7]，强一致性等 [8]）：让一个系统看起来好像只有一个数据副本， 且所有的操作都是原子的。

在一个可线性化的系统中， 一旦某个客户端成功提交写请求，所有客户端的读请求一定都能看到刚刚写入的值。这种看似单一副本的假象意味着它可以保证读取最近最新值，而不是过期的缓存。

![非线性化例子](../../.go_study/assets/ddia/9-1.png)

上图 Bob 读到过期的结果，因此就不是线性化。

可线性化一个重要的约束：当一个客户端读取到最新值，则之后其他客户端读取到的都必须是新值（不一定是针对数据库，也可以是文件系统，kv存储等），如下图所示；

![image-20220604114452416](../../.go_study/assets/ddia/9-2.png)

图 9-3 中的箭头表示时序依赖关系。客户端 A 首先读到新值 1，在 A 读取返回之后， B 开始读取，由于 B 的读取严格在 A 的读取之后发生，因此即使 C 的写入仍在进行之中，也必须返回 1。

另外，可线性化要求，如果连接这些标记的坚线，它们必须总是按时间箭头（从左到右）向前移动，而不能向后移动。这个要求确保了之前所讨论的就近性保证： 一旦新值被写入或读取，所有后续的读都看到的是最新的值，直到被再次覆盖。

![image-20220604120751743](../../.go_study/assets/ddia/9-3.png)

上图中的 cas 是 compare and swap，即原子比较和设置。上图中客户端 B 的最后读取不满足可线性化，因为在 B 读取之前 A 已经读出 x = 4，所以 B 读取应该也是 4。

总结，可线性化：基本想法是让一个系统看起来只有一个数据副本，且当一个客户端读取到最新值，则之后其他客户端读取到的都必须是新值（不一定是针对数据库，也可以是文件系统，kv存储等），存在一条无分叉的时间线。

### 线性化的使用场景

- 加锁与主节点选举：比如 Zookeeper 和 etcd 都使用了支持容错的共识算法确保可线性化。来实现分布式锁和主节点选举。
- 约束与唯一性保证：硬性的唯一性约束， 常见如关系型数据库中主键的约束，需要线性化保证。
- 跨通道的时间依赖：如下图所示，web 服务器和调整模块之间存在两个不同的通信通道：文件存储器和消息队列。如果没有线性化的就近性保证，这两个通道之间存在竞争条件。消息队列可能比存储保存图片更快，这样图片调整模块可能读取的是旧版本或者读不到数据。

![image-20220604122829978](../../.go_study/assets/ddia/9-4.png)

### 实现线性化系统

- 主从复制：部分支持可线性化。如果从主节点或者同步更新的从节点上读取，则可以满足线性化。但有些数据库采用了快照隔离的设计，或者实现时存在井发方面的bug [10]，就不满足可线性化。
- 共识算法：可线性化。
- 多主复制：不可线性化。
- 无主复制：可能不可线性化。

### 可线性化的代价

#### CAP

正式定义的 CAP 定理 [30] 范围很窄，它只考虑了一种一致性模型（即线性化）和一种故障（网络分区，节点仍处于活动状态但相互断开），而没有考虑网络延迟、节点失败或其他需要折中的情况。因此，尽管 CAP 在历史上具有重大的影响力，但对于一个具体的系统设计来说，它可能没有太大的实际价值 [9,40]。

分布式系统中还有很多有趣的研究结果 [41]，目前 CAP 已被更精确的研究成果所取代 [2,42]，所以它现在更多的是代表历史上曾经的一个关注热点而已。

> CAP 更准确的称呼应该是“网络分区情况下，选择一致还是可用 ”。



Attiya 和 Welch [47] 证明如果想要满足线性化， 那么读、写请求的响应时间至少要与网络中延迟成正比。 考虑到多数计算机网络高度不确定的网络延迟（参阅第7章 “超时与无限延迟”），线性化读写的性能势必非常差。 虽然没有足够快的线性化算法，但弱一致性模型的性能则快得多， 这种取舍对于延迟敏感的系统非常重要。



## 顺序保证

### 顺序与因果关系

因果关系对所发生的事件施加了某种排序：发送消息先于收到消息：问题出现在答案之前等。但因果顺序并非全序，而是偏序。

全序和偏序的差异也会体现在不同的数据库一致性模型中：

- 可线性化


在一个可线性化的系统中，存在全序操作关系。系统的行为就好像只有一个数据副本，且每个操作都是原子的，这意味着对于任何两个操作，我们总是可以指出哪个操作在先。

- 因果关系


如果两个操作都没有发生在对方之前，那么这两个操作是井发关系。换言之，如果两个事件是因果关系（ 一个发生在另一个之前），那么这两个事件可以被排序；而井发的事件则无法排序比较。这表明因果关系至少可以定义为偏序，而非全序。

在可线性化数据存储中不存在并发操作， 一定有一个时间线将所有操作都全序执行。单个时间轴，单个数据副本，没有并发。

#### 可线性化强于因果一致性

任何可线性化的系统都将正确地保证因果关系 [7]。 特别是如果系统存在多个通信通道（见图 9-5 中的消息队列和文件存储服务），可线性化确保了因果关系会自动全部保留，而不需要额外的工作（比如在不同组件之间的传递时间戳）。

因果一致性可以认为是，不会由于网络延迟而显著影响性能，又能对网络故障提供容错的最强的一致性模型 [2,42]。

### 序列号排序

可以使用序列号或时间戳来排序事件，时间戳可以只是一个逻辑时钟， 例如采用算法来产生一个数字序列用以识别操作，通常是递增的计数器。这样保证了全序关系。

#### Lamport时间戳

Lamport时间戳是一个值对（计数器，节点 ID）。计数器是已处理的请求数。

![image-20220604144521113](../../.go_study/assets/ddia/9-5.png)

可以保证全序 ： 给定两个 Lamport 时间戳，计数器较大那个时间戳大；如计数器值正好相同，则节点 ID 越 大，时间戳越大。

每个节点以及每个客户端都跟踪迄今为止所见到的最大计数器值，井在每个请求中附带该最大计数器值。当节点收到某个请求（或者回复）时，如果发现请求内嵌的最大计数器值大于节点自身的计数器值，则它立即把自己的计数器修改为该最大值。

 Lamport 时间戳与版本向量的目的不同：版本向量用以区分两个操作是并发还是因果依赖，而Lamport时间戳则主要用于确保全序关系。即使 Lamport 时间戳与因果序一致，但根据其全序关系却无法区分两个操作属于并发关系，还是因果依赖关系 。 Lamport时间戳优于版本向量之处在于它更加紧凑和高效。

#### 时间戳排序依然不够

Lamport 时间戳问题的关键是，只有在收集了所有的请求信息之后，才能清楚这些请求之间的全序关系。 如果另一个节点执行了某些操作，但你无法知道那是什么，就无法构造出最终的请求序列。

要想知道什么时候全序关系已经确定就需要之后的“全序关系广播”。

### 全序关系广播

如前所述，主从复制首先确定某一个节点作为主节点，然后在主节点上顺序执行操作。接下来的主要挑战在于，如何扩展系统的吞吐量使之突破单一主节点的限制，以及如何处理主节点失效时的故障切换（参阅第5章“处理节点失效”）。在分布式系统研究文献中，这些问题被称为全序关系广播或者原子广播 [25,57,58]。

全序关系广播需要两个基本安全属性：

- 可靠发送 

  没有消息丢失，如果消息发送到了某一个节点，则它一定要发送到所有节点。

- 严格有序

  消息总是以相同的顺序发送给每个节点。

即使节点或网络出现了故障，全序关系广播算法的正确实现也必须保证上述两条。当然，网络中断时是不可能发送成功的，但算法要继续重试，直到最终网络修复，消息发送成功（且必须以正确的顺序发送）。

#### 使用全序关系广播

像 Zookeeper 和 etcd 这样的共识服务实际上就实现了序全关系广播。

用处：数据库复制（状态机复制），实现可串行化事务等，对于提供 fencing 令牌的锁服务也很有用。

#### 全序关系广播与线性化存储

全序关系广播是基于异步模型：保证消息以固定的顺序可靠地发送，但是不保证消息何时发送成功（因此某个接收者可能明显落后于其他接收者 ）。

而可线性化则强调就近性：读取时保证能够看到最新的写入值。

线性化的原子比较－设置（或自增 ）寄存器与全序关系广播二者都等价于共识问题［28,67］。



## 分布式事务与共识

有很多重要的场景都需要集群节点达成某种一致，比如：

- 主节点选举：对于主从复制的数据库，所有节点需要就谁来充当主节点达成一致。
- 原子事务提交：所有节点对事务的结果达成一致，要么全部成功提交，要么终止/回滚。

### 原子提交与两阶段提交

如果一部分节点提交了事务，而其他节点却放弃了事务，节点之间就会变得不一致。而且某个节点一旦提交了事务，即使事后发现其他节点发生中止，它也无法再撤销已提交的事务。

#### 两阶段提交

两阶段提交（ two-phase commit, 2PC ）是一种在多节点之间实现事务原子提交的算法，用来确保所有节点要么全部提交要么全部中止。它是分布式数据库中的经典算法之一 [13,35,75]。

![2PC](../../.go_study/assets/ddia/9-6.png)

2PC 引入了单节点事务所没有的一个新组件 ：协调者（也称为事务管理器）。

2PC 保证分布式原子性，但因为存在协调者，当协调者崩溃，各参与节点处于不确定状态，是否提交或回滚，所以不支持可终止性，不具备容错性。

![image-20220604154826082](../../.go_study/assets/ddia/9-7.png)

### 实践中的分布式事务

#### XA

XA (eXtended Architecture, XA）是异构环境下实施两阶段提交的一个工业标准 [76,77]。XA并不是一个网络协议，而是一个与事务协调者进行通信的C API 。

#### 停顿时仍持有锁

数据库事务通常持有待修改行的行级独占锁，用以防止脏写。此外，如果要使用可串行化的隔离，则两阶段锁的数据库还会对事务曾经读取的行持有读－共享锁。因此如果协调者崩溃井且需要20分钟才能重启恢复，那么这些对象将被锁定20分钟；如果协调者的日志由于某种原因而彻底丢失，这些数据对象将永久处于加锁状态，至少管理员采用手动方式解决之前只能如此。

#### 从协调者故障中恢复

即使重启那些处于停顿状态的数据库节点也无法解决这个问题，这是由于 2PC 的正确实现要求即使发生了重启，也要继续保持重启之前事务的加锁（否则就会违背原子性保证）。唯一的出路只能是让管理员手动决定究竟是执行提交还是回滚。

许多 XA 的实现都支持某种紧急避险措施称之为启发式决策：这样参与者节点可以在紧急情况下单方面做出决定，放弃或者继续那些停顿的事务，而不需要等到协调者发出指令 [76,77,91]。

### 支持容错的共识

共识算法的性质：

- 协商一致性：所有节点都接受相同的决议。
- 诚实性：所有节点不能反悔，即对一项提议不能有两次决定。
- 合法性：如果决定了值v，则v一定是某个节点提出的。
- 可终止性：节点如果不崩溃则最终一定能达成决议；

可终止性引入了容错的思想。它重点强调一个共识算法不能原地空转，永远不做事情，换句话说，它必须取得实质性进展。即使某些节点出现了故障，其他节点也必须最终做出决定。可终止性属于一种活性，而另外三种则 属于安全性方面的属性。

而之前的两阶段提交就不支持可终止性和容错。

任何共识算法都需要至少大部分节点正确运行才能确保终止性 [67]。 而这个多数就可以安全地构成 quorum 。

#### 共识算法与全序广播

最著名的共识算法包括 VSR [94,95], Paxos [96-99], Raft [22, 100, 101], Zab [15, 21, 102]。

全序关系广播的要点是，消息按照相同的顺序发送到所有节点，有且只有一次 。

全序关系广播相当于持续的多轮共识（每一轮共识的决定对应于一条消息）：

- 由于协商一致性，所有节点决定以相同的顺序发送相同的消息。
- 由于诚实性，消息不能重复。
- 由于合法性，消息不会被破坏，也不是凭空捏造的。
- 由于可终止性，消息不会丢失。

#### 共识的局限性

主要包括：

- 节点投票的过程是一个同步复制过程。
- 共识体系需要严格的多数节点才能运行。如果由于网络故障切断了节点之间的连接，则只有多数节点所在的分区可以继续工作 ，剩下的少数节点分区则处于事实上的停顿状态。
- 多数共识算在是假定一组固定参与投票的节点集，即不能动态添加或删除节点。
- 共识系统通常依靠超时机制来检测节点失效。如果网络延迟比较大，那么可能会发生频繁的主节点选举，系统最终会花费更多的时间和资源在选举主节点上而不是原本的服务任务。

### 成员与协调服务

ZooKeeper 的实现其实模仿了 Google 的 Chubby 分布式锁服务 [14,98]，但它不仅实现了全序广播（因此实现了共识），还提供了以下特性：

- 线性化的原子操作。使用原子比较－设置操作，可以实现加锁服务。
- 操作全序。主要由全局递增的事务 ID zxid 和版本号 cversion 实现。
- 故障检测。通过交换心跳实现。
- 更改通知。订阅通知机制。



## 小结

线性化（一种流行的一 致性模型）：其目标是使多副本对外看起来好像是单一副本，然后所有操作以原子方式运行，就像一个单线程程序操作变量一样。主要问题在于性能，特别是在网络延迟较大的环境中。

因果关系对事件进行了某种排序（根据事件发生的原因结果依赖关系）。线性化是将所有操作都放在唯一的、全局有序时间线上，而因果性则不同，它为我们提供了一个弱一致性模型 ：允许存在某些井发事件，所以版本历史是一个包含多个分支与合井的时间线。因果一致性避免了线性化昂贵的协调开销，且对网络延迟的敏感性要低很多。

然而即使有了因果关系，还需要通过共识来查询其他节点是否存在竞争请求。

共识意味着就某一项提议，所有节点做出一致的决定，而且决定不可撤销。



## 参考文献

1. Peter Bailis and Ali Ghodsi: “[Eventual Consistency Today: Limitations, Extensions, and Beyond](http://queue.acm.org/detail.cfm?id=2462076),” *ACM Queue*, volume 11, number 3, pages 55-63, March 2013. [doi:10.1145/2460276.2462076](http://dx.doi.org/10.1145/2460276.2462076)
2. Prince Mahajan, Lorenzo Alvisi, and Mike Dahlin: “[Consistency, Availability, and Convergence](http://apps.cs.utexas.edu/tech_reports/reports/tr/TR-2036.pdf),” University of Texas at Austin, Department of Computer Science, Tech Report UTCS TR-11-22, May 2011.
3. Alex Scotti: “[Adventures in Building Your Own Database](http://www.slideshare.net/AlexScotti1/allyourbase-55212398),” at *All Your Base*, November 2015.
4. Peter Bailis, Aaron Davidson, Alan Fekete, et al.: “[Highly Available Transactions: Virtues and Limitations](http://arxiv.org/pdf/1302.0309.pdf),” at *40th International Conference on Very Large Data Bases* (VLDB), September 2014. Extended version published as pre-print arXiv:1302.0309 [cs.DB].
5. Paolo Viotti and Marko Vukolić: “[Consistency in Non-Transactional Distributed Storage Systems](http://arxiv.org/abs/1512.00168),” arXiv:1512.00168, 12 April 2016.
6. Maurice P. Herlihy and Jeannette M. Wing: “[Linearizability: A Correctness Condition for Concurrent Objects](http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf),” *ACM Transactions on Programming Languages and Systems* (TOPLAS), volume 12, number 3, pages 463–492, July 1990. [doi:10.1145/78969.78972](http://dx.doi.org/10.1145/78969.78972)
7. Leslie Lamport: “[On interprocess communication](http://research.microsoft.com/en-us/um/people/lamport/pubs/interprocess.pdf),” *Distributed Computing*, volume 1, number 2, pages 77–101, June 1986. [doi:10.1007/BF01786228](http://dx.doi.org/10.1007/BF01786228)
8. David K. Gifford: “[Information Storage in a Decentralized Computer System](http://www.mirrorservice.org/sites/www.bitsavers.org/pdf/xerox/parc/techReports/CSL-81-8_Information_Storage_in_a_Decentralized_Computer_System.pdf),” Xerox Palo Alto Research Centers, CSL-81-8, June 1981.
9. Martin Kleppmann: “[Please Stop Calling Databases CP or AP](http://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html),” *martin.kleppmann.com*, May 11, 2015.
10. Kyle Kingsbury: “[Call Me Maybe: MongoDB Stale Reads](https://aphyr.com/posts/322-call-me-maybe-mongodb-stale-reads),” *aphyr.com*, April 20, 2015.
11. Kyle Kingsbury: “[Computational Techniques in Knossos](https://aphyr.com/posts/314-computational-techniques-in-knossos),” *aphyr.com*, May 17, 2014.
12. Peter Bailis: “[Linearizability Versus Serializability](http://www.bailis.org/blog/linearizability-versus-serializability/),” *bailis.org*, September 24, 2014.
13. Philip A. Bernstein, Vassos Hadzilacos, and Nathan Goodman: [*Concurrency Control and Recovery in Database Systems*](http://research.microsoft.com/en-us/people/philbe/ccontrol.aspx). Addison-Wesley, 1987. ISBN: 978-0-201-10715-9, available online at *research.microsoft.com*.
14. Mike Burrows: “[The Chubby Lock Service for Loosely-Coupled Distributed Systems](http://research.google.com/archive/chubby.html),” at *7th USENIX Symposium on Operating System Design and Implementation* (OSDI), November 2006.
15. Flavio P. Junqueira and Benjamin Reed: *ZooKeeper: Distributed Process Coordination*. O'Reilly Media, 2013. ISBN: 978-1-449-36130-3
16. “[etcd 2.0.12 Documentation](https://coreos.com/etcd/docs/2.0.12/),” CoreOS, Inc., 2015.
17. “[Apache Curator](http://curator.apache.org/),” Apache Software Foundation, *curator.apache.org*, 2015.
18. Morali Vallath: *Oracle 10g RAC Grid, Services & Clustering*. Elsevier Digital Press, 2006. ISBN: 978-1-555-58321-7
19. Peter Bailis, Alan Fekete, Michael J Franklin, et al.: “[Coordination-Avoiding Database Systems](http://arxiv.org/pdf/1402.2237.pdf),” *Proceedings of the VLDB Endowment*, volume 8, number 3, pages 185–196, November 2014.
20. Kyle Kingsbury: “[Call Me Maybe: etcd and Consul](https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul),” *aphyr.com*, June 9, 2014.
21. Flavio P. Junqueira, Benjamin C. Reed, and Marco Serafini: “[Zab: High-Performance Broadcast for Primary-Backup Systems](https://pdfs.semanticscholar.org/b02c/6b00bd5dbdbd951fddb00b906c82fa80f0b3.pdf),” at *41st IEEE International Conference on Dependable Systems and Networks* (DSN), June 2011. [doi:10.1109/DSN.2011.5958223](http://dx.doi.org/10.1109/DSN.2011.5958223)
22. Diego Ongaro and John K. Ousterhout: “[In Search of an Understandable Consensus Algorithm (Extended Version)](http://ramcloud.stanford.edu/raft.pdf),” at *USENIX Annual Technical Conference* (ATC), June 2014.
23. Hagit Attiya, Amotz Bar-Noy, and Danny Dolev: “[Sharing Memory Robustly in Message-Passing Systems](http://www.cse.huji.ac.il/course/2004/dist/p124-attiya.pdf),” *Journal of the ACM*, volume 42, number 1, pages 124–142, January 1995. [doi:10.1145/200836.200869](http://dx.doi.org/10.1145/200836.200869)
24. Nancy Lynch and Alex Shvartsman: “[Robust Emulation of Shared Memory Using Dynamic Quorum-Acknowledged Broadcasts](http://groups.csail.mit.edu/tds/papers/Lynch/FTCS97.pdf),” at *27th Annual International Symposium on Fault-Tolerant Computing* (FTCS), June 1997. [doi:10.1109/FTCS.1997.614100](http://dx.doi.org/10.1109/FTCS.1997.614100)
25. Christian Cachin, Rachid Guerraoui, and Luís Rodrigues: [*Introduction to Reliable and Secure Distributed Programming*](http://www.distributedprogramming.net/), 2nd edition. Springer, 2011. ISBN: 978-3-642-15259-7, [doi:10.1007/978-3-642-15260-3](http://dx.doi.org/10.1007/978-3-642-15260-3)
26. Sam Elliott, Mark Allen, and Martin Kleppmann: [personal communication](https://twitter.com/lenary/status/654761711933648896), thread on *twitter.com*, October 15, 2015.
27. Niklas Ekström, Mikhail Panchenko, and Jonathan Ellis: “[Possible Issue with Read Repair?](http://mail-archives.apache.org/mod_mbox/cassandra-dev/201210.mbox/),” email thread on *cassandra-dev* mailing list, October 2012.
28. Maurice P. Herlihy: “[Wait-Free Synchronization](https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf),” *ACM Transactions on Programming Languages and Systems* (TOPLAS), volume 13, number 1, pages 124–149, January 1991. [doi:10.1145/114005.102808](http://dx.doi.org/10.1145/114005.102808)
29. Armando Fox and Eric A. Brewer: “[Harvest, Yield, and Scalable Tolerant Systems](http://radlab.cs.berkeley.edu/people/fox/static/pubs/pdf/c18.pdf),” at *7th Workshop on Hot Topics in Operating Systems* (HotOS), March 1999. [doi:10.1109/HOTOS.1999.798396](http://dx.doi.org/10.1109/HOTOS.1999.798396)
30. Seth Gilbert and Nancy Lynch: “[Brewer’s Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services](http://www.comp.nus.edu.sg/~gilbert/pubs/BrewersConjecture-SigAct.pdf),” *ACM SIGACT News*, volume 33, number 2, pages 51–59, June 2002. [doi:10.1145/564585.564601](http://dx.doi.org/10.1145/564585.564601)
31. Seth Gilbert and Nancy Lynch: “[Perspectives on the CAP Theorem](http://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf),” *IEEE Computer Magazine*, volume 45, number 2, pages 30–36, February 2012. [doi:10.1109/MC.2011.389](http://dx.doi.org/10.1109/MC.2011.389)
32. Eric A. Brewer: “[CAP Twelve Years Later: How the 'Rules' Have Changed](http://cs609.cs.ua.edu/CAP12.pdf),” *IEEE Computer Magazine*, volume 45, number 2, pages 23–29, February 2012. [doi:10.1109/MC.2012.37](http://dx.doi.org/10.1109/MC.2012.37)
33. Susan B. Davidson, Hector Garcia-Molina, and Dale Skeen: “[Consistency in Partitioned Networks](http://delab.csd.auth.gr/~dimitris/courses/mpc_fall05/papers/invalidation/acm_csur85_partitioned_network_consistency.pdf),” *ACM Computing Surveys*, volume 17, number 3, pages 341–370, September 1985. [doi:10.1145/5505.5508](http://dx.doi.org/10.1145/5505.5508)
34. Paul R. Johnson and Robert H. Thomas: “[RFC 677: The Maintenance of Duplicate Databases](https://tools.ietf.org/html/rfc677),” Network Working Group, January 27, 1975.
35. Bruce G. Lindsay, Patricia Griffiths Selinger, C. Galtieri, et al.: “[Notes on Distributed Databases](http://domino.research.ibm.com/library/cyberdig.nsf/papers/A776EC17FC2FCE73852579F100578964/$File/RJ2571.pdf),” IBM Research, Research Report RJ2571(33471), July 1979.
36. Michael J. Fischer and Alan Michael: “[Sacrificing Serializability to Attain High Availability of Data in an Unreliable Network](http://www.cs.ucsb.edu/~agrawal/spring2011/ugrad/p70-fischer.pdf),” at *1st ACM Symposium on Principles of Database Systems* (PODS), March 1982. [doi:10.1145/588111.588124](http://dx.doi.org/10.1145/588111.588124)
37. Eric A. Brewer: “[NoSQL: Past, Present, Future](http://www.infoq.com/presentations/NoSQL-History),” at *QCon San Francisco*, November 2012.
38. Henry Robinson: “[CAP Confusion: Problems with 'Partition Tolerance,'](http://blog.cloudera.com/blog/2010/04/cap-confusion-problems-with-partition-tolerance/)” *blog.cloudera.com*, April 26, 2010.
39. Adrian Cockcroft: “[Migrating to Microservices](http://www.infoq.com/presentations/migration-cloud-native),” at *QCon London*, March 2014.
40. Martin Kleppmann: “[A Critique of the CAP Theorem](http://arxiv.org/abs/1509.05393),” arXiv:1509.05393, September 17, 2015.
41. Nancy A. Lynch: “[A Hundred Impossibility Proofs for Distributed Computing](http://groups.csail.mit.edu/tds/papers/Lynch/podc89.pdf),” at *8th ACM Symposium on Principles of Distributed Computing* (PODC), August 1989. [doi:10.1145/72981.72982](http://dx.doi.org/10.1145/72981.72982)
42. Hagit Attiya, Faith Ellen, and Adam Morrison: “[Limitations of Highly-Available Eventually-Consistent Data Stores](http://www.cs.technion.ac.il/people/mad/online-publications/podc2015-replds.pdf),” at *ACM Symposium on Principles of Distributed Computing* (PODC), July 2015. doi:10.1145/2767386.2767419](http://dx.doi.org/10.1145/2767386.2767419)
43. Peter Sewell, Susmit Sarkar, Scott Owens, et al.: “[x86-TSO: A Rigorous and Usable Programmer's Model for x86 Multiprocessors](http://www.cl.cam.ac.uk/~pes20/weakmemory/cacm.pdf),” *Communications of the ACM*, volume 53, number 7, pages 89–97, July 2010. [doi:10.1145/1785414.1785443](http://dx.doi.org/10.1145/1785414.1785443)
44. Martin Thompson: “[Memory Barriers/Fences](http://mechanical-sympathy.blogspot.co.uk/2011/07/memory-barriersfences.html),” *mechanical-sympathy.blogspot.co.uk*, July 24, 2011.
45. Ulrich Drepper: “[What Every Programmer Should Know About Memory](http://www.akkadia.org/drepper/cpumemory.pdf),” *akkadia.org*, November 21, 2007.
46. Daniel J. Abadi: “[Consistency Tradeoffs in Modern Distributed Database System Design](http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf),” *IEEE Computer Magazine*, volume 45, number 2, pages 37–42, February 2012. [doi:10.1109/MC.2012.33](http://dx.doi.org/10.1109/MC.2012.33)
47. Hagit Attiya and Jennifer L. Welch: “[Sequential Consistency Versus Linearizability](http://courses.csail.mit.edu/6.852/01/papers/p91-attiya.pdf),” *ACM Transactions on Computer Systems* (TOCS), volume 12, number 2, pages 91–122, May 1994. [doi:10.1145/176575.176576](http://dx.doi.org/10.1145/176575.176576)
48. Mustaque Ahamad, Gil Neiger, James E. Burns, et al.: “[Causal Memory: Definitions, Implementation, and Programming](http://www-i2.informatik.rwth-aachen.de/i2/fileadmin/user_upload/documents/Seminar_MCMM11/Causal_memory_1996.pdf),” *Distributed Computing*, volume 9, number 1, pages 37–49, March 1995. [doi:10.1007/BF01784241](http://dx.doi.org/10.1007/BF01784241)
49. Wyatt Lloyd, Michael J. Freedman, Michael Kaminsky, and David G. Andersen: “[Stronger Semantics for Low-Latency Geo-Replicated Storage](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final149.pdf),” at *10th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), April 2013.
50. Marek Zawirski, Annette Bieniusa, Valter Balegas, et al.: “[SwiftCloud: Fault-Tolerant Geo-Replication Integrated All the Way to the Client Machine](http://arxiv.org/abs/1310.3107),” INRIA Research Report 8347, August 2013.
51. Peter Bailis, Ali Ghodsi, Joseph M Hellerstein, and Ion Stoica: “[Bolt-on Causal Consistency](http://db.cs.berkeley.edu/papers/sigmod13-bolton.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), June 2013.
52. Philippe Ajoux, Nathan Bronson, Sanjeev Kumar, et al.: “[Challenges to Adopting Stronger Consistency at Scale](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-ajoux.pdf),” at *15th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2015.
53. Peter Bailis: “[Causality Is Expensive (and What to Do About It)](http://www.bailis.org/blog/causality-is-expensive-and-what-to-do-about-it/),” *bailis.org*, February 5, 2014.
54. Ricardo Gonçalves, Paulo Sérgio Almeida, Carlos Baquero, and Victor Fonte: “[Concise Server-Wide Causality Management for Eventually Consistent Data Stores](http://haslab.uminho.pt/tome/files/global_logical_clocks.pdf),” at *15th IFIP International Conference on Distributed Applications and Interoperable Systems* (DAIS), June 2015. [doi:10.1007/978-3-319-19129-4_6](http://dx.doi.org/10.1007/978-3-319-19129-4_6)
55. Rob Conery: “[A Better ID Generator for PostgreSQL](http://rob.conery.io/2014/05/29/a-better-id-generator-for-postgresql/),” *rob.conery.io*, May 29, 2014.
56. Leslie Lamport: “[Time, Clocks, and the Ordering of Events in a Distributed System](http://research.microsoft.com/en-US/um/people/Lamport/pubs/time-clocks.pdf),” *Communications of the ACM*, volume 21, number 7, pages 558–565, July 1978. [doi:10.1145/359545.359563](http://dx.doi.org/10.1145/359545.359563)
57. Xavier Défago, André Schiper, and Péter Urbán: “[Total Order Broadcast and Multicast Algorithms: Taxonomy and Survey](https://dspace.jaist.ac.jp/dspace/bitstream/10119/4883/1/defago_et_al.pdf),” *ACM Computing Surveys*, volume 36, number 4, pages 372–421, December 2004. [doi:10.1145/1041680.1041682](http://dx.doi.org/10.1145/1041680.1041682)
58. Hagit Attiya and Jennifer Welch: *Distributed Computing: Fundamentals, Simulations and Advanced Topics*, 2nd edition. John Wiley & Sons, 2004. ISBN: 978-0-471-45324-6, [doi:10.1002/0471478210](http://dx.doi.org/10.1002/0471478210)
59. Mahesh Balakrishnan, Dahlia Malkhi, Vijayan Prabhakaran, et al.: “[CORFU: A Shared Log Design for Flash Clusters](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final30.pdf),” at *9th USENIX Symposium on Networked Systems Design and Implementation* (NSDI), April 2012.
60. Fred B. Schneider: “[Implementing Fault-Tolerant Services Using the State Machine Approach: A Tutorial](http://www.cs.cornell.edu/fbs/publications/smsurvey.pdf),” *ACM Computing Surveys*, volume 22, number 4, pages 299–319, December 1990.
61. Alexander Thomson, Thaddeus Diamond, Shu-Chun Weng, et al.: “[Calvin: Fast Distributed Transactions for Partitioned Database Systems](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), May 2012.
62. Mahesh Balakrishnan, Dahlia Malkhi, Ted Wobber, et al.: “[Tango: Distributed Data Structures over a Shared Log](http://research.microsoft.com/pubs/199947/Tango.pdf),” at *24th ACM Symposium on Operating Systems Principles* (SOSP), November 2013. [doi:10.1145/2517349.2522732](http://dx.doi.org/10.1145/2517349.2522732)
63. Robbert van Renesse and Fred B. Schneider: “[Chain Replication for Supporting High Throughput and Availability](http://static.usenix.org/legacy/events/osdi04/tech/full_papers/renesse/renesse.pdf),” at *6th USENIX Symposium on Operating System Design and Implementation* (OSDI), December 2004.
64. Leslie Lamport: “[How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs](http://research-srv.microsoft.com/en-us/um/people/lamport/pubs/multi.pdf),” *IEEE Transactions on Computers*, volume 28, number 9, pages 690–691, September 1979. [doi:10.1109/TC.1979.1675439](http://dx.doi.org/10.1109/TC.1979.1675439)
65. Enis Söztutar, Devaraj Das, and Carter Shanklin: “[Apache HBase High Availability at the Next Level](http://hortonworks.com/blog/apache-hbase-high-availability-next-level/),” *hortonworks.com*, January 22, 2015.
66. Brian F Cooper, Raghu Ramakrishnan, Utkarsh Srivastava, et al.: “[PNUTS: Yahoo!’s Hosted Data Serving Platform](http://www.mpi-sws.org/~druschel/courses/ds/papers/cooper-pnuts.pdf),” at *34th International Conference on Very Large Data Bases* (VLDB), August 2008. [doi:10.14778/1454159.1454167](http://dx.doi.org/10.14778/1454159.1454167)
67. Tushar Deepak Chandra and Sam Toueg: “[Unreliable Failure Detectors for Reliable Distributed Systems](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf),” *Journal of the ACM*, volume 43, number 2, pages 225–267, March 1996. [doi:10.1145/226643.226647](http://dx.doi.org/10.1145/226643.226647)
68. Michael J. Fischer, Nancy Lynch, and Michael S. Paterson: “[Impossibility of Distributed Consensus with One Faulty Process](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf),” *Journal of the ACM*, volume 32, number 2, pages 374–382, April 1985. [doi:10.1145/3149.214121](http://dx.doi.org/10.1145/3149.214121)
69. Michael Ben-Or: “Another Advantage of Free Choice: Completely Asynchronous Agreement Protocols,” at *2nd ACM Symposium on Principles of Distributed Computing* (PODC), August 1983. [doi:10.1145/800221.806707](http://dl.acm.org/citation.cfm?id=806707)
70. Jim N. Gray and Leslie Lamport: “[Consensus on Transaction Commit](http://db.cs.berkeley.edu/cs286/papers/paxoscommit-tods2006.pdf),” *ACM Transactions on Database Systems* (TODS), volume 31, number 1, pages 133–160, March 2006. [doi:10.1145/1132863.1132867](http://dx.doi.org/10.1145/1132863.1132867)
71. Rachid Guerraoui: “[Revisiting the Relationship Between Non-Blocking Atomic Commitment and Consensus](https://pdfs.semanticscholar.org/5d06/489503b6f791aa56d2d7942359c2592e44b0.pdf),” at *9th International Workshop on Distributed Algorithms* (WDAG), September 1995. [doi:10.1007/BFb0022140](http://dx.doi.org/10.1007/BFb0022140)
72. Thanumalayan Sankaranarayana Pillai, Vijay Chidambaram, Ramnatthan Alagappan, et al.: “[All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications](http://research.cs.wisc.edu/wind/Publications/alice-osdi14.pdf),” at *11th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2014.
73. Jim Gray: “[The Transaction Concept: Virtues and Limitations](http://research.microsoft.com/en-us/um/people/gray/papers/theTransactionConcept.pdf),” at *7th International Conference on Very Large Data Bases* (VLDB), September 1981.
74. Hector Garcia-Molina and Kenneth Salem: “[Sagas](http://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), May 1987. [doi:10.1145/38713.38742](http://dx.doi.org/10.1145/38713.38742)
75. C. Mohan, Bruce G. Lindsay, and Ron Obermarck: “[Transaction Management in the R* Distributed Database Management System](https://cs.brown.edu/courses/csci2270/archives/2012/papers/dtxn/p378-mohan.pdf),” *ACM Transactions on Database Systems*, volume 11, number 4, pages 378–396, December 1986. [doi:10.1145/7239.7266](http://dx.doi.org/10.1145/7239.7266)
76. “[Distributed Transaction Processing: The XA Specification](http://pubs.opengroup.org/onlinepubs/009680699/toc.pdf),” X/Open Company Ltd., Technical Standard XO/CAE/91/300, December 1991. ISBN: 978-1-872-63024-3
77. Mike Spille: “[XA Exposed, Part II](http://www.jroller.com/pyrasun/entry/xa_exposed_part_ii_schwartz),” *jroller.com*, April 3, 2004.
78. Ivan Silva Neto and Francisco Reverbel: “[Lessons Learned from Implementing WS-Coordination and WS-AtomicTransaction](http://www.ime.usp.br/~reverbel/papers/icis2008.pdf),” at *7th IEEE/ACIS International Conference on Computer and Information Science* (ICIS), May 2008. [doi:10.1109/ICIS.2008.75](http://dx.doi.org/10.1109/ICIS.2008.75)
79. James E. Johnson, David E. Langworthy, Leslie Lamport, and Friedrich H. Vogt: “[Formal Specification of a Web Services Protocol](http://research.microsoft.com/en-us/um/people/lamport/pubs/wsfm-web.pdf),” at *1st International Workshop on Web Services and Formal Methods* (WS-FM), February 2004. [doi:10.1016/j.entcs.2004.02.022](http://dx.doi.org/10.1016/j.entcs.2004.02.022)
80. Dale Skeen: “[Nonblocking Commit Protocols](http://www.cs.utexas.edu/~lorenzo/corsi/cs380d/papers/Ske81.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), April 1981. [doi:10.1145/582318.582339](http://dx.doi.org/10.1145/582318.582339)
81. Gregor Hohpe: “[Your Coffee Shop Doesn’t Use Two-Phase Commit](http://www.martinfowler.com/ieeeSoftware/coffeeShop.pdf),” *IEEE Software*, volume 22, number 2, pages 64–66, March 2005. [doi:10.1109/MS.2005.52](http://dx.doi.org/10.1109/MS.2005.52)
82. Pat Helland: “[Life Beyond Distributed Transactions: An Apostate’s Opinion](http://www-db.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf),” at *3rd Biennial Conference on Innovative Data Systems Research* (CIDR), January 2007.
83. Jonathan Oliver: “[My Beef with MSDTC and Two-Phase Commits](http://blog.jonathanoliver.com/my-beef-with-msdtc-and-two-phase-commits/),” *blog.jonathanoliver.com*, April 4, 2011.
84. Oren Eini (Ahende Rahien): “[The Fallacy of Distributed Transactions](http://ayende.com/blog/167362/the-fallacy-of-distributed-transactions),” *ayende.com*, July 17, 2014.
85. Clemens Vasters: “[Transactions in Windows Azure (with Service Bus) – An Email Discussion](https://blogs.msdn.microsoft.com/clemensv/2012/07/30/transactions-in-windows-azure-with-service-bus-an-email-discussion/),” *vasters.com*, July 30, 2012.
86. “[Understanding Transactionality in Azure](https://docs.particular.net/nservicebus/azure/understanding-transactionality-in-azure),” NServiceBus Documentation, Particular Software, 2015.
87. Randy Wigginton, Ryan Lowe, Marcos Albe, and Fernando Ipar: “[Distributed Transactions in MySQL](https://www.percona.com/live/mysql-conference-2013/sites/default/files/slides/XA_final.pdf),” at *MySQL Conference and Expo*, April 2013.
88. Mike Spille: “[XA Exposed, Part I](http://www.jroller.com/pyrasun/entry/xa_exposed),” *jroller.com*, April 3, 2004.
89. Ajmer Dhariwal: “[Orphaned MSDTC Transactions (-2 spids)](http://www.eraofdata.com/orphaned-msdtc-transactions-2-spids/),” *eraofdata.com*, December 12, 2008.
90. Paul Randal: “[Real World Story of DBCC PAGE Saving the Day](http://www.sqlskills.com/blogs/paul/real-world-story-of-dbcc-page-saving-the-day/),” *sqlskills.com*, June 19, 2013.
91. “[in-doubt xact resolution Server Configuration Option](https://msdn.microsoft.com/en-us/library/ms179586.aspx),” SQL Server 2016 documentation, Microsoft, Inc., 2016.
92. Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer: “[Consensus in the Presence of Partial Synchrony](http://www.net.t-labs.tu-berlin.de/~petr/ADC-07/papers/DLS88.pdf),” *Journal of the ACM*, volume 35, number 2, pages 288–323, April 1988. [doi:10.1145/42282.42283](http://dx.doi.org/10.1145/42282.42283)
93. Miguel Castro and Barbara H. Liskov: “[Practical Byzantine Fault Tolerance and Proactive Recovery](http://zoo.cs.yale.edu/classes/cs426/2012/bib/castro02practical.pdf),” *ACM Transactions on Computer Systems*, volume 20, number 4, pages 396–461, November 2002. [doi:10.1145/571637.571640](http://dx.doi.org/10.1145/571637.571640)
94. Brian M. Oki and Barbara H. Liskov: “[Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems](http://www.cs.princeton.edu/courses/archive/fall11/cos518/papers/viewstamped.pdf),” at *7th ACM Symposium on Principles of Distributed Computing* (PODC), August 1988. [doi:10.1145/62546.62549](http://dx.doi.org/10.1145/62546.62549)
95. Barbara H. Liskov and James Cowling: “[Viewstamped Replication Revisited](http://pmg.csail.mit.edu/papers/vr-revisited.pdf),” Massachusetts Institute of Technology, Tech Report MIT-CSAIL-TR-2012-021, July 2012.
96. Leslie Lamport: “[The Part-Time Parliament](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf),” *ACM Transactions on Computer Systems*, volume 16, number 2, pages 133–169, May 1998. [doi:10.1145/279227.279229](http://dx.doi.org/10.1145/279227.279229)
97. Leslie Lamport: “[Paxos Made Simple](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf),” *ACM SIGACT News*, volume 32, number 4, pages 51–58, December 2001.
98. Tushar Deepak Chandra, Robert Griesemer, and Joshua Redstone: “[Paxos Made Live – An Engineering Perspective](http://www.read.seas.harvard.edu/~kohler/class/08w-dsi/chandra07paxos.pdf),” at *26th ACM Symposium on Principles of Distributed Computing* (PODC), June 2007.
99. Robbert van Renesse: “[Paxos Made Moderately Complex](http://www.cs.cornell.edu/home/rvr/Paxos/paxos.pdf),” *cs.cornell.edu*, March 2011.
100. Diego Ongaro: “[Consensus: Bridging Theory and Practice](https://github.com/ongardie/dissertation),” PhD Thesis, Stanford University, August 2014.
101. Heidi Howard, Malte Schwarzkopf, Anil Madhavapeddy, and Jon Crowcroft: “[Raft Refloated: Do We Have Consensus?](http://www.cl.cam.ac.uk/~ms705/pub/papers/2015-osr-raft.pdf),” *ACM SIGOPS Operating Systems Review*, volume 49, number 1, pages 12–21, January 2015. [doi:10.1145/2723872.2723876](http://dx.doi.org/10.1145/2723872.2723876)
102. André Medeiros: “[ZooKeeper’s Atomic Broadcast Protocol: Theory and Practice](http://www.tcs.hut.fi/Studies/T-79.5001/reports/2012-deSouzaMedeiros.pdf),” Aalto University School of Science, March 20, 2012.
103. Robbert van Renesse, Nicolas Schiper, and Fred B. Schneider: “[Vive La Différence: Paxos vs. Viewstamped Replication vs. Zab](http://arxiv.org/abs/1309.5671),” *IEEE Transactions on Dependable and Secure Computing*, volume 12, number 4, pages 472–484, September 2014. [doi:10.1109/TDSC.2014.2355848](http://dx.doi.org/10.1109/TDSC.2014.2355848)
104. Will Portnoy: “[Lessons Learned from Implementing Paxos](http://blog.willportnoy.com/2012/06/lessons-learned-from-paxos.html),” *blog.willportnoy.com*, June 14, 2012.
105. Heidi Howard, Dahlia Malkhi, and Alexander Spiegelman: “[Flexible Paxos: Quorum Intersection Revisited](https://arxiv.org/abs/1608.06696),” *arXiv:1608.06696*, August 24, 2016.
106. Heidi Howard and Jon Crowcroft: “[Coracle: Evaluating Consensus at the Internet Edge](http://www.sigcomm.org/sites/default/files/ccr/papers/2015/August/2829988-2790010.pdf),” at *Annual Conference of the ACM Special Interest Group on Data Communication* (SIGCOMM), August 2015. [doi:10.1145/2829988.2790010](http://dx.doi.org/10.1145/2829988.2790010)
107. Kyle Kingsbury: “[Call Me Maybe: Elasticsearch 1.5.0](https://aphyr.com/posts/323-call-me-maybe-elasticsearch-1-5-0),” *aphyr.com*, April 27, 2015.
108. Ivan Kelly: “[BookKeeper Tutorial](https://github.com/ivankelly/bookkeeper-tutorial),” *github.com*, October 2014.
109. Camille Fournier: “[Consensus Systems for the Skeptical Architect](http://www.ustream.tv/recorded/61483409),” at *Craft Conference*, Budapest, Hungary, April 2015.
110. Kenneth P. Birman: “[A History of the Virtual Synchrony Replication Model](https://www.truststc.org/pubs/713.html),” in *Replication: Theory and Practice*, Springer LNCS volume 5959, chapter 6, pages 91–120, 2010. ISBN: 978-3-642-11293-5, [doi:10.1007/978-3-642-11294-2_6](http://dx.doi.org/10.1007/978-3-642-11294-2_6)