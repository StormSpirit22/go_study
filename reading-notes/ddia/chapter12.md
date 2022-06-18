# 第十二章：数据系统的未来

## 数据集成

### 派生数据与分布式事务

分布式事务通过使用锁机制进行互斥来决定写操作的顺序（请参阅第7章“两阶段加锁”），而 CDC（变更数据捕获） 和事件源使用日志进行排序。分布式事务使用原子提交来确保更改只生效一次， 而基于日志的系统通常基于确定性重试和幂等性。

最大的不同在于事务系统通常提供线性化（请参阅第9章“ 线性化”），它可以保证读自己的写，等一致性（请参阅第5章“读自己的写 ”）。 另一方面， 派生的数据系统通常是异步更新的，所以默认情况下它们无怯提供类似级别保证。

但是分布式事务的 XA 规范的容错性和性能不尽如人意，限制了实用性。

### Lambda 架构

Lambda 架构体系结构的核心思想是进来的数据以不可变事件形式追加写到不断增长的数据集，类似于事件源（请参阅第 11 章的“事件溯据”）。基于这些总事件，可以派生出读优化的视图。lambda 结构建议并行运行两个不同的系统 ：一个批处理系统如Hadoop MapReduce，以及一个单独的流处理系统，如 Storm 。

在 lambda 方法中，流处理器处理事件并快速产生对视图的近似更新；批处理系统则处理同一组事件并产生派生视图的校正版本。因为批处理更简单，不易出现错误，而流处理器则相对不太可靠，难以实现容错。而且流处理可以使用快速的近似算法，而批处理只能用较慢的精确算法。

## 分拆数据库

### 编排多种数据存储技术

#### 创建一个索引

create index 过程：

- 扫描表的一致性快照，挑选出所有被索引的字段值，对它们进行排序，然后得到索引。
- 处理从一致性快照创建以来所累计的写入操作。之后，只要有事务写入表中， 数据库就必须持续保持索引处于最新状态。

#### 分离式数据库

传统的同步写依赖于跨异构存储系统的分布式事务 [18]，我认为这是个错误的解决方案（ 请参阅本章前面“派生数据与分布式事务 ”）。在单个存储系统内或流处理系统内的事务是可行的，但是当数据跨越不同技术的边界时，我认为具有幂等写入的异步事件日志是一种更加健壮和可行的方法。



### 幂等性

#### 操作标识符

可以为操作生成一个唯一的标识符（如 UUID ），并将其作为一个隐藏的表单字段包含在客户机应用程序中：或者对所有相关表单字段计算一个哈希值依次来代表操作 ID [3] 。如果 Web浏览器两次提交 POST请求，则两个请求具有相同的操作 ID 。然后可以将该操作 ID 一直传递到数据库，检查并确保对一个给定的 ID 只执行一个操作，参见示例 12-2 。

**例 12-2 使用唯一 ID 来抑制重复请求**

```sql
ALTER TABLE requests ADD UNIQUE (request_id);

BEGIN TRANSACTION;
    INSERT INTO requests
        (request_id, from_account, to_account, amount)
        VALUES('0286FDB8-D7E1-423F-B40B-792B3608036C', 4321, 1234, 11.00);
    UPDATE accounts SET balance = balance + 11.00 WHERE account_id = 1234;
    UPDATE accounts SET balance = balance - 11.00 WHERE account_id = 4321;
COMMIT;
```

示例 12-2 依赖于request_id 列上的唯一性约束。如果一个事务试图插入一个已存在的 ID，那么 INSERT 失败，事务被中止而退出 ，这样无法执行两次。即使在弱隔离级别下，关系型数据库通常也可以正确地确保唯一性约束，而“应用程序采用先检查再插入”的方式可能会在不可串行化隔离下失败，参见第7章“写倾斜与幻读”的例 子。

除了消除重复请求之外，示例 12-2 中 requests 表作为一种事件日志，还可以表征事件源（请参阅第 11 章 “事件溯源”）。账户余额的更新操作不一定要和插入处于同一个事务内，它们可以是冗余的，从下游消费者的请求事件中派生而来，而基于唯一请求 ID 可以确保事件被处理一次 。

### 强制约束

#### 唯一性约束需要达成共识

保证唯一性约束需要达成共识，达成这一共识的最常见的方式是将单一节点作为主节点，井负责作出所有的决定。

采用分区方案可以提高唯一性检查的扩展性，即基于要求唯一性的字段进行分区。例如，如果需要通过请求标识来确保唯一性，参见示例 12-2 ，可以把相同请求标识的所有请求都路由到同一分区（请参阅第6章）。如果需要用户名唯一，可以通过用户名的哈希值进行分区。

