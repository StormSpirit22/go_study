# Channel & Context

## Channel

channels 是一种类型安全的消息队列，充当两个 goroutine 之间的管道，将通过它同步的进行任意资源的交换。chan 控制 goroutines 交互的能力从而创建了 Go 同步机制。当创建的 chan 没有容量时，称为无缓冲通道。反过来，使用容量创建的 chan 称为缓冲通道。

### 无缓冲 channel

如下图所示，无缓冲的 channel 会阻塞直到数据接收完成，常用于两个 goroutine 互相等待同步。

![image-20220405212618378](/Users/tianyou/Documents/Github/ty/go_study/.go_study/assets/go_advanced/chan-1.png)

### 有缓冲 channel

有缓冲的 channel 如果在缓冲区未满的情况下发送是不阻塞的，在缓冲区不为空时，接收是不阻塞的。

![image-20220405212629202](/Users/tianyou/Documents/Github/ty/go_study/.go_study/assets/go_advanced/chan-2.png)

### Go 并发模式

- Timing out

```go
select {
case <-ch:
    // a read from ch has occurred
case <- time.After(1 * time.Second):
    // the read from ch has timed out
}
```

- Moving on

多个 goroutine 查询结果只接受最快的那个返回。

```go
func Query(conns []Conn, query string) Result {
  	// 这里改成有缓冲的 buffer 是为了避免查询结果太快，而最后 <- ch 还没准备好接收数据，所有的 goroutine 的 select 第一个 case 都会阻塞而都选择 default 执行了。有 buffer 的 channel 就可以保证至少写入一个数据，等待最后接收。
    ch := make(chan Result, 1)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <-ch
}
```

- Pipeline

由 channel 连接起来的一些阶段组成一个 pipeline。第一个阶段可以称之为生产者，最后一个阶段称之为消费者。

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Set up the pipeline and consume the output.
    for n := range sq(sq(gen(2, 3))) {
        fmt.Println(n) // 16 then 81
    }
}
```

- Fan-out, Fan-in

> Multiple functions can read from the same channel until that channel is closed; this is called *fan-out*. This provides a way to distribute work amongst a group of workers to parallelize CPU use and I/O.
>
> A function can read from multiple inputs and proceed until all are closed by multiplexing the input channels onto a single channel that’s closed when all the inputs are closed. This is called *fan-in*.

多个方法从同一个 channel 里读取的模式称之为 fan-out，使得任务能够被分配给多个 worker 并行去处理。

从多个输入 channels 读取数据并处理最后发送到一个 channel 的模式叫做 fan-in。

```go
func main() {
    in := gen(2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(in)
    c2 := sq(in)

    // Consume the merged output from c1 and c2.
    for n := range merge(c1, c2) {
        fmt.Println(n) // 4 then 9, or 9 then 4
    }
}

func merge(cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c is closed, then calls wg.Done.
    output := func(c <-chan int) {
        for n := range c {
            out <- n
        }
        wg.Done()
    }
    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    // Start a goroutine to close out once all the output goroutines are
    // done.  This must start after the wg.Add call.
    go func() {
        wg.Wait()
        // close channel 必须要在所有发送者发送完了之后才能 close。
        close(out)
    }()
    return out
}
```

- Cancellation
  - Close 先于 Receive 发生（类似 Buffered）。
  - 不需要传递数据，或者传递 nil。
  - 非常适合取消和超时控制。

pipeline 构建的一些原则：

> - stages close their outbound channels when all the send operations are done.
> - stages keep receiving values from inbound channels until those channels are closed or the senders are unblocked.

例子：通过 close channel 来通知其他 goroutine 取消。

```go
func main() {
    // Set up a done channel that's shared by the whole pipeline,
    // and close that channel when this pipeline exits, as a signal
    // for all the goroutines we started to exit.
    done := make(chan struct{})
    defer close(done)          

    in := gen(done, 2, 3)

    // Distribute the sq work across two goroutines that both read from in.
    c1 := sq(done, in)
    c2 := sq(done, in)

    // Consume the first value from output.
    out := merge(done, c1, c2)
    fmt.Println(<-out) // 4 or 9

    // done will be closed by the deferred call.      
}

func merge(done <-chan struct{}, cs ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    out := make(chan int)

    // Start an output goroutine for each input channel in cs.  output
    // copies values from c to out until c or done is closed, then calls
    // wg.Done.
    output := func(c <-chan int) {
        defer wg.Done()
        for n := range c {
            select {
            case out <- n:
            case <-done:
                return
            }
        }
    }
    // ... the rest is unchanged ...
  
  
func sq(done <-chan struct{}, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-done:
                return
            }
        }
    }()
    return out
}
```

- Context

参考资料：

https://blog.golang.org/concurrency-timeouts

https://blog.golang.org/pipelines

https://talks.golang.org/2013/advconc.slide#1

https://github.com/go-kratos/kratos/tree/master/pkg/sync



以下内容来自[Go并发编程(十) 深入理解 Channel](https://lailin.xyz/post/go-training-week3-channel.html)

### 源码分析

#### 数据结构

```go
type hchan struct {
	qcount   uint           // 队列中元素总数量
	dataqsiz uint           // 循环队列的长度
	buf      unsafe.Pointer // 指向长度为 dataqsiz 的底层数组，只有在有缓冲时这个才有意义
	elemsize uint16         // 能够发送和接受的元素大小
	closed   uint32         // 是否关闭
	elemtype *_type // 元素的类型
	sendx    uint   // 当前已发送的元素在队列当中的索引位置
	recvx    uint   // 当前已接收的元素在队列当中的索引位置
	recvq    waitq  // 接收 Goroutine 链表
	sendq    waitq  // 发送 Goroutine 链表

	lock mutex // 互斥锁
}

