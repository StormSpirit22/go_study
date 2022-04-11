# Goroutine

由于我目前看的《Go 进阶训练营》是第四期，而 [**mohuishou**](https://lailin.xyz/) 博客里的是第 0 期，内容有些微不同，主要是例子不太一样，不过大致思想是一样的，我这里记录一下第四期的笔记，第 0 期笔记：[Go并发编程(一) goroutine](https://lailin.xyz/post/go-training-week3-goroutine.html)



## 对你创建的 goroutine 负责

对于 goroutine 的使用，有以下几点需要注意：

### Keep yourself busy or do the work yourself

如果你的 goroutine 在从另一个 goroutine 获得结果之前无法取得进展，那么通常情况下，你自己去做这项工作比委托它( go func() )更简单。

比如以下这个例子：

```go
func main() {
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World")
  })
  go func() {
    if err := http.ListenAndServe(":8080", nil); err != nil {
      log.Fatal(err)
    }
  }()
  
  select {}
}
```

其实完全可以将 goroutine 里的东西放到 main 函数里执行。

```go
func main() {
  http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World")
  })

  if err := http.ListenAndServe(":8080", nil); err != nil {
    log.Fatal(err)
  }
}
```



### Leave concurrency to the caller

将是否并发留给调用者来选择。一般基础库的方法不太需要使用 goroutine，因为调用它的应用层无法控制这个 goroutine 的生命周期，所以并发可以留给应用层来决定是否使用。

有这两个 api：

```go
func ListDirectory(dir string) ([]string, error)

func ListDirectory(dir string) chan string
```

第一个 api 读取完一个目录下所有的文件再返回，需要同步等待。第二个 api 返回一个 channel ，调用者需要持续读取。

ListDirectory chan 版本有两个问题：

- 通过使用一个关闭的通道作为不再需要处理的项目的信号，ListDirectory 无法告诉调用者通过通道返回的项目集不完整，因为中途遇到了错误。调用方无法区分空目录与完全从目录读取的错误之间的区别。这两种方法都会导致从 ListDirectory 返回的通道会立即关闭。
- 调用者必须持续从通道读取，直到它关闭，因为这是调用者知道填充 chan 的 goroutine 已经停止的唯一方法。这对 ListDirectory 的使用是一个严重的限制，调用者必须花时间从通道读取数据，即使它可能已经收到了它想要的答案。对于大中型目录，它可能在内存使用方面更为高校，但这种方法并不比原始的基于 slice 的方法快。

可以使用以下 api 的方式来改进：

```go
func ListDirectory(dir string, fn func(string)) 
```

传入一个函数，它来决定如何处理文件。



### Never start a goroutine without knowning when it will stop

不要创建一个你不知道何时退出的 goroutine。

比如有这么个例子：

```go
func search(term string) (string, error) {
  time.Sleep(200 * time.Millisecond)
  return "some value", nil
}

func process(term string, ctx context.Context) error {
  go func() {
    record, err := search(term)
    ch <- result{record, err}
  }()

  select {
  case <- ctx.Done():
    return errors.New("search canceled")
  case result := <-ch:
    if result.err != nil {
      return result.err
    }
    fmt.Println("Received:", result.record)
    return nil
  }
}
```

search 函数是一个模拟实现，用于模拟长时间运行的操作，如数据库查询或 rpc 调用。在本例中，硬编码为200ms。定义了一个名为 process 的函数，接受字符串参数，传递给 search。对于某些应用程序，顺序调用产生的延迟可能是不可接受的。

这个例子有个问题，如果 select 里的 ctx 被取消了，那么 ch 的值就不会被读取出来，而 go func 里的 ch <- 会一直阻塞住（因为它是个无缓冲 channel），这个 goroutine 就会泄露。解决方法是可以将 ch 设置成有缓冲的 channel，这样长时间之后就会被 gc 回收掉。

任何时候开启一个 goroutine 你都应该问自己：

- 它什么时候会结束？
- 什么条件会阻止它结束？

再看一个 http 服务的例子：

```go
func serveApp() {
  mux := http.NewServeMux()
  mux.HandleFunc("/", func(resp http.ResponseWriter, req *http.Request) {
    fmt.Fprintln(resp, "Hello, QCon!")
  })
  if err := http.ListenAndServe("0.0.0.0:8080", mux); err != nil {
    log.Fatal(err)
  }
}

func serveDebug() {
  if err := http.ListenAndServe("127.0.0.1:8001", mux); err != nil {
    log.Fatal(err)
  }
}

func main() {
  go serveDebug()
  go serveApp()
  select{}
}
```

这个例子需要开启两个 http 服务，这里使用两个 goroutine 来启动。

问题：

serveDebug 是在一个单独的 goroutine 中运行的，如果它返回，那么所在的 goroutine 将退出，而程序的其余部分继续运行。由于 /debug 处理程序很久以前就停止工作了，所以其他同学会很不高兴地发现他们无法在需要时从您的应用程序中获取统计信息。

如果 ListenAndServer 返回 error，由于在 goroutine 里，所以 main.main 无法退出。而且 **log.Fatal 调用了 os.Exit，defers 不会被调用到**（建议：只有在 main 函数和 init 函数里才能调用 log.Fatal，基础库里一般不使用）。

改良：

```go
func serve(addr string, handler http.Handler, stop <- chan struct{}) error {
  s := http.Server {
    Addr: addr,
    Handler: handler,
  }
  
  go func() {
    <- stop
    s.Shutdown(context.Background())
  }()
  
  return s.ListenAndServe()
}

func main() {
  done := make(chan error, 2)
  stop := make(chan struct{})
  go func() {
    done <- serveDebug(stop)
  }()
  go func() {
    done <- serveApp(stop)
  }()
  
  var stopped bool 
  for i := 0; i < cap(done); i++ {
    if err := <- done; err != nil {
      fmt.Println("error: %v", err)
    }
    if stopped {
      stopped = true
      close(stop)
    }
  }
}
```

这里通过 done channel 来获取是否有一个协程退出，比如 serveDebug 退出了，然后 for 循环里 done 会输出错误，然后关闭 stop channel，这样 serveApp 里的 serve 函数的 go func 就会执行 shutdown，这样 serveApp 协程也退出了。



最后一个例子：

```go 
package main

import (
	"context"
	"fmt"
	"time"
)

func main() {
	tr := NewTracker()
	go tr.Run()
	_ = tr.Event(context.Background(), "test1")
	_ = tr.Event(context.Background(), "test2")
	_ = tr.Event(context.Background(), "test3")
	time.Sleep(3 * time.Second)
	ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5*time.Second))
	defer cancel()
	tr.Shutdown(ctx)
}

func NewTracker() *Tracker {
	return &Tracker{
		ch: make(chan string, 10),
	}
}

type Tracker struct {
	ch   chan string
	stop chan struct{}
}

func (t *Tracker) Event(ctx context.Context, data string) error {
	select {
	case t.ch <- data:
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func (t *Tracker) Run() {
	for data := range t.ch {
		time.Sleep(1 * time.Second)
		fmt.Println(data)
	}
	t.stop <- struct{}{}
}

func (t *Tracker) Shutdown(ctx context.Context) {
	// 最好是数据的发送者来关闭 channel。
	close(t.ch)
	select {
	case <-t.stop:
		fmt.Println("stop")
	case <-ctx.Done():
		fmt.Println("ctx cancel")
	}
}
```

输出：

```bash
test1
test2
test3
ctx cancel
```

这个例子是将数据放在后台的一个 goroutine 里处理，而不是创建大量的 goroutine 来处理数据，而且这个 goroutine 是由调用者来创建的。



## 参考文献

- https://dave.cheney.net/practical-go/presentations/qcon-china.html 这篇 dave 在 Qcon China 上的文章值得好好拜读几遍
- https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html
- https://www.ardanlabs.com/blog/2019/04/concurrency-trap-2-incomplete-work.html
- https://www.ardanlabs.com/blog/2014/01/concurrency-goroutines-and-gomaxprocs.html