但是，它无法支持异步的多主节点复制，因为可能会发生不同的主节点同时接受冲突写入，无法保证值的唯一（ 请参阅第 9 章“实现线性化系统”）。如果想达到立即拒绝任何违反约束的写请求，则同步式的协调不可避免 [56]。

#### 分区请求处理

任何可能冲突的写入都被路由到特定的分区并按顺序处理（比如通过用户名的哈希值分区，将对同一个用户的请求都在一个分区内按顺序处理）。这个方法不仅适用于唯一性约束，而且适用于许多其他类型的约束。

#### 多分区请求处理

比如在示例 12-2 中，可能有三个分区：一个包含请求 ID，一个包含收款人账户，另一个包含付款人账户。

使用分区日志是可以实现同等的正确性，并且不需要原子提交：

1. 账户 A 向账户 B 转账的请求由客户端赋予一个唯一的请求 ID，基于该请求 ID 追加到对应的日志分区。

2. 流处理系统读取请求日志。对于每个请求消息，它发出两条输出消息：到付款人账户 A 的付款指令（按照 A 进行分区），以及到收款人账户 B （按照 B 进行分区） 的信用指令。原始的请求 ID 都包含在这两条消息中。

3. 后续操作接收上述指令，并通过请求 ID 进行重复数据消除，将最后的更改应用于账户余额。

为了避免处理分布式事务，我们首先持久地将请求记录为单条消息，然后从第一条消息中派生出信用和付款两条指令。单一对象写入在几乎所有的数据系统中都是原子的。

如果步骤 2 的流处理操作发生崩溃，则从上一个快照检查点来恢复处理。这样做不会跳过任何请求消息，但它可能会多次处理请求，并产生重复的信用和付款指令。 但是由于它是确定性的，它只会再次产生相同的指令，步骤 3 中的处理可以基于端到端的请求 ID 来轻松实现重复数据消除。

如果还想确保付款人账户不会发生透支，可以添加另一个流处理操作符（按照付款人账号进行分区）维护账户余额井验证转账金额。在步骤 1 中，只有有效的事务才能放入到请求日志中 。

#### 数据流系统的正确性

如上面所述，可靠的流处理系统可以在不需要分布式事务和原子提交协议的情况下保持完整性，这意味着它们可以实现等价的正确性，同时具有更好的性能和操作鲁棒性。 我们主要是通过以下机制实现了这一完整性：

- 将写入操作的内容表示为单条消息，可以轻松地采用原子方式编写，这种方法非常适合事件源（请参阅第 11 章“事件溯源”）。


- 使用确定性派生函数从该条消息派生所有其他状态的更新操作（与存储过程类似） 。
- 通过所有这些级别的处理来传递客户端生成的请求 ID，实现端到端重复消除和幂等性。
- 消息不可变，并支持多次重新处理派生数据，从而使错误恢复变得更容易（请参阅第 11 章“不可变事件的优势”）。



## 参考文献