// waitq 是一个双向链表，里面保存了 goroutine
type waitq struct {
	first *sudog
	last  *sudog
}
```

如下图所示，channel 底层其实是一个循环队列：

![image.png](/Users/tianyou/Documents/Github/ty/go_study/.go_study/assets/go_advanced/chan-3.png)

#### 创建

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 先做一些检查
    // 元素大小不能大于等于 64k
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
    // 判断当前的 hchanSize 是否是 maxAlign 整数倍，并且元素的对齐大小不能大于最大对齐的大小
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

    // 这里计算内存是否超过限制
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0: // 如果是无缓冲通道
		c = (*hchan)(mallocgc(hchanSize, nil, true)) // 为 hchan 分配内存
		c.buf = c.raceaddr() // 这个是 for data race 检测的
	case elem.ptrdata == 0: // 元素不包含指针
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true)) // 为 hchan 和底层数组分配一段连续的内存地址
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default: // 如果元素包含指针，分别为 hchan 和 底层数组分配内存地址
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

    // 初始化一些值
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	return c
}
```

##### 小结

- 创建时会做一些检查
  - 元素大小不能超过 64K。
  - 元素的对齐大小不能超过 maxAlign 也就是 8 字节，也是 64 位 CPU 下的 cacheline 的大小。
  - 计算出来的内存是否超过限制。
- 创建时的策略
  - 如果是无缓冲的 channel，会直接给 hchan 分配内存。
  - 如果是有缓冲的 channel，并且元素不包含指针，那么会为 hchan 和底层数组分配一段连续的地址。
  - 如果是有缓冲的 channel，并且元素包含指针，那么会为 hchan 和底层数组分别分配地址。

#### 发送数据

