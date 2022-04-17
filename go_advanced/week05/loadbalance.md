# 负载均衡

## 目标

- 均衡的流量分发。
- 可靠的识别异常节点。
- scale-out，增加同质节点扩容。
- 减少错误，提高可用性（一般采用 N+2 个节点做高可用，比如 1 台机器就能够对外正常进行服务，我们就需要 3 台机器来同时对外服务，因为如果只有两台机器的话， 1 台机器宕机了，所有流量就会全部转发到第二台机器，可能也会马上出现问题。所以至少要 N+2 的机器做流量的冗余）。



### 问题

普通的轮询、加权轮询可能并不能做到很好的负载均衡：

每台机器之间的负载差异比较大的原因：

- 每个请求的处理成本不同（比如取 1 个用户和取 100 个用户的数据成本不一样）。
- 物理机环境的差异:
  - 服务器很难强同质性（可能有的机器老一些性能没有新机器强）。
  - 存在共享资源争用（内存缓存、带宽、IO等）。
- 性能因素:
  - FullGC。
  - JVM JIT（Just In Time，即时编译）。

参考 JSQ（最闲轮询）可能带来的问题：缺乏的是服务端全局视图，因此我们目标需要综合考虑：负载+可用性。

## 方案

参考了《[The power of two choices in randomized load balancing](https://ieeexplore.ieee.org/document/963420)》的思路，我们使用 p2c 算法，随机选取的两个节点进行打分，选择更优的节点:

- 选择 backend（服务器指标）：**CPU 使用率**，client（客户端指标）：**health**（可简化为请求成功率）、**inflight**（QPS）、**latency** 作为指标，使用一个简单的线性方程进行打分。

- 对新启动的节点使用常量惩罚值（penalty），以及使用探针方式最小化放量，进行预热。

- 打分比较低的节点，避免进入“永久黑名单”而无法恢复，使用统计衰减的方式，让节点指标逐渐恢复到初始状态(即默认值)。

  >  在 kratos 的实现里，会设定一个时间段 forcePick，如果某台机器在这段时间里一直没有被调用，那么下次选到它的时候肯定会被调用一次，放点流量进去更新它的分值。避免它永远都不会被调用而分值一直很低也得不到更新。）

- 当前发出去的请求超过了 predict lagtency，就会加惩罚。（比如说某个机器的 latency 都高于 50 分位值，那么就会给它打分低些）

  > 假设有100个请求，按照响应时间从小到大排列，位置为X的值，即为PX值。
  >
  > **P50:** 即中位数值。100个请求按照响应时间从小到大排列，位置为50的值，即为P50值。如果响应时间的P50值为200ms，代表我们有半数的用户响应耗时在200ms之内，有半数的用户响应耗时大于200ms。如果你觉得中位数值不够精确，那么可以使用P95和P99.9。
  >
  > **P95：**响应耗时从小到大排列，顺序处于95%位置的值即为P95值。
  >
  > 还是采用上面那个例子，100个请求按照响应时间从小到大排列，位置为95的值，即为P95值。 我们假设该值为200ms，那么对95%的用户的响应耗时在200ms之内，只有5%的用户的响应耗时大于200ms，据此，我们掌握了更精确的服务响应耗时信息。
  >
  > **P99.9：**许多大型的互联网公司会采用P99.9值，也就是99.9%用户耗时作为指标，意思就是1000个用户里面，999个用户的耗时上限，通过测量与优化该值，就可保证绝大多数用户的使用体验。 至于P99.99值，优化成本过高，而且服务响应由于网络波动、系统抖动等不能解决之情况，因此大多数时候都不考虑该指标。

- 指标计算结合 moving average，使用时间衰减，计算vt = v(t-1) * β + at * (1-β) ，β 为若干次幂的倒数即: Math.Exp((-span) / 600ms)。

- > 意思就是在某一段时间里会根据时间周期的数据来加权计算，比如在之前的时间段的数据权重比较低，而最近的数据权重高一些，这样打的分就更符合当前的情况。）

#### 采集 cpu 的方法 

对于 QPS 高的一些服务，由于 rpc response 里也可以带 context metadata，所以可以把机器的 cpu 使用率放在 metadata 里返回。

对于 QPS 低的一些服务，可以利用 health check 心跳机制将数据带回来。

### kratos 的 p2c 实现

p2c 算法在 kratos 的实现 [p2c.go](https://github.com/go-kratos/kratos/blob/main/selector/p2c/p2c.go)，这里贴出部分代码。课程里的 kratos 是 1.0x 版本，目前最新已经是 2.2.1 了，算法实现有点不太一样。最新的算法在计算负载（load）的实现去掉了 cpu 指标，只通过平均延迟和 QPS 来计算负载：`load = uint64(avgLag) * uint64(atomic.LoadInt64(&n.inflight))` ，然后再根据 success（成功率）和负载来计算机器的分值。

```go

// choose two distinct nodes.
func (s *Balancer) prePick(nodes []selector.WeightedNode) (nodeA selector.WeightedNode, nodeB selector.WeightedNode) {
	s.mu.Lock()
	a := s.r.Intn(len(nodes))
	b := s.r.Intn(len(nodes) - 1)
	s.mu.Unlock()
	if b >= a {
		b = b + 1
	}
	nodeA, nodeB = nodes[a], nodes[b]
	return
}

// Pick pick a node.
func (s *Balancer) Pick(ctx context.Context, nodes []selector.WeightedNode) (selector.WeightedNode, selector.DoneFunc, error) {
	if len(nodes) == 0 {
		return nil, nil, selector.ErrNoAvailable
	}
	if len(nodes) == 1 {
		done := nodes[0].Pick()
		return nodes[0], done, nil
	}

	var pc, upc selector.WeightedNode
	nodeA, nodeB := s.prePick(nodes)
	// meta.Weight is the weight set by the service publisher in discovery
	if nodeB.Weight() > nodeA.Weight() {
		pc, upc = nodeB, nodeA
	} else {
		pc, upc = nodeA, nodeB
	}

	// If the failed node has never been selected once during forceGap, it is forced to be selected once
	// Take advantage of forced opportunities to trigger updates of success rate and delay
	if upc.PickElapsed() > forcePick && atomic.CompareAndSwapInt64(&s.picked, 0, 1) {
		pc = upc
		atomic.StoreInt64(&s.picked, 0)
	}
	done := pc.Pick()
	return pc, done, nil
}
```

[node.go](https://github.com/go-kratos/kratos/blob/main/selector/node/ewma/node.go)：

```go
func (n *Node) load() (load uint64) {
	now := time.Now().UnixNano()
	avgLag := atomic.LoadInt64(&n.lag)
	lastPredictTs := atomic.LoadInt64(&n.predictTs)
	predictInterval := avgLag / 5
	if predictInterval < int64(time.Millisecond*5) {
		predictInterval = int64(time.Millisecond * 5)
	} else if predictInterval > int64(time.Millisecond*200) {
		predictInterval = int64(time.Millisecond * 200)
	}
	if now-lastPredictTs > predictInterval {
		if atomic.CompareAndSwapInt64(&n.predictTs, lastPredictTs, now) {
			var (
				total   int64
				count   int
				predict int64
			)
			n.lk.RLock()
			first := n.inflights.Front()
			for first != nil {
				lag := now - first.Value.(int64)
				if lag > avgLag {
					count++
					total += lag
				}
				first = first.Next()
			}
			if count > (n.inflights.Len()/2 + 1) {
				predict = total / int64(count)
			}
			n.lk.RUnlock()
			atomic.StoreInt64(&n.predict, predict)
		}
	}

	if avgLag == 0 {
		// penalty is the penalty value when there is no data when the node is just started.
		// The default value is 1e9 * 10
		load = penalty * uint64(atomic.LoadInt64(&n.inflight))
	} else {
		predict := atomic.LoadInt64(&n.predict)
		if predict > avgLag {
			avgLag = predict
		}
		load = uint64(avgLag) * uint64(atomic.LoadInt64(&n.inflight))
	}
	return
}

// Pick pick a node.
func (n *Node) Pick() selector.DoneFunc {
	now := time.Now().UnixNano()
	atomic.StoreInt64(&n.lastPick, now)
	atomic.AddInt64(&n.inflight, 1)
	atomic.AddInt64(&n.reqs, 1)
	n.lk.Lock()
	e := n.inflights.PushBack(now)
	n.lk.Unlock()
	return func(ctx context.Context, di selector.DoneInfo) {
		n.lk.Lock()
		n.inflights.Remove(e)
		n.lk.Unlock()
		atomic.AddInt64(&n.inflight, -1)

		now := time.Now().UnixNano()
		// get moving average ratio w
		stamp := atomic.SwapInt64(&n.stamp, now)
		td := now - stamp
		if td < 0 {
			td = 0
		}
		w := math.Exp(float64(-td) / float64(tau))

		start := e.Value.(int64)
		lag := now - start
		if lag < 0 {
			lag = 0
		}
		oldLag := atomic.LoadInt64(&n.lag)
		if oldLag == 0 {
			w = 0.0
		}
		lag = int64(float64(oldLag)*w + float64(lag)*(1.0-w))
		atomic.StoreInt64(&n.lag, lag)

		success := uint64(1000) // error value ,if error set 1
		if di.Err != nil {
			if n.errHandler != nil {
				if n.errHandler(di.Err) {
					success = 0
				}
			} else if errors.Is(context.DeadlineExceeded, di.Err) || errors.Is(context.Canceled, di.Err) ||
				errors.IsServiceUnavailable(di.Err) || errors.IsGatewayTimeout(di.Err) {
				success = 0
			}
		}
		oldSuc := atomic.LoadUint64(&n.success)
		success = uint64(float64(oldSuc)*w + float64(success)*(1.0-w))
		atomic.StoreUint64(&n.success, success)
	}
}

func (n *Node) health() uint64 {
	return atomic.LoadUint64(&n.success)
}
// Weight is node effective weight.
func (n *Node) Weight() (weight float64) {
	weight = float64(n.health()*uint64(time.Second)) / float64(n.load())
	return
}

func (n *Node) PickElapsed() time.Duration {
	return time.Duration(time.Now().UnixNano() - atomic.LoadInt64(&n.lastPick))
}
```



## 总结

- 变更管理:
  - 70％的问题是由变更引起的，恢复可用代码并不总是坏事。
- 避免过载:
  - 过载保护、流量调度等。
- 依赖管理:
  - 任何依赖都可能故障，做 chaos monkey testing，注入故障测试。
- 优雅降级:
  - 有损服务，避免核心链路依赖故障。
- 重试退避:
  - 退让算法，冻结时间，API retry detail 控制策略（通过 api response 下发重试策略）。
- 超时控制:
  - 进程内 + 服务间 超时控制。
- 极限压测 + 故障演练。
- 扩容 + 重启 + 消除有害流量。



## Reference

http://www.360doc.com/content/16/1124/21/31263000_609259745.shtml

http://www.infoq.com/cn/articles/basis-frameworkto-implement-micro-service/

http://www.infoq.com/cn/news/2017/04/linkerd-celebrates-one-year

https://medium.com/netflix-techblog/netflix-edge-load-balancing-695308b5548c

https://mp.weixin.qq.com/s?__biz=MzAwNjQwNzU2NQ==&mid=402841629&idx=1&sn=f598fec9b370b8a6f2062233b31122e0&mpshare=1&scene=23&srcid=0404qP0fH8zRiIiFzQBiuzuU#rd

https://mp.weixin.qq.com/s?__biz=MzIzMzk2NDQyMw==&mid=2247486641&idx=1&sn=1660fb41b0c5b8d8d6eacdfc1b26b6a6&source=41#wechat_redirect

https://blog.acolyer.org/2018/11/16/overload-control-for-scaling-wechat-microservices/

https://www.cs.columbia.edu/~ruigu/papers/socc18-final100.pdf

https://github.com/alibaba/Sentinel/wiki/系统负载保护

https://blog.csdn.net/okiwilldoit/article/details/81738782

http://alex-ii.github.io/notes/2019/02/13/predictive_load_balancing.html

https://blog.csdn.net/m0_38106113/article/details/81542863