1. Rachid Belaid: “[Postgres Full-Text Search is Good Enough!](http://rachbelaid.com/postgres-full-text-search-is-good-enough/),” *rachbelaid.com*, July 13, 2015.
2. Philippe Ajoux, Nathan Bronson, Sanjeev Kumar, et al.: “[Challenges to Adopting Stronger Consistency at Scale](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-ajoux.pdf),” at *15th USENIX Workshop on Hot Topics in Operating Systems* (HotOS), May 2015.
3. Pat Helland and Dave Campbell: “[Building on Quicksand](https://database.cs.wisc.edu/cidr/cidr2009/Paper_133.pdf),” at *4th Biennial Conference on Innovative Data Systems Research* (CIDR), January 2009.
4. Jessica Kerr: “[Provenance and Causality in Distributed Systems](http://blog.jessitron.com/2016/09/provenance-and-causality-in-distributed.html),” *blog.jessitron.com*, September 25, 2016.
5. Kostas Tzoumas: “[Batch Is a Special Case of Streaming](http://data-artisans.com/batch-is-a-special-case-of-streaming/),” *data-artisans.com*, September 15, 2015.
6. Shinji Kim and Robert Blafford: “[Stream Windowing Performance Analysis: Concord and Spark Streaming](http://concord.io/posts/windowing_performance_analysis_w_spark_streaming),” *concord.io*, July 6, 2016.
7. Jay Kreps: “[The Log: What Every Software Engineer Should Know About Real-Time Data's Unifying Abstraction](http://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying),” *engineering.linkedin.com*, December 16, 2013.
8. Pat Helland: “[Life Beyond Distributed Transactions: An Apostate’s Opinion](http://www-db.cs.wisc.edu/cidr/cidr2007/papers/cidr07p15.pdf),” at *3rd Biennial Conference on Innovative Data Systems Research* (CIDR), January 2007.
9. “[Great Western Railway (1835–1948)](https://www.networkrail.co.uk/VirtualArchive/great-western/),” Network Rail Virtual Archive, *networkrail.co.uk*.
10. Jacqueline Xu: “[Online Migrations at Scale](https://stripe.com/blog/online-migrations),” *stripe.com*, February 2, 2017.
11. Molly Bartlett Dishman and Martin Fowler: “[Agile Architecture](http://conferences.oreilly.com/software-architecture/sa2015/public/schedule/detail/40388),” at *O'Reilly Software Architecture Conference*, March 2015.
12. Nathan Marz and James Warren: [*Big Data: Principles and Best Practices of Scalable Real-Time Data Systems*](https://www.manning.com/books/big-data). Manning, 2015. ISBN: 978-1-617-29034-3
13. Oscar Boykin, Sam Ritchie, Ian O'Connell, and Jimmy Lin: “[Summingbird: A Framework for Integrating Batch and Online MapReduce Computations](http://www.vldb.org/pvldb/vol7/p1441-boykin.pdf),” at *40th International Conference on Very Large Data Bases* (VLDB), September 2014.
14. Jay Kreps: “[Questioning the Lambda Architecture](https://www.oreilly.com/ideas/questioning-the-lambda-architecture),” *oreilly.com*, July 2, 2014.
15. Raul Castro Fernandez, Peter Pietzuch, Jay Kreps, et al.: “[Liquid: Unifying Nearline and Offline Big Data Integration](http://www.cidrdb.org/cidr2015/Papers/CIDR15_Paper25u.pdf),” at *7th Biennial Conference on Innovative Data Systems Research* (CIDR), January 2015.
16. Dennis M. Ritchie and Ken Thompson: “[The UNIX Time-Sharing System](http://www.cs.virginia.edu/~zaher/classes/CS656/p365-ritchie.pdf),” *Communications of the ACM*, volume 17, number 7, pages 365–375, July 1974. [doi:10.1145/361011.361061](http://dx.doi.org/10.1145/361011.361061)
17. Eric A. Brewer and Joseph M. Hellerstein: “[CS262a: Advanced Topics in Computer Systems](http://people.eecs.berkeley.edu/~brewer/cs262/systemr.html),” lecture notes, University of California, Berkeley, *cs.berkeley.edu*, August 2011.
18. Michael Stonebraker: “[The Case for Polystores](http://wp.sigmod.org/?p=1629),” *wp.sigmod.org*, July 13, 2015.
19. Jennie Duggan, Aaron J. Elmore, Michael Stonebraker, et al.: “[The BigDAWG Polystore System](http://dspace.mit.edu/openaccess-disseminate/1721.1/100936),” *ACM SIGMOD Record*, volume 44, number 2, pages 11–16, June 2015. [doi:10.1145/2814710.2814713](http://dx.doi.org/10.1145/2814710.2814713)
20. Patrycja Dybka: “[Foreign Data Wrappers for PostgreSQL](http://www.vertabelo.com/blog/technical-articles/foreign-data-wrappers-for-postgresql),” *vertabelo.com*, March 24, 2015.
21. David B. Lomet, Alan Fekete, Gerhard Weikum, and Mike Zwilling: “[Unbundling Transaction Services in the Cloud](https://www.microsoft.com/en-us/research/publication/unbundling-transaction-services-in-the-cloud/),” at *4th Biennial Conference on Innovative Data Systems Research* (CIDR), January 2009.
22. Martin Kleppmann and Jay Kreps: “[Kafka, Samza and the Unix Philosophy of Distributed Data](http://martin.kleppmann.com/papers/kafka-debull15.pdf),” *IEEE Data Engineering Bulletin*, volume 38, number 4, pages 4–14, December 2015.
23. John Hugg: “[Winning Now and in the Future: Where VoltDB Shines](https://voltdb.com/blog/winning-now-and-future-where-voltdb-shines),” *voltdb.com*, March 23, 2016.
24. Frank McSherry, Derek G. Murray, Rebecca Isaacs, and Michael Isard: “[Differential Dataflow](http://cidrdb.org/cidr2013/Papers/CIDR13_Paper111.pdf),” at *6th Biennial Conference on Innovative Data Systems Research* (CIDR), January 2013.
25. Derek G Murray, Frank McSherry, Rebecca Isaacs, et al.: “[Naiad: A Timely Dataflow System](http://research.microsoft.com/pubs/201100/naiad_sosp2013.pdf),” at *24th ACM Symposium on Operating Systems Principles* (SOSP), pages 439–455, November 2013. [doi:10.1145/2517349.2522738](http://dx.doi.org/10.1145/2517349.2522738)
26. Gwen Shapira: “[We have a bunch of customers who are implementing ‘database inside-out’ concept and they all ask ‘is anyone else doing it? are we crazy?’](https://twitter.com/gwenshap/status/758800071110430720)” *twitter.com*, July 28, 2016.
27. Martin Kleppmann: “[Turning the Database Inside-out with Apache Samza,](http://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html)” at *Strange Loop*, September 2014.
28. Peter Van Roy and Seif Haridi: [*Concepts, Techniques, and Models of Computer Programming*](http://www.epsa.org/forms/uploadFiles/3B6300000000.filename.booksingle.pdf). MIT Press, 2004. ISBN: 978-0-262-22069-9
29. “[Juttle Documentation](http://juttle.github.io/juttle/),” *juttle.github.io*, 2016.
30. Evan Czaplicki and Stephen Chong: “[Asynchronous Functional Reactive Programming for GUIs](http://people.seas.harvard.edu/~chong/pubs/pldi13-elm.pdf),” at *34th ACM SIGPLAN Conference on Programming Language Design and Implementation* (PLDI), June 2013. [doi:10.1145/2491956.2462161](http://dx.doi.org/10.1145/2491956.2462161)
31. Engineer Bainomugisha, Andoni Lombide Carreton, Tom van Cutsem, Stijn Mostinckx, and Wolfgang de Meuter: “[A Survey on Reactive Programming](http://soft.vub.ac.be/Publications/2012/vub-soft-tr-12-13.pdf),” *ACM Computing Surveys*, volume 45, number 4, pages 1–34, August 2013. [doi:10.1145/2501654.2501666](http://dx.doi.org/10.1145/2501654.2501666)
32. Peter Alvaro, Neil Conway, Joseph M. Hellerstein, and William R. Marczak: “[Consistency Analysis in Bloom: A CALM and Collected Approach](http://www.eecs.berkeley.edu/~palvaro/cidr11.pdf),” at *5th Biennial Conference on Innovative Data Systems Research* (CIDR), January 2011.
33. Felienne Hermans: “[Spreadsheets Are Code](https://vimeo.com/145492419),” at *Code Mesh*, November 2015.
34. Dan Bricklin and Bob Frankston: “[VisiCalc: Information from Its Creators](http://danbricklin.com/visicalc.htm),” *danbricklin.com*.
35. D. Sculley, Gary Holt, Daniel Golovin, et al.: “[Machine Learning: The High-Interest Credit Card of Technical Debt](http://research.google.com/pubs/pub43146.html),” at *NIPS Workshop on Software Engineering for Machine Learning* (SE4ML), December 2014.
36. Peter Bailis, Alan Fekete, Michael J Franklin, et al.: “[Feral Concurrency Control: An Empirical Investigation of Modern Application Integrity](http://www.bailis.org/papers/feral-sigmod2015.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), June 2015. [doi:10.1145/2723372.2737784](http://dx.doi.org/10.1145/2723372.2737784)
37. Guy Steele: “[Re: Need for Macros (Was Re: Icon)](https://people.csail.mit.edu/gregs/ll1-discuss-archive-html/msg01134.html),” email to *ll1-discuss* mailing list, *people.csail.mit.edu*, December 24, 2001.
38. David Gelernter: “[Generative Communication in Linda](http://cseweb.ucsd.edu/groups/csag/html/teaching/cse291s03/Readings/p80-gelernter.pdf),” *ACM Transactions on Programming Languages and Systems* (TOPLAS), volume 7, number 1, pages 80–112, January 1985. [doi:10.1145/2363.2433](http://dx.doi.org/10.1145/2363.2433)
39. Patrick Th. Eugster, Pascal A. Felber, Rachid Guerraoui, and Anne-Marie Kermarrec: “[The Many Faces of Publish/Subscribe](http://www.cs.ru.nl/~pieter/oss/manyfaces.pdf),” *ACM Computing Surveys*, volume 35, number 2, pages 114–131, June 2003. [doi:10.1145/857076.857078](http://dx.doi.org/10.1145/857076.857078)
40. Ben Stopford: “[Microservices in a Streaming World](https://www.infoq.com/presentations/microservices-streaming),” at *QCon London*, March 2016.
41. Christian Posta: “[Why Microservices Should Be Event Driven: Autonomy vs Authority](http://blog.christianposta.com/microservices/why-microservices-should-be-event-driven-autonomy-vs-authority/),” *blog.christianposta.com*, May 27, 2016.
42. Alex Feyerke: “[Say Hello to Offline First](http://hood.ie/blog/say-hello-to-offline-first.html),” *hood.ie*, November 5, 2013.
43. Sebastian Burckhardt, Daan Leijen, Jonathan Protzenko, and Manuel Fähndrich: “[Global Sequence Protocol: A Robust Abstraction for Replicated Shared State](http://drops.dagstuhl.de/opus/volltexte/2015/5238/),” at *29th European Conference on Object-Oriented Programming* (ECOOP), July 2015. [doi:10.4230/LIPIcs.ECOOP.2015.568](http://dx.doi.org/10.4230/LIPIcs.ECOOP.2015.568)
44. Mark Soper: “[Clearing Up React Data Management Confusion with Flux, Redux, and Relay](https://medium.com/@marksoper/clearing-up-react-data-management-confusion-with-flux-redux-and-relay-aad504e63cae),” *medium.com*, December 3, 2015.
45. Eno Thereska, Damian Guy, Michael Noll, and Neha Narkhede: “[Unifying Stream Processing and Interactive Queries in Apache Kafka](http://www.confluent.io/blog/unifying-stream-processing-and-interactive-queries-in-apache-kafka/),” *confluent.io*, October 26, 2016.
46. Frank McSherry: “[Dataflow as Database](https://github.com/frankmcsherry/blog/blob/master/posts/2016-07-17.md),” *github.com*, July 17, 2016.
47. Peter Alvaro: “[I See What You Mean](https://www.youtube.com/watch?v=R2Aa4PivG0g),” at *Strange Loop*, September 2015.
48. Nathan Marz: “[Trident: A High-Level Abstraction for Realtime Computation](https://blog.twitter.com/2012/trident-a-high-level-abstraction-for-realtime-computation),” *blog.twitter.com*, August 2, 2012.
49. Edi Bice: “[Low Latency Web Scale Fraud Prevention with Apache Samza, Kafka and Friends](http://www.slideshare.net/edibice/extremely-low-latency-web-scale-fraud-prevention-with-apache-samza-kafka-and-friends),” at *Merchant Risk Council MRC Vegas Conference*, March 2016.
50. Charity Majors: “[The Accidental DBA](https://charity.wtf/2016/10/02/the-accidental-dba/),” *charity.wtf*, October 2, 2016.
51. Arthur J. Bernstein, Philip M. Lewis, and Shiyong Lu: “[Semantic Conditions for Correctness at Different Isolation Levels](http://db.cs.berkeley.edu/cs286/papers/isolation-icde2000.pdf),” at *16th International Conference on Data Engineering* (ICDE), February 2000. [doi:10.1109/ICDE.2000.839387](http://dx.doi.org/10.1109/ICDE.2000.839387)
52. Sudhir Jorwekar, Alan Fekete, Krithi Ramamritham, and S. Sudarshan: “[Automating the Detection of Snapshot Isolation Anomalies](http://www.vldb.org/conf/2007/papers/industrial/p1263-jorwekar.pdf),” at *33rd International Conference on Very Large Data Bases* (VLDB), September 2007.
53. Kyle Kingsbury: [Jepsen blog post series](https://aphyr.com/tags/jepsen), *aphyr.com*, 2013–2016.
54. Michael Jouravlev: “[Redirect After Post](http://www.theserverside.com/news/1365146/Redirect-After-Post),” *theserverside.com*, August 1, 2004.
55. Jerome H. Saltzer, David P. Reed, and David D. Clark: “[End-to-End Arguments in System Design](http://www.ece.drexel.edu/courses/ECE-C631-501/SalRee1984.pdf),” *ACM Transactions on Computer Systems*, volume 2, number 4, pages 277–288, November 1984. [doi:10.1145/357401.357402](http://dx.doi.org/10.1145/357401.357402)
56. Peter Bailis, Alan Fekete, Michael J. Franklin, et al.: “[Coordination-Avoiding Database Systems](http://arxiv.org/pdf/1402.2237.pdf),” *Proceedings of the VLDB Endowment*, volume 8, number 3, pages 185–196, November 2014.
57. Alex Yarmula: “[Strong Consistency in Manhattan](https://blog.twitter.com/2016/strong-consistency-in-manhattan),” *blog.twitter.com*, March 17, 2016.
58. Douglas B Terry, Marvin M Theimer, Karin Petersen, et al.: “[Managing Update Conflicts in Bayou, a Weakly Connected Replicated Storage System](http://css.csail.mit.edu/6.824/2014/papers/bayou-conflicts.pdf),” at *15th ACM Symposium on Operating Systems Principles* (SOSP), pages 172–182, December 1995. [doi:10.1145/224056.224070](http://dx.doi.org/10.1145/224056.224070)
59. Jim Gray: “[The Transaction Concept: Virtues and Limitations](http://research.microsoft.com/en-us/um/people/gray/papers/theTransactionConcept.pdf),” at *7th International Conference on Very Large Data Bases* (VLDB), September 1981.
60. Hector Garcia-Molina and Kenneth Salem: “[Sagas](http://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf),” at *ACM International Conference on Management of Data* (SIGMOD), May 1987. [doi:10.1145/38713.38742](http://dx.doi.org/10.1145/38713.38742)
61. Pat Helland: “[Memories, Guesses, and Apologies](http://blogs.msdn.com/b/pathelland/archive/2007/05/15/memories-guesses-and-apologies.aspx),” *blogs.msdn.com*, May 15, 2007.
62. Yoongu Kim, Ross Daly, Jeremie Kim, et al.: “[Flipping Bits in Memory Without Accessing Them: An Experimental Study of DRAM Disturbance Errors](https://users.ece.cmu.edu/~yoonguk/papers/kim-isca14.pdf),” at *41st Annual International Symposium on Computer Architecture* (ISCA), June 2014. [doi:10.1145/2678373.2665726](http://dx.doi.org/10.1145/2678373.2665726)
63. Mark Seaborn and Thomas Dullien: “[Exploiting the DRAM Rowhammer Bug to Gain Kernel Privileges](https://googleprojectzero.blogspot.co.uk/2015/03/exploiting-dram-rowhammer-bug-to-gain.html),” *googleprojectzero.blogspot.co.uk*, March 9, 2015.
64. Jim N. Gray and Catharine van Ingen: “[Empirical Measurements of Disk Failure Rates and Error Rates](https://www.microsoft.com/en-us/research/publication/empirical-measurements-of-disk-failure-rates-and-error-rates/),” Microsoft Research, MSR-TR-2005-166, December 2005.
65. Annamalai Gurusami and Daniel Price: “[Bug #73170: Duplicates in Unique Secondary Index Because of Fix of Bug#68021](http://bugs.mysql.com/bug.php?id=73170),” *bugs.mysql.com*, July 2014.
66. Gary Fredericks: “[Postgres Serializability Bug](https://github.com/gfredericks/pg-serializability-bug),” *github.com*, September 2015.
67. Xiao Chen: “[HDFS DataNode Scanners and Disk Checker Explained](http://blog.cloudera.com/blog/2016/12/hdfs-datanode-scanners-and-disk-checker-explained/),” *blog.cloudera.com*, December 20, 2016.
68. Jay Kreps: “[Getting Real About Distributed System Reliability](http://blog.empathybox.com/post/19574936361/getting-real-about-distributed-system-reliability),” *blog.empathybox.com*, March 19, 2012.
69. Martin Fowler: “[The LMAX Architecture](http://martinfowler.com/articles/lmax.html),” *martinfowler.com*, July 12, 2011.
70. Sam Stokes: “[Move Fast with Confidence](http://blog.samstokes.co.uk/blog/2016/07/11/move-fast-with-confidence/),” *blog.samstokes.co.uk*, July 11, 2016.
71. “[Sawtooth Lake Documentation](http://intelledger.github.io/introduction.html),” Intel Corporation, *intelledger.github.io*, 2016.
72. Richard Gendal Brown: “[Introducing R3 Corda™: A Distributed Ledger Designed for Financial Services](https://gendal.me/2016/04/05/introducing-r3-corda-a-distributed-ledger-designed-for-financial-services/),” *gendal.me*, April 5, 2016.
73. Trent McConaghy, Rodolphe Marques, Andreas Müller, et al.: “[BigchainDB: A Scalable Blockchain Database](https://www.bigchaindb.com/whitepaper/bigchaindb-whitepaper.pdf),” *bigchaindb.com*, June 8, 2016.
74. Ralph C. Merkle: “[A Digital Signature Based on a Conventional Encryption Function](https://people.eecs.berkeley.edu/~raluca/cs261-f15/readings/merkle.pdf),” at *CRYPTO '87*, August 1987. [doi:10.1007/3-540-48184-2_32](http://dx.doi.org/10.1007/3-540-48184-2_32)
75. Ben Laurie: “[Certificate Transparency](http://queue.acm.org/detail.cfm?id=2668154),” *ACM Queue*, volume 12, number 8, pages 10-19, August 2014. [doi:10.1145/2668152.2668154](http://dx.doi.org/10.1145/2668152.2668154)
76. Mark D. Ryan: “[Enhanced Certificate Transparency and End-to-End Encrypted Mail](http://www.internetsociety.org/doc/enhanced-certificate-transparency-and-end-end-encrypted-mail),” at *Network and Distributed System Security Symposium* (NDSS), February 2014. [doi:10.14722/ndss.2014.23379](http://dx.doi.org/10.14722/ndss.2014.23379)
77. “[Software Engineering Code of Ethics and Professional Practice](http://www.acm.org/about/se-code),” Association for Computing Machinery, *acm.org*, 1999.
78. François Chollet: “[Software development is starting to involve important ethical choices](https://twitter.com/fchollet/status/792958695722201088),” *twitter.com*, October 30, 2016.
79. Igor Perisic: “[Making Hard Choices: The Quest for Ethics in Machine Learning](https://engineering.linkedin.com/blog/2016/11/making-hard-choices--the-quest-for-ethics-in-machine-learning),” *engineering.linkedin.com*, November 2016.
80. John Naughton: “[Algorithm Writers Need a Code of Conduct](https://www.theguardian.com/commentisfree/2015/dec/06/algorithm-writers-should-have-code-of-conduct),” *theguardian.com*, December 6, 2015.
81. Logan Kugler: “[What Happens When Big Data Blunders?](http://cacm.acm.org/magazines/2016/6/202655-what-happens-when-big-data-blunders/fulltext),” *Communications of the ACM*, volume 59, number 6, pages 15–16, June 2016. [doi:10.1145/2911975](http://dx.doi.org/10.1145/2911975)
82. Bill Davidow: “[Welcome to Algorithmic Prison](http://www.theatlantic.com/technology/archive/2014/02/welcome-to-algorithmic-prison/283985/),” *theatlantic.com*, February 20, 2014.
83. Don Peck: “[They're Watching You at Work](http://www.theatlantic.com/magazine/archive/2013/12/theyre-watching-you-at-work/354681/),” *theatlantic.com*, December 2013.
84. Leigh Alexander: “[Is an Algorithm Any Less Racist Than a Human?](https://www.theguardian.com/technology/2016/aug/03/algorithm-racist-human-employers-work)” *theguardian.com*, August 3, 2016.
85. Jesse Emspak: “[How a Machine Learns Prejudice](https://www.scientificamerican.com/article/how-a-machine-learns-prejudice/),” *scientificamerican.com*, December 29, 2016.
86. Maciej Cegłowski: “[The Moral Economy of Tech](http://idlewords.com/talks/sase_panel.htm),” *idlewords.com*, June 2016.
87. Cathy O'Neil: [*Weapons of Math Destruction: How Big Data Increases Inequality and Threatens Democracy*](https://weaponsofmathdestructionbook.com/). Crown Publishing, 2016. ISBN: 978-0-553-41881-1
88. Julia Angwin: “[Make Algorithms Accountable](http://www.nytimes.com/2016/08/01/opinion/make-algorithms-accountable.html),” *nytimes.com*, August 1, 2016.
89. Bryce Goodman and Seth Flaxman: “[European Union Regulations on Algorithmic Decision-Making and a ‘Right to Explanation’](https://arxiv.org/abs/1606.08813),” *arXiv:1606.08813*, August 31, 2016.
90. “[A Review of the Data Broker Industry: Collection, Use, and Sale of Consumer Data for Marketing Purposes](https://www.commerce.senate.gov/public/index.cfm/reports?ID=57C428EC-8F20-44EE-BFB8-A570E9BE0CCC),” Staff Report, *United States Senate Committee on Commerce, Science, and Transportation*, *commerce.senate.gov*, December 2013.
91. Olivia Solon: “[Facebook’s Failure: Did Fake News and Polarized Politics Get Trump Elected?](https://www.theguardian.com/technology/2016/nov/10/facebook-fake-news-election-conspiracy-theories)” *theguardian.com*, November 10, 2016.
92. Donella H. Meadows and Diana Wright: *Thinking in Systems: A Primer*. Chelsea Green Publishing, 2008. ISBN: 978-1-603-58055-7
93. Daniel J. Bernstein: “[Listening to a ‘big data’/‘data science’ talk](https://twitter.com/hashbreaker/status/598076230437568512),” *twitter.com*, May 12, 2015.
94. Marc Andreessen: “[Why Software Is Eating the World](http://genius.com/Marc-andreessen-why-software-is-eating-the-world-annotated),” *The Wall Street Journal*, 20 August 2011.
95. J. M. Porup: “[‘Internet of Things’ Security Is Hilariously Broken and Getting Worse](http://arstechnica.com/security/2016/01/how-to-search-the-internet-of-things-for-photos-of-sleeping-babies/),” *arstechnica.com*, January 23, 2016.
96. Bruce Schneier: [*Data and Goliath: The Hidden Battles to Collect Your Data and Control Your World*](https://www.schneier.com/books/data_and_goliath/). W. W. Norton, 2015. ISBN: 978-0-393-35217-7
97. The Grugq: “[Nothing to Hide](https://grugq.tumblr.com/post/142799983558/nothing-to-hide),” *grugq.tumblr.com*, April 15, 2016.
98. Tony Beltramelli: “[Deep-Spying: Spying Using Smartwatch and Deep Learning](https://arxiv.org/abs/1512.05616),” Masters Thesis, IT University of Copenhagen, December 2015. Available at *arxiv.org/abs/1512.05616*
99. Shoshana Zuboff: “[Big Other: Surveillance Capitalism and the Prospects of an Information Civilization](http://papers.ssrn.com/sol3/papers.cfm?abstract_id=2594754),” *Journal of Information Technology*, volume 30, number 1, pages 75–89, April 2015.[doi:10.1057/jit.2015.5](http://dx.doi.org/10.1057/jit.2015.5)
100. Carina C. Zona: “[Consequences of an Insightful Algorithm](https://www.youtube.com/watch?v=YRI40A4tyWU),” at *GOTO Berlin*, November 2016.
101. Bruce Schneier: “[Data Is a Toxic Asset, So Why Not Throw It Out?](https://www.schneier.com/essays/archives/2016/03/data_is_a_toxic_asse.html),” *schneier.com*, March 1, 2016.
102. John E. Dunn: “[The UK’s 15 Most Infamous Data Breaches](http://www.techworld.com/security/uks-most-infamous-data-breaches-2016-3604586/),” *techworld.com*, November 18, 2016.
103. Cory Scott: “[Data is not toxic - which implies no benefit - but rather hazardous material, where we must balance need vs. want](https://twitter.com/cory_scott/status/706586399483437056),” *twitter.com*, March 6, 2016.
104. Bruce Schneier: “[Mission Creep: When Everything Is Terrorism](https://www.schneier.com/essays/archives/2013/07/mission_creep_when_e.html),” *schneier.com*, July 16, 2013.
105. Lena Ulbricht and Maximilian von Grafenstein: “[Big Data: Big Power Shifts?](http://policyreview.info/articles/analysis/big-data-big-power-shifts),” *Internet Policy Review*, volume 5, number 1, March 2016. [doi:10.14763/2016.1.406](http://dx.doi.org/10.14763/2016.1.406)
106. Ellen P. Goodman and Julia Powles: “[Facebook and Google: Most Powerful and Secretive Empires We've Ever Known](https://www.theguardian.com/technology/2016/sep/28/google-facebook-powerful-secretive-empire-transparency),” *theguardian.com*, September 28, 2016.
107. [Directive 95/46/EC on the protection of individuals with regard to the processing of personal data and on the free movement of such data](http://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX:31995L0046), Official Journal of the European Communities No. L 281/31, *eur-lex.europa.eu*, November 1995.
108. Brendan Van Alsenoy: “[Regulating Data Protection: The Allocation of Responsibility and Risk Among Actors Involved in Personal Data Processing](https://lirias.kuleuven.be/handle/123456789/545027),” Thesis, KU Leuven Centre for IT and IP Law, August 2016.
109. Michiel Rhoen: “[Beyond Consent: Improving Data Protection Through Consumer Protection Law](http://policyreview.info/articles/analysis/beyond-consent-improving-data-protection-through-consumer-protection-law),” *Internet Policy Review*, volume 5, number 1, March 2016. [doi:10.14763/2016.1.404](http://dx.doi.org/10.14763/2016.1.404)
110. Jessica Leber: “[Your Data Footprint Is Affecting Your Life in Ways You Can’t Even Imagine](https://www.fastcoexist.com/3057514/your-data-footprint-is-affecting-your-life-in-ways-you-cant-even-imagine),” *fastcoexist.com*, March 15, 2016.
111. Maciej Cegłowski: “[Haunted by Data](http://idlewords.com/talks/haunted_by_data.htm),” *idlewords.com*, October 2015.
112. Sam Thielman: “[You Are Not What You Read: Librarians Purge User Data to Protect Privacy](https://www.theguardian.com/us-news/2016/jan/13/us-library-records-purged-data-privacy),” *theguardian.com*, January 13, 2016.
113. Conor Friedersdorf: “[Edward Snowden’s Other Motive for Leaking](http://www.theatlantic.com/politics/archive/2014/05/edward-snowdens-other-motive-for-leaking/370068/),” *theatlantic.com*, May 13, 2014.
114. Phillip Rogaway: “[The Moral Character of Cryptographic Work](http://web.cs.ucdavis.edu/~rogaway/papers/moral-fn.pdf),” Cryptology ePrint 2015/1162, December 2015.