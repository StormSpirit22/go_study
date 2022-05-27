# 第一章：可靠性、可伸缩性和可维护性

本章将从我们所要实现的基础目标开始：可靠、可伸缩、可维护的数据系统。

- 可靠性（Reliability）

系统在 **困境**（adversity，比如硬件故障、软件故障、人为错误）中仍可正常工作（正确完成功能，并能达到期望的性能水准）。

- 可伸缩性（Scalability）

有合理的办法应对系统的增长（数据量、流量、复杂性）。

- 可维护性（Maintainability）

许多不同的人（工程师、运维）在不同的生命周期，都能高效地在系统上工作（使系统保持现有行为，并适应新的应用场景）。

## 可靠性

我们可以通过故意引发故障来确保容错机制不断运行并接受考验，从而提高故障自然发生时系统能正确处理的信心。Netflix 公司的 [Chaos Monkey](https://github.com/Netflix/chaosmonkey) 就是这种方法的一个例子。

故障种类：硬件故障，软件错误，人为错误。

## 可伸缩性

### 描述负载

负载可以用一些称为 **负载参数（load parameters）** 的数字来描述。

以推特业务为例：

- 发布推文

  用户可以向其粉丝发布新消息（平均 4.6k 请求 / 秒，峰值超过 12k 请求 / 秒）。

- 主页时间线

  用户可以查阅他们关注的人发布的推文（300k 请求 / 秒）。

处理每秒 12,000 次写入（发推文的速率峰值）还是很简单的。然而推特的伸缩性挑战并不是主要来自推特量，而是来自 **扇出（fan-out）**（在事务处理系统中，我们使用它来描述为了服务一个传入请求而需要执行其他服务的请求数量）—— 每个用户关注了很多人，也被很多人关注。

大体上讲，这一对操作有两种实现方式。

1. 发布推文时，只需将新推文插入全局推文集合即可。当一个用户请求自己的主页时间线时，首先查找他关注的所有人，查询这些被关注用户发布的推文并按时间顺序合并。在如下图所示的关系型数据库中，可以编写这样的查询：

   ```sql
   SELECT tweets.*, users.*
     FROM tweets
     JOIN users   ON tweets.sender_id = users.id
     JOIN follows ON follows.followee_id = users.id
     WHERE follows.follower_id = current_user
   ```

   ![img](../../.go_study/assets/ddia/1-1.png)

   **图 1-2 推特主页时间线的关系型模式简单实现**

2. 为每个用户的主页时间线维护一个缓存，就像每个用户的推文收件箱（如下图）。 当一个用户发布推文时，查找所有关注该用户的人，并将新的推文插入到每个主页时间线缓存中。 因此读取主页时间线的请求开销很小，因为结果已经提前计算好了。

   ![img](../../.go_study/assets/ddia/1-2.png)

   **图 1-3 用于分发推特至关注者的数据流水线，2012 年 11 月的负载参数【16】**

推特轶事的最终版本：现在已经稳健地实现了方法 2，推特逐步转向了两种方法的混合。大多数用户发的推文会被扇出写入其粉丝主页时间线缓存中。但是少数拥有海量粉丝的用户（即名流）会被排除在外。当用户读取主页时间线时，分别地获取出该用户所关注的每位名流的推文，再与用户的主页时间线缓存合并，如方法 1 所示。这种混合方法能始终如一地提供良好性能。

### 描述性能

百分位点，例如第 95、99 和 99.9 百分位点（缩写为 p95，p99 和 p999）。它们意味着 95%、99% 或 99.9% 的请求响应时间要比该阈值快。响应时间的高百分位点（也称为 **尾部延迟**，即 **tail latencies**）非常重要，比如  99.9% 的请求都在 1 秒内，0.1% 的请求远远大于 1 秒，那么如何优化这个尾部延迟是非常需要考虑的。

百分位点通常用于 **服务级别目标（SLO, service level objectives）** 和 **服务级别协议（SLA, service level agreements）**，即定义服务预期性能和可用性的合同。 SLA 可能会声明，如果服务响应时间的中位数小于 200 毫秒，99% 的请求响应时间小于 1 秒，且 99.9% 的时间都要满足以上指标。

### 应对负载增加的方法

#### 实践中的百分位数

一次完整的服务包含很多请求，用户需要等待最慢的那个执行完成才行，这可能会拖累整个服务，称之为长尾效应。

可以设置一个 10 min 的滑动窗口，监控响应时间，滚动计算窗口中的中位数和各种百分位数，绘制性能图表。可以使用一些近似算法比如：正向衰减、t-digest 或 HdrHistogram 来计算百分位数的近似值。

垂直扩展（即升级到更强大的机器）和水平扩展（即将负载 分布到多个更小的机器）。

## 可维护性

软件系统的三个设计原则：

- 可运维性

方便运营团队来保持系统平稳运行。 

- 简单性 

简化系统复杂性，使新工程师能够轻松理解系统。注意这与用户界面的简单性并不一样。 

- 可演化性 

后续工程师能够轻松地对系统进行改进，井根据需求变化将其适配到非典型场景，也称为可延伸性、易修改性或可塑性。



## 参考文献

1. Michael Stonebraker and Uğur Çetintemel: “['One Size Fits All': An Idea Whose Time Has Come and Gone](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.68.9136&rep=rep1&type=pdf),” at *21st International Conference on Data Engineering* (ICDE), April 2005.
2. Walter L. Heimerdinger and Charles B. Weinstock: “[A Conceptual Framework for System Fault Tolerance](http://www.sei.cmu.edu/reports/92tr033.pdf),” Technical Report CMU/SEI-92-TR-033, Software Engineering Institute, Carnegie Mellon University, October 1992.
3. Ding Yuan, Yu Luo, Xin Zhuang, et al.: “[Simple Testing Can Prevent Most Critical Failures: An Analysis of Production Failures in Distributed Data-Intensive Systems](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-yuan.pdf),” at *11th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2014.
4. Yury Izrailevsky and Ariel Tseitlin: “[The Netflix Simian Army](http://techblog.netflix.com/2011/07/netflix-simian-army.html),” *techblog.netflix.com*, July 19, 2011.
5. Daniel Ford, François Labelle, Florentina I. Popovici, et al.: “[Availability in Globally Distributed Storage Systems](http://research.google.com/pubs/archive/36737.pdf),” at *9th USENIX Symposium on Operating Systems Design and Implementation* (OSDI), October 2010.
6. Brian Beach: “[Hard Drive Reliability Update – Sep 2014](https://www.backblaze.com/blog/hard-drive-reliability-update-september-2014/),” *backblaze.com*, September 23, 2014.
7. Laurie Voss: “[AWS: The Good, the Bad and the Ugly](https://web.archive.org/web/20160429075023/http://blog.awe.sm/2012/12/18/aws-the-good-the-bad-and-the-ugly/),” *blog.awe.sm*, December 18, 2012.
8. Haryadi S. Gunawi, Mingzhe Hao, Tanakorn Leesatapornwongsa, et al.: “[What Bugs Live in the Cloud?](http://ucare.cs.uchicago.edu/pdf/socc14-cbs.pdf),” at *5th ACM Symposium on Cloud Computing* (SoCC), November 2014. [doi:10.1145/2670979.2670986](http://dx.doi.org/10.1145/2670979.2670986)
9. Nelson Minar: “[Leap Second Crashes Half the Internet](http://www.somebits.com/weblog/tech/bad/leap-second-2012.html),” *somebits.com*, July 3, 2012.
10. Amazon Web Services: “[Summary of the Amazon EC2 and Amazon RDS Service Disruption in the US East Region](http://aws.amazon.com/message/65648/),” *aws.amazon.com*, April 29, 2011.
11. Richard I. Cook: “[How Complex Systems Fail](http://web.mit.edu/2.75/resources/random/How Complex Systems Fail.pdf),” Cognitive Technologies Laboratory, April 2000.
12. Jay Kreps: “[Getting Real About Distributed System Reliability](http://blog.empathybox.com/post/19574936361/getting-real-about-distributed-system-reliability),” *blog.empathybox.com*, March 19, 2012.
13. David Oppenheimer, Archana Ganapathi, and David A. Patterson: “[Why Do Internet Services Fail, and What Can Be Done About It?](http://static.usenix.org/legacy/events/usits03/tech/full_papers/oppenheimer/oppenheimer.pdf),” at *4th USENIX Symposium on Internet Technologies and Systems* (USITS), March 2003.
14. Nathan Marz: “[Principles of Software Engineering, Part 1](http://nathanmarz.com/blog/principles-of-software-engineering-part-1.html),” *nathanmarz.com*, April 2, 2013.
15. Michael Jurewitz:“[The Human Impact of Bugs](http://jury.me/blog/2013/3/14/the-human-impact-of-bugs),” *jury.me*, March 15, 2013.
16. Raffi Krikorian: “[Timelines at Scale](http://www.infoq.com/presentations/Twitter-Timeline-Scalability),” at *QCon San Francisco*, November 2012.
17. Martin Fowler: *Patterns of Enterprise Application Architecture*. Addison Wesley, 2002. ISBN: 978-0-321-12742-6
18. Kelly Sommers: “[After all that run around, what caused 500ms disk latency even when we replaced physical server?](https://twitter.com/kellabyte/status/532930540777635840)” *twitter.com*, November 13, 2014.
19. Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, et al.: “[Dynamo: Amazon's Highly Available Key-Value Store](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf),” at *21st ACM Symposium on Operating Systems Principles* (SOSP), October 2007.
20. Greg Linden: “[Make Data Useful](http://glinden.blogspot.co.uk/2006/12/slides-from-my-talk-at-stanford.html),” slides from presentation at Stanford University Data Mining class (CS345), December 2006.
21. Tammy Everts: “[The Real Cost of Slow Time vs Downtime](http://www.webperformancetoday.com/2014/11/12/real-cost-slow-time-vs-downtime-slides/),” *webperformancetoday.com*, November 12, 2014.
22. Jake Brutlag:“[Speed Matters for Google Web Search](http://googleresearch.blogspot.co.uk/2009/06/speed-matters.html),” *googleresearch.blogspot.co.uk*, June 22, 2009.
23. Tyler Treat: “[Everything You Know About Latency Is Wrong](http://bravenewgeek.com/everything-you-know-about-latency-is-wrong/),” *bravenewgeek.com*, December 12, 2015.
24. Jeffrey Dean and Luiz André Barroso: “[The Tail at Scale](http://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext),” *Communications of the ACM*, volume 56, number 2, pages 74–80, February 2013. [doi:10.1145/2408776.2408794](http://dx.doi.org/10.1145/2408776.2408794)
25. Graham Cormode, Vladislav Shkapenyuk, Divesh Srivastava, and Bojian Xu: “[Forward Decay: A Practical Time Decay Model for Streaming Systems](http://dimacs.rutgers.edu/~graham/pubs/papers/fwddecay.pdf),” at *25th IEEE International Conference on Data Engineering* (ICDE), March 2009.
26. Ted Dunning and Otmar Ertl: “[Computing Extremely Accurate Quantiles Using t-Digests](https://github.com/tdunning/t-digest),” *github.com*, March 2014.
27. Gil Tene: “[HdrHistogram](http://www.hdrhistogram.org/),” *hdrhistogram.org*.
28. Baron Schwartz: “[Why Percentiles Don’t Work the Way You Think](https://www.vividcortex.com/blog/why-percentiles-dont-work-the-way-you-think),” *vividcortex.com*, December 7, 2015.
29. James Hamilton: “[On Designing and Deploying Internet-Scale Services](https://www.usenix.org/legacy/events/lisa07/tech/full_papers/hamilton/hamilton.pdf),” at *21st Large Installation System Administration Conference* (LISA), November 2007.
30. Brian Foote and Joseph Yoder: “[Big Ball of Mud](http://www.laputan.org/pub/foote/mud.pdf),” at *4th Conference on Pattern Languages of Programs* (PLoP), September 1997.
31. Frederick P Brooks: “No Silver Bullet – Essence and Accident in Software Engineering,” in *The Mythical Man-Month*, Anniversary edition, Addison-Wesley, 1995. ISBN: 978-0-201-83595-3
32. Ben Moseley and Peter Marks: “[Out of the Tar Pit](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.93.8928),” at *BCS Software Practice Advancement* (SPA), 2006.
33. Rich Hickey: “[Simple Made Easy](http://www.infoq.com/presentations/Simple-Made-Easy),” at *Strange Loop*, September 2011.
34. Hongyu Pei Breivold, Ivica Crnkovic, and Peter J. Eriksson: “[Analyzing Software Evolvability](http://www.mrtc.mdh.se/publications/1478.pdf),” at *32nd Annual IEEE International Computer Software and Applications Conference* (COMPSAC), July 2008. [doi:10.1109/COMPSAC.2008.50](http://dx.doi.org/10.1109/COMPSAC.2008.50)