```go
// 代码中删除了调试相关的代码
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 如果是一个 nil 值的 channel
    // 如果是非阻塞的话就直接返回
    // 如果不是，那么则调用 gopark 休眠当前 goroutine 并且抛出 panic 错误
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

    // fast path 如果当前是非阻塞的
    // 并且通道尚未关闭
    // 并且缓冲区已满时，直接返回
	if !block && c.closed == 0 && full(c) {
		return false
	}

    // 加锁
	lock(&c.lock)

    // 如果通道已经关闭了，直接 panic，不允许向一个已经关闭的 channel 写入数据
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

    // 如果当前存在等待接收数据的 goroutine 直接取出第一个，将数据传递给第一个等待的 goroutine
	if sg := c.recvq.dequeue(); sg != nil {
		// send 用于发送数据，我们后面再看
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 如果当前 channel 包含缓冲区，并且缓冲区没有满
	if c.qcount < c.dataqsiz {
		// 计算数组中下一个可以存放数据的地址
		qp := chanbuf(c, c.sendx)

        // 将当前的数据放到缓冲区中
		typedmemmove(c.elemtype, qp, ep)

        // 索引加一
        c.sendx++

        // 由于是循环队列，如果索引地址等于数组长度，就需要将索引移动到 0
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}

        // 当前缓存数据量加一
		c.qcount++
		unlock(&c.lock)
		return true
	}

    // 如果是非阻塞的就直接返回了，因为非阻塞发送的情况已经走完了，下面是阻塞发送的逻辑
	if !block {
		unlock(&c.lock)
		return false
	}

	// 获取发送数据的 goroutine
	gp := getg()
    // 获取 sudog 结构体，并且设置相关信息，包括当前的 channel，是否是 select 等
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil

    // 将 sudog 结构加入到发送的队列中
	c.sendq.enqueue(mysg)

    // 挂起当前 goroutine 等待接收 channel数据
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

    // 保证当前数据处于活跃状态避免被回收
	KeepAlive(ep)

	// 发送者 goroutine 被唤醒，检查当前 sg 的状态
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}

    // 取消 channel 绑定
	mysg.c = nil
    // 释放 sudog
	releaseSudog(mysg)
	return true
}
```

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    // 如果 sudog 上存在数据元素，就调用 sendDirect 直接把数据拷贝到接收变量的地址上
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}

    // 调用 goready 将接受者的 Goroutine 标记为可运行状态，并把它放到发送方的所在处理器的 runnext 等待执行，下次调度时就会执行到它。
    // 注意这里不是立即执行
	goready(gp, skip+1)
}
```

##### 小结

向 channel 中发送数据时大概分为两大块，检查和数据发送，而数据发送又分为三种情况：

- 如果 channel 的 `recvq` 存在阻塞等待的接收数据的 goroutine 那么将会直接将数据发送给第一个等待的 goroutine。
  - 这里会直接将数据拷贝到 `x <-ch` 接收者的变量 `x` 上。
  - 然后将接收者的 Goroutine 修改为可运行状态，并把它放到发送方所在处理器的 runnext 上等待下一次调度时执行。
- 如果 channel 是有缓冲的，并且缓冲区没有满，这个时候就会把数据放到缓冲区中。
- 如果 channel 的缓冲区满了，这个时候就会走阻塞发送的流程，获取到 sudog 之后将当前 Goroutine 挂起等待唤醒，唤醒后将相关的数据解绑，回收掉 sudog。

#### 接收数据

```go
// selected 用于 select{} 语法中是否会选中该分支
// received 表示当前是否真正的接收到数据，用来判断 channel 是否 closed 掉了
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 和发送数据类似，先判断是否为nil，如果是 nil 并且阻塞接收就会 panic
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: 检查非阻塞的操作
    // empty 主要是有两种情况返回 true:
    // 1. 无缓冲channel，并且没有阻塞住发送者
    // 2. 有缓冲 channel，但是缓冲区没有数据
	if !block && empty(c) {
		// 这里判断通道是否关闭，如果是未关闭的通道说明当前还没准备好数据，直接返回
		if atomic.Load(&c.closed) == 0 {
			return
		}
		// 如果通道已经关闭了，再检查一下通道还有没有数据，如果已经没数据了，我们清理到 ep 指针中的数据并且返回
		if empty(c) {
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	// 上锁
	lock(&c.lock)

    // 和上面类似，如果通道已经关闭了，并且已经没数据了，我们清理到 ep 指针中的数据并且返回
	if c.closed != 0 && c.qcount == 0 {
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

    // 和发送类似，接收数据时也是先看一下有没有正在阻塞的等待发送数据的 Goroutine
    // 如果有的话 直接调用 recv 方法从发送者或者是缓冲区中接收数据，recv 方法后面会讲到
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

    // 如果 channel 的缓冲区还有数据
	if c.qcount > 0 {
		// 获取当前 channel 接收的地址
		qp := chanbuf(c, c.recvx)

        // 如果传入的指针不是 nil 直接把数据复制到对应的变量上
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
        // 清除队列中的数据，设置接受者索引并且返回
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

    // 和发送一样剩下的就是阻塞操作了，如果是非阻塞的情况，直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 阻塞接受，和发送类似，拿到当前 Goroutine 和 sudog 并且做一些数据填充
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil

    // 把 sudog 放入到接收者队列当中
	c.recvq.enqueue(mysg)
    // 然后休眠当前 Goroutine 等待唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// Goroutine 被唤醒，接收完数据，做一些数据清理的操作，释放掉 sudog 然后返回
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

recv

```go
// selected 用于 select{} 语法中是否会选中该分支
// received 表示当前是否真正的接收到数据，用来判断 channel 是否 closed 掉了
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 和发送数据类似，先判断是否为nil，如果是 nil 并且阻塞接收就会 panic
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: 检查非阻塞的操作
    // empty 主要是有两种情况返回 true:
    // 1. 无缓冲channel，并且没有阻塞住发送者
    // 2. 有缓冲 channel，但是缓冲区没有数据
	if !block && empty(c) {
		// 这里判断通道是否关闭，如果是未关闭的通道说明当前还没准备好数据，直接返回
		if atomic.Load(&c.closed) == 0 {
			return
		}
		// 如果通道已经关闭了，再检查一下通道还有没有数据，如果已经没数据了，我们清理到 ep 指针中的数据并且返回
		if empty(c) {
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	// 上锁
	lock(&c.lock)

    // 和上面类似，如果通道已经关闭了，并且已经没数据了，我们清理到 ep 指针中的数据并且返回
	if c.closed != 0 && c.qcount == 0 {
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

    // 和发送类似，接收数据时也是先看一下有没有正在阻塞的等待发送数据的 Goroutine
    // 如果有的话 直接调用 recv 方法从发送者或者是缓冲区中接收数据，recv 方法后面会讲到
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

    // 如果 channel 的缓冲区还有数据
	if c.qcount > 0 {
		// 获取当前 channel 接收的地址
		qp := chanbuf(c, c.recvx)

        // 如果传入的指针不是 nil 直接把数据复制到对应的变量上
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
        // 清除队列中的数据，设置接受者索引并且返回
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

    // 和发送一样剩下的就是阻塞操作了，如果是非阻塞的情况，直接返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 阻塞接受，和发送类似，拿到当前 Goroutine 和 sudog 并且做一些数据填充
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil

    // 把 sudog 放入到接收者队列当中
	c.recvq.enqueue(mysg)
    // 然后休眠当前 Goroutine 等待唤醒
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// Goroutine 被唤醒，接收完数据，做一些数据清理的操作，释放掉 sudog 然后返回
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, !closed
}
```

##### 小结

数据接收和发送其实大同小异，也是分为检查和数据接收，数据接收又分三种情况：

- 直接获取数据，如果当前有阻塞的发送者就走这条路。
  - 如果是无缓冲 channel，直接从发送者那里把数据拷贝给接收变量。
  - 如果是有缓冲 channel，并且 channel 已经满了，就先从 channel 的底层数组拷贝数据，再把阻塞的发送者 Goroutine 的数据拷贝到 channel 的循环队列中。
- 从 channel 的缓冲中获取数据，有缓冲 channel 并且缓存队列有数据时走这条路。
  - 直接从缓存队列中复制数据给接收变量。
- 阻塞接收，剩余情况走这里：
  - 和发送类似，先获取当前 Goroutine 信息，构造 sudog 加入到 channel 的 recvq 上。
  - 然后休眠当前 Goroutine 等待唤醒。
  - 唤醒后做一些清理工作，释放 sudog 返回。

#### 关闭 channel

```go
func closechan(c *hchan) {
    // 关闭 nil 的 channel 会导致 panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

    // 加锁
	lock(&c.lock)

    // 关闭已关闭的 channel 会导致 panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	// 设置 channel 状态
	c.closed = 1

	var glist gList

	// 释放所有的接收者 Goroutine
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil

		glist.push(gp)
	}

	// 释放所有的发送者channel，会 panic 因为不允许向已关闭的 channel 发送数据
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// 将所有的 Goroutine 设置为可运行状态
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

##### 小结

- 关闭一个 nil 的 channel 和已关闭了的 channel 都会导致 panic
- 关闭 channel 后会释放所有因为 channel 而阻塞的 Goroutine



## Context

#### 使用准则

context 包一开始就告诉了我们应该怎么用，不应该怎么用，这是应该被共同遵守的约定。

- 对 server 应用而言，**每个传入的请求都应该创建一个 context，它的作用域是请求级别的**。
- 通过 `WithCancel` , `WithDeadline` , `WithTimeout` 创建的 Context 会同时返回一个 cancel 方法，这个方法必须要被执行，不然会导致 context 泄漏，这个可以通过执行 `go vet` 命令进行检查。
- 应该将 `context.Context` 作为函数的第一个参数进行传递，参数命名一般为 `ctx` 不应该将 Context 作为字段放在结构体中。
- 不要给 context 传递 nil，如果你不知道应该传什么的时候就传递 `context.TODO()`。
- 不要将函数的可选参数放在 context 当中，context 中一般只放一些全局通用的 metadata 数据，例如 tracing id 等等。
- context 是并发安全的可以在多个 goroutine 中并发调用。

使用 context 的一个很好的心智模型是它应该在程序中流动，应该贯穿你的代码。这通常意味着不要将其存储在结构体之中。它从一个函数传递到另一个函数，并根据需要进行扩展。理想情况下，每个请求都会创建一个 context 对象，**并在请求结束时过期**。



#### 函数签名

```go
// 创建一个带有新的 Done channel 的 context，并且返回一个取消的方法
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
// 创建一个具有截止时间的 context
// 截止时间是 d 和 parent(如果有截止时间的话) 的截止时间中更早的那一个
// 当 parent 执行完毕，或 cancel 被调用 或者 截止时间到了的时候，这个 context done 掉
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
// 其实就是调用的 WithDeadline
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
type CancelFunc
type Context
	// 一般用于创建 root context，这个 context 永远也不会被取消，或者是 done
    func Background() Context
	// 底层和 Background 一致，但是含义不同，当不清楚用什么的时候或者是还没准备好的时候可以用它
    func TODO() Context
	// 为 context 附加值
	// key 应该具有可比性
    func WithValue(parent Context, key, val interface{}) Context
```



##### Context.WithValue

`WithValue` 实际上就是在 Context 树形结构中，增加一个节点，每次需要设置一个值，都需要调用 `WithValue` 来生成一个新的节点。

**Context 是 immutable 的**。如果要修改某个值，可以将老的值 deep copy 出来一份，修改然后再通过 `WithValue` 创建一个新的节点放到 Context 里。

```go
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```



基于 valueCtx 实现：

```go
type valueCtx struct {
    Context
    key, val interface{}
}
```

查找值：

```go
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

内部在查找 key 时候，使用递归方式不断从当前节点、父节点寻找匹配的 key，直到 root context（Backgrond 和 TODO Value 函数会返回 nil）。

![img](https://3531624266-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FhUllJBLnT2dJaXaijzYO%2Fuploads%2Fgit-blob-d8fc8196f1311fb65745bd0a850bf97a31b1e8c9%2Fstandard-4.png?alt=media)



###### 为什么要选择链表而不是 map 的形式来存储 kv 数据？

因为 context 要保证并发读写安全，map 并发读写数据会 panic。而使用链表添加节点的形式，一旦添加了一个节点就不允许修改这个节点，每次都是新增一个节点的方式来处理，这样就没有并发读写的问题了。

##### Context 是 immutable 的

context.WithValue 方法允许上下文携带请求范围的数据。这些数据必须是**安全**的，以便多个 goroutine 同时使用。这里的数据，更多是面向请求的元数据，不应该作为函数的可选参数来使用（比如 context 里面挂了一个sql.Tx 对象，传递到 data 层使用），因为元数据相对函数参数更加是隐含的，面向请求的。而参数是更加显示的。

- 同一个 context 对象可以传递给在不同 goroutine 中运行的函数；
- 上下文对于多个 goroutine 同时使用是安全的。
- 对于值类型最容易犯错的地方，在于 context value 应该是 immutable 的，每次重新赋值应该是新的 context 节点。

可以参考 [grpc metadata](https://pkg.go.dev/google.golang.org/grpc/metadata) 里面 api 的实现方式，比如：

```go
type rawMD struct {
	md    MD
	added [][]string
}
type mdOutgoingKey struct{}

func AppendToOutgoingContext(ctx context.Context, kv ...string) context.Context {
	if len(kv)%2 == 1 {
		panic(fmt.Sprintf("metadata: AppendToOutgoingContext got an odd number of input pairs for metadata: %d", len(kv)))
	}
	md, _ := ctx.Value(mdOutgoingKey{}).(rawMD)
	added := make([][]string, len(md.added)+1)
  // 将 md.added 的 [][]string 复制给一个新的变量 added
	copy(added, md.added)
	added[len(added)-1] = make([]string, len(kv))
	copy(added[len(added)-1], kv)
 	// 再将 append 之后的 added 数组通过 WithValue 构造一个新的节点放到 context 里
	return context.WithValue(ctx, mdOutgoingKey{}, rawMD{md: md.md, added: added})
}
```

实际业务中也可以将 context value 设置成一个 map，每次要在 map 里添加值时也是通过上面的方式去添加，即把值 copy 出来给一个新的变量然后通过 `context.WithValue` 来构造一个新的节点。这也是一种 COW 的思想。具体思路：

从 ctx1 中获取 map1（可以理解为 v1 版本的 map 数据）。构建一个新的 map 对象 map2，复制所有 map1 数据，同时追加新的数据 “k2”: “v2” 键值对，使用 context.WithValue 创建新的 ctx2，ctx2 会传递到其他的 goroutine 中。这样各自读取的副本都是自己的数据，写行为追加的数据，在 ctx2 中也能完整读取到，同时也不会污染 ctx1 中的数据。

![image-20220406120421369](/Users/tianyou/Library/Application Support/typora-user-images/image-20220406120421369.png)

以下内容来自 [Go并发编程(九) 深入理解 Context](https://lailin.xyz/post/go-training-week3-context.html#%E5%A6%82%E4%BD%95%E5%8F%96%E6%B6%88-context-WithCancel)

#### context.Context 接口

```go
type Context interface {
    // 返回当前 context 的结束时间，如果 ok = false 说明当前 context 没有设置结束时间
	Deadline() (deadline time.Time, ok bool)
    // 返回一个 channel，用于判断 context 是否结束，多次调用同一个 context done 方法会返回相同的 channel
	Done() <-chan struct{}
    // 当 context 结束时才会返回错误，有两种情况
    // context 被主动调用 cancel 方法取消：Canceled
    // context 超时取消: DeadlineExceeded
	Err() error
    // 用于返回 context 中保存的值, 如何查找，这个后面会讲到
	Value(key interface{}) interface{}
}
```

#### 默认上下文: context.Backgroud

**Backgroud()，**在前面有讲到， 一般用于创建 root context，这个 context 永远也不会被取消，或超时
**TODO()， **底层和 Background 一致，但是含义不同，当不清楚用什么的时候或者是还没准备好的时候可以用它

```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

查看源码我们可以发现，background 和 todo 都是实例化了一个 emptyCtx

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

emptyCtx 就如同他的名字一样，全都返回空值

#### 如何取消 context : WithCancel

**WithCancel(),** 方法会创建一个可以取消的 context

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
    // 包装出新的 cancelContext
	c := newCancelCtx(parent)
    // 构建父子上下文的联系，确保当父 Context 取消的时候，子 Context 也会被取消
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

不止 WithCancel 方法，其他的 WithXXX 方法也不允许传入一个 nil 值的父 context
`newCancelCtx` 只是一个简单的包装就不展开了， `propagateCancel` 比较有意思，我们一起来看看

```go
func propagateCancel(parent Context, child canceler) {
	// 首先判断 parent 能不能被取消
    done := parent.Done()
	if done == nil {
		return // parent is never canceled
	}

    // 如果可以，看一下 parent 是不是已经被取消了，已经被取消的情况下直接取消 子 context
	select {
	case <-done:
		// parent is already canceled
		child.cancel(false, parent.Err())
		return
	default:
	}

    // 这里是向上查找可以被取消的 parent context
	if p, ok := parentCancelCtx(parent); ok {
        // 如果找到了并且没有被取消的话就把这个子 context 挂载到这个 parent context 上
        // 这样只要 parent context 取消了子 context 也会跟着被取消
		p.mu.Lock()
		if p.err != nil {
			// parent has already been canceled
			child.cancel(false, p.err)
		} else {
			if p.children == nil {
				p.children = make(map[canceler]struct{})
			}
			p.children[child] = struct{}{}
		}
		p.mu.Unlock()
	} else {
        // 如果没有找到的话就会启动一个 goroutine 去监听 parent context 的取消 channel
        // 收到取消信号之后再去调用 子 context 的 cancel 方法
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

接下来我们就看看 cancelCtx 长啥样

```go
type cancelCtx struct {
	Context // 这里保存的是父 Context

	mu       sync.Mutex            // 互斥锁
	done     chan struct{}         // 关闭信号
	children map[canceler]struct{} // 保存所有的子 context，当取消的时候会被设置为 nil
	err      error
}
```

在 Done 方法这里采用了 懒汉式加载的方式，第一次调用的时候才会去创建这个 channel

```go
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```

Value 方法很有意思，这里相当于是内部 `cancelCtxKey` 这个变量的地址作为了一个特殊的 key，当查询这个 key 的时候就会返回当前 context 如果不是这个 key 就会向上递归的去调用 parent context 的 Value 方法查找有没有对应的值

```go
func (c *cancelCtx) Value(key interface{}) interface{} {
	if key == &cancelCtxKey {
		return c
	}
	return c.Context.Value(key)
}
```

在前面讲到构建父子上下文之间的关系的时候，有一个去查找可以被取消的父 context 的方法 `parentCancelCtx` 就用到了这个特殊 value

```go
func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    // 这里先判断传入的 parent 是不是永远不可取消的，如果是就直接返回了
	done := parent.Done()
	if done == closedchan || done == nil {
		return nil, false
	}

    // 这里利用了 context.Value 不断向上查询值的特点，只要出现第一个可以取消的 context 的时候就会返回
    // 如果没有的话，这时候 ok 就会等于 false
	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
	if !ok {
		return nil, false
	}
    // 这里去判断返回的 parent 的 channel 和传入的 parent 是不是同一个，是的话就返回这个 parent
	p.mu.Lock()
	ok = p.done == done
	p.mu.Unlock()
	if !ok {
		return nil, false
	}
	return p, true
}
```

接下来我们来看最重要的这个 cancel 方法，cancel 接收两个参数，removeFromParent 用于确认是不是把自己从 parent context 中移除，err 是 ctx.Err() 最后返回的错误信息

```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
    // 由于 cancel context 的 done 是懒加载的，所以有可能存在还没有初始化的情况
	if c.done == nil {
		c.done = closedchan
	} else {
		close(c.done)
	}
    // 循环的将所有的子 context 取消掉
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
    // 将所有的子 context 和当前 context 关系解除
	c.children = nil
	c.mu.Unlock()

    // 如果需要将当前 context 从 parent context 移除，就移除掉
	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

#### 超时自动取消如何实现: WithDeadline, WithTimeout

我们先看看比较常用的 WithTimeout, 可以发现 WithTimeout 其实就是调用了 WithDeadline 然后再传入的参数上用当前时间加上了 timeout 的时间

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
	return WithDeadline(parent, time.Now().Add(timeout))
}
```

再来看一下实现超时的 timerCtx，WithDeadline 我们放到后面一点点

```go
type timerCtx struct {
	cancelCtx // 这里复用了 cancelCtx
	timer *time.Timer // Under cancelCtx.mu.

	deadline time.Time // 这里保存了快到期的时间
}
```

`Deadline()` 就是返回了结构体中保存的过期时间

```go
func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
	return c.deadline, true
}
```

`cancel` 其实就是复用了 cancelCtx 中的取消方法，唯一区别的地方就是在后面加上了对 timer 的判断，如果 timer 没有结束主动结束 timer

```go
func (c *timerCtx) cancel(removeFromParent bool, err error) {
	c.cancelCtx.cancel(false, err)
	if removeFromParent {
		// Remove this timerCtx from its parent cancelCtx's children.
		removeChild(c.cancelCtx.Context, c)
	}
	c.mu.Lock()
	if c.timer != nil {
		c.timer.Stop()
		c.timer = nil
	}
	c.mu.Unlock()
}
```

timerCtx 并没有重新实现 Done() 和 Value 方法，直接复用了 cancelCtx 的相关方法

最后我们再看看这个最重要的 WithDeadline 方法

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}

   	// 会先判断 parent context 的过期时间，如果过期时间比当前传入的时间要早的话，就没有必要再设置过期时间了
    // 只需要返回 WithCancel 就可以了，因为在 parent 过期的时候，子 context 也会被取消掉
	if cur, ok := parent.Deadline(); ok && cur.Before(d) {
		// The current deadline is already sooner than the new one.
		return WithCancel(parent)
	}

    // 构造相关结构体
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}

    // 和 WithCancel 中的逻辑相同，构建上下文关系
	propagateCancel(parent, c)

    // 判断传入的时间是不是已经过期，如果已经过期了就 cancel 掉然后再返回
	dur := time.Until(d)
	if dur <= 0 {
		c.cancel(true, DeadlineExceeded) // deadline has already passed
		return c, func() { c.cancel(false, Canceled) }
	}
	c.mu.Lock()
	defer c.mu.Unlock()

    // 这里是超时取消的逻辑，启动 timer 时间到了之后就会调用取消方法
	if c.err == nil {
		c.timer = time.AfterFunc(dur, func() {
			c.cancel(true, DeadlineExceeded)
		})
	}
	return c, func() { c.cancel(true, Canceled) }
}
```

可以发现超时控制其实就是在复用 cancelCtx 的基础上加上了一个 timer 来做定时取消

#### 如何为 Context 附加一些值: WithValue

WithValue 相对简单一点，主要就是校验了一下 Key 是不是可比较的，然后构造出一个 valueCtx 的结构

```go
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

valueCtx 主要就是嵌入了 parent context 然后附加了一个 key val

```
type valueCtx struct {
	Context
	key, val interface{}
}
```

Value 的查找和之前 cancelCtx 类似，都是先判断当前有没有，没有就向上递归，只是在 cancelCtx 当中 key 是一个固定的 key 而已

```go
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

Value 就没有实现 Context 接口的其他方法了，其他的方法全都是复用的 parent context 的方法

### 使用场景

#### 超时控制

这就是文章开始时候第一个场景下的一个例子

```go
package main

import (
	"context"
	"fmt"
	"time"
)

// 模拟一个耗时的操作
func rpc() (string, error) {
	time.Sleep(100 * time.Millisecond)
	return "rpc done", nil
}

type result struct {
	data string
	err  error
}

func handle(ctx context.Context, ms int) {
	ctx, cancel := context.WithTimeout(ctx, time.Duration(ms)*time.Millisecond)
	defer cancel()

	r := make(chan result)
	go func() {
		data, err := rpc()
		r <- result{data: data, err: err}
	}()

	select {
	case <-ctx.Done():
		fmt.Printf("timeout: %d ms, context exit: %+v\n", ms, ctx.Err())
	case res := <-r:
		fmt.Printf("result: %s, err: %+v\n", res.data, res.err)
	}
}

func main() {
	// 这里模拟接受请求，启动一个协程去发起请求
	for i := 1; i < 5; i++ {
		time.Sleep(1 * time.Second)
		go handle(context.Background(), i*50)
	}

	// for test, hang
	time.Sleep(time.Second)
}
```

执行结果

```
▶ go run *.go
timeout: 50 ms, context exit: context deadline exceeded
result: rpc done, err: <nil>
result: rpc done, err: <nil>
result: rpc done, err: <nil>
```

我们可以发现在第一次执行的时候传入的超时时间 50ms 程序超时直接退出了，但是后面超过 50ms 的时候均返回了结果。

#### 错误取消

这是第二个场景的一个例子，假设我们在 main 中并发调用了 `f1` `f2` 两个函数，但是 `f1` 很快就返回了，但是 `f2` 还在阻塞

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"
)

func f1(ctx context.Context) error {
	select {
	case <-ctx.Done():
		return fmt.Errorf("f1: %w", ctx.Err())
	case <-time.After(time.Millisecond): // 模拟短时间报错
		return fmt.Errorf("f1 err in 1ms")
	}
}

func f2(ctx context.Context) error {
	select {
	case <-ctx.Done():
		return fmt.Errorf("f2: %w", ctx.Err())
	case <-time.After(time.Hour): // 模拟一个耗时操作
		return nil
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		defer wg.Done()
		if err := f1(ctx); err != nil {
			fmt.Println(err)
			cancel()
		}
	}()

	go func() {
		defer wg.Done()
		if err := f2(ctx); err != nil {
			fmt.Println(err)
			cancel()
		}
	}()

	wg.Wait()
}
```

执行结果，可以看到 f1 返回之后 f2 立即就返回了，并且报错 context 被取消

```
▶ go run *.go
f1 err in 1ms
f2: context canceled
```

细心的同学可能发现了，这个例子不就是 errgroup 的逻辑么，是的它就是类似 errgroup 的简单逻辑，这时候再反过来去看一下 《[Week03: Go 并发编程(七) 深入理解 errgroup - Mohuishou](https://lailin.xyz/post/go-training-week3-errgroup.html)》这篇文章可能会有不一样的体会

#### 传递共享数据

一般会用来传递 tracing id, request id 这种数据，不要用来传递可选参数，这里借用一下饶大的一个例子，在实际的生产案例中我们代码也是这样大同小异

```go
const requestIDKey int = 0

func WithRequestID(next http.Handler) http.Handler {
	return http.HandlerFunc(
		func(rw http.ResponseWriter, req *http.Request) {
			// 从 header 中提取 request-id
			reqID := req.Header.Get("X-Request-ID")
			// 创建 valueCtx。使用自定义的类型，不容易冲突
			ctx := context.WithValue(
				req.Context(), requestIDKey, reqID)

			// 创建新的请求
			req = req.WithContext(ctx)

			// 调用 HTTP 处理函数
			next.ServeHTTP(rw, req)
		}
	)
}

// 获取 request-id
func GetRequestID(ctx context.Context) string {
	ctx.Value(requestIDKey).(string)
}

func Handle(rw http.ResponseWriter, req *http.Request) {
	// 拿到 reqId，后面可以记录日志等等
	reqID := GetRequestID(req.Context())
	...
}

func main() {
	handler := WithRequestID(http.HandlerFunc(Handle))
	http.ListenAndServe("/", handler)
}
```

#### 在某些情况下可以用来防止 goroutine 泄漏

我们看一下官方文档的这个例子, 这里面 gen 这个函数中如果不使用 context done 来控制的话就会导致 goroutine 泄漏，因为这里面的 for 是一个死循环，没有 ctx 就没有相关的退出机制

```go
func main() {
	// gen generates integers in a separate goroutine and
	// sends them to the returned channel.
	// The callers of gen need to cancel the context once
	// they are done consuming generated integers not to leak
	// the internal goroutine started by gen.
	gen := func(ctx context.Context) <-chan int {
		dst := make(chan int)
		n := 1
		go func() {
			for {
				select {
				case <-ctx.Done():
					return // returning not to leak the goroutine
				case dst <- n:
					n++
				}
			}
		}()
		return dst
	}

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel() // cancel when we are finished consuming integers

	for n := range gen(ctx) {
		fmt.Println(n)
		if n == 5 {
			break
		}
	}
}
```

## 总结

### 使用准则

context 包一开始就告诉了我们应该怎么用，不应该怎么用，这是应该被共同遵守的约定。

- 对 server 应用而言，传入的请求应该创建一个 context，接受
- 通过 `WithCancel` , `WithDeadline` , `WithTimeout` 创建的 Context 会同时返回一个 cancel 方法，这个方法必须要被执行，不然会导致 context 泄漏，这个可以通过执行 `go vet` 命令进行检查
- 应该将 `context.Context` 作为函数的第一个参数进行传递，参数命名一般为 `ctx` 不应该将 Context 作为字段放在结构体中。
- 不要给 context 传递 nil，如果你不知道应该传什么的时候就传递 `context.TODO()`
- 不要将函数的可选参数放在 context 当中，context 中一般只放一些全局通用的 metadata 数据，例如 tracing id 等等
- context 是并发安全的可以在多个 goroutine 中并发调用

### 使用场景

- 超时控制
- 错误取消
- 跨 goroutine 数据同步
- 防止 goroutine 泄漏

### 缺点

- 最显著的一个就是 context 引入需要修改函数签名，并且会病毒的式的扩散到每个函数上面，不过这个见仁见智，我看着其实还好
- 某些情况下虽然是可以做到超时返回提高用户体验，但是实际上是不会退出相关 goroutine 的，这时候可能会导致 goroutine 的泄漏，针对这个我们来看一个例子

我们使用标准库的 timeout handler 来实现超时控制，底层是通过 context 来实现的。我们设置了超时时间为 1ms 并且在 handler 中模拟阻塞 1000s 不断的请求，然后看 pprof 的 goroutine 数据

```go
package main

import (
	"net/http"
	_ "net/http/pprof"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		// 这里阻塞住，goroutine 不会释放的
		time.Sleep(1000 * time.Second)
		rw.Write([]byte("hello"))
	})
	handler := http.TimeoutHandler(mux, time.Millisecond, "xxx")
	go func() {
		if err := http.ListenAndServe("0.0.0.0:8066", nil); err != nil {
			panic(err)
		}
	}()
	http.ListenAndServe(":8080", handler)
}

```

查看数据我们可以发现请求返回后， goroutine 其实并未回收，但是如果不阻塞的话是会立即回收的

```
goroutine profile: total 29
24 @ 0x103b125 0x106cc9f 0x1374110 0x12b9584 0x12bb4ad 0x12c7fbf 0x106fd01
```

我们来看看它的源码，超时控制主要在 ServeHTTP 中实现，我删掉了部分不关键的数据， 我们可以看到函数内部启动了一个 goroutine 去处理请求逻辑，然后再外面等待，但是这里的问题是，当 context 超时之后 ServeHTTP 这个函数就直接返回了，在这里面启动的这个 goroutine 就没人管了

```go
func (h *timeoutHandler) ServeHTTP(w ResponseWriter, r *Request) {
	ctx := h.testContext
	if ctx == nil {
		var cancelCtx context.CancelFunc
		ctx, cancelCtx = context.WithTimeout(r.Context(), h.dt)
		defer cancelCtx()
	}
	r = r.WithContext(ctx)
	done := make(chan struct{})
	tw := &timeoutWriter{
		w:   w,
		h:   make(Header),
		req: r,
	}
	panicChan := make(chan interface{}, 1)
	go func() {
		defer func() {
			if p := recover(); p != nil {
				panicChan <- p
			}
		}()
		h.handler.ServeHTTP(tw, r)
		close(done)
	}()
	select {
	case p := <-panicChan:
		panic(p)
	case <-done:
		// ...
	case <-ctx.Done():
		// ...
	}
}
```

### 总结

context 是一个优缺点都十分明显的包，这个包目前基本上已经成为了在 go 中做超时控制错误取消的标准做法，但是为了添加超时取消我们需要去修改所有的函数签名，对代码的侵入性比较大，如果之前一直都没有使用后续再添加的话还是会有一些改造成本

### 参考文献

1. [context · pkg.go.dev](https://pkg.go.dev/context)
2. [Go 语言实战笔记（二十）| Go Context](https://www.flysnow.org/2017/05/12/go-in-action-go-context.html#初识context)
3. [Go 语言并发编程与 Context | Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-context/)
4. [深度解密 Go 语言之 context | qcrao](https://qcrao.com/2019/06/12/dive-into-go-context/)
5. [Go Concurrency Patterns: Context - The Go Blog](https://blog.golang.org/context)
6. [Go Concurrency Patterns: Pipelines and cancellation - The Go Blog](https://blog.golang.org/pipelines)