# 日志

## Kratos Log

### 源码浅析

浅析一下 [kratos](https://github.com/go-kratos/kratos) 日志模块代码，里面 fmt... 的语句都是我加到里面调试用的。

其中 Logger 接口是 kratos 定义的统一的日志接口入口，这个 logger 可以使用其他的第三方库的 log 来打印日志。比如下面的 stdLog，也就是官方 log 库，也可以使用比如 fluent 库，在 contrib/log/fluent/fluent.go 文件里。

这样做的好处是可以任意使用第三方库来打印日志，而不至于让其他库和自己的项目过于耦合。先直接看代码吧：

log/log.go:

```go
// DefaultLogger is default logger.
var DefaultLogger = NewStdLogger(log.Writer())

// Logger is a logger interface.
type Logger interface {
	Log(level Level, keyvals ...interface{}) error
}

type logger struct {
	logs      []Logger
	prefix    []interface{}
	hasValuer bool
	ctx       context.Context
}

func (c *logger) Log(level Level, keyvals ...interface{}) error {
  fmt.Println("-----log log")
	kvs := make([]interface{}, 0, len(c.prefix)+len(keyvals))
	kvs = append(kvs, c.prefix...)
	if c.hasValuer {
		// 将 kvs 中的 Valuer 对象解析出 timestamp 和 caller，重新赋值到 kvs 里
		bindValues(c.ctx, kvs)
	}
	fmt.Printf("kvs %v\n", kvs)
	kvs = append(kvs, keyvals...)
	for _, l := range c.logs {
    // 调用第三方库的 Log 方法
		if err := l.Log(level, kvs...); err != nil {
			return err
		}
	}
	return nil
}

// With with logger fields.
func With(l Logger, kv ...interface{}) Logger {
	if c, ok := l.(*logger); ok {
		kvs := make([]interface{}, 0, len(c.prefix)+len(kv))
		kvs = append(kvs, kv...)
		kvs = append(kvs, c.prefix...)
		return &logger{
			logs:      c.logs,
			prefix:    kvs,
			hasValuer: containsValuer(kvs),
			ctx:       c.ctx,
		}
	}
	return &logger{logs: []Logger{l}, prefix: kv, hasValuer: containsValuer(kv)}
}

```

这个 `Logger` 接口只有一个 `Log` 方法，`logger` 结构体实现了这个方法。

log/std.go：

```go
var _ Logger = (*stdLogger)(nil)

type stdLogger struct {
	log  *log.Logger
	pool *sync.Pool
}

// NewStdLogger new a logger with writer.
func NewStdLogger(w io.Writer) Logger {
	return &stdLogger{
		log: log.New(w, "", 0),
		pool: &sync.Pool{
			New: func() interface{} {
				return new(bytes.Buffer)
			},
		},
	}
}

// Log print the kv pairs log.
func (l *stdLogger) Log(level Level, keyvals ...interface{}) error {
	fmt.Println("-----std log")
	if len(keyvals) == 0 {
		return nil
	}
	if (len(keyvals) & 1) == 1 {
		keyvals = append(keyvals, "KEYVALS UNPAIRED")
	}
	buf := l.pool.Get().(*bytes.Buffer)
	buf.WriteString(level.String())
	for i := 0; i < len(keyvals); i += 2 {
		_, _ = fmt.Fprintf(buf, " %s=%v", keyvals[i], keyvals[i+1])
	}
	_ = l.log.Output(4, buf.String()) //nolint:gomnd
	buf.Reset()
	l.pool.Put(buf)
	return nil
}

```

定义了 `stdLogger` 结构体里包含了标准库的 log 和一个日志的缓冲区，实现了 `Logger` 接口的 `Log` 方法。 

log/log_test.go：

```go
func TestInfo(t *testing.T) {
	logger := DefaultLogger
	logger = With(logger, "ts", DefaultTimestamp, "caller", DefaultCaller)
	_ = logger.Log(LevelInfo, "key1", "value1")
}
```

我们根据测试函数代码一步步解析：

测试函数里定义了一个 `DefaultLogger`，这个其实就是最上面的 `var DefaultLogger = NewStdLogger(log.Writer())` 即 `stdLogger` 类型。

然后调用 `With` 函数，参数里的 `DefaultTimestamp` 和 `DefaultCaller` 是一个 `Valuer` 对象，在 log/value.go 里实现，可以简单理解为运行时会将这两个对象通过 `BindValues` 方法转化成当前时间和当前调用地址。

 `With` 函数内部，因为 `stdLogger` 和 `logger` 不是一个结构体，所以 `if c, ok := l.(*logger);` 这里的 ok 为 false，会将 `stdLogger` 对象放到 `[]Logger` 数组中，转化成 `logger` 结构体返回。

最后 `logger.Log(LevelInfo, "key1", "value1")`，此时这个 `logger` 已经是 log.go 的 `logger` 结构体了，调用 `Log` 方法，会先执行 log.go 的 `Log` 方法，然后这个 `Log` 方法会遍历 `[]Logger` 数组再调用对应对象的 `Log` 方法。

我们看下测试输出：

```shell
=== RUN   TestInfo
-----log log
kvs [ts 2022-05-17T15:55:21+08:00 caller log_test.go:12]
-----std log
INFO ts=2022-05-17T15:55:21+08:00 caller=log_test.go:12 key1=value1
--- PASS: TestInfo (0.00s)
PASS
```

可以看到先打印 "-----log log"，即 log.go 的 `Log` 方法，将时间戳和调用地址解析出来，然后再打印 "-----std log"，即调用 stdLogger 的 `Log` 方法，将传入的 "key1", "value1" 输出出来，这样我们就将 `stdLogger` 和 `logger` 解耦出来，这个 `stdLogger` 也可以替换成别的第三方库，只要它实现了 `Log` 方法。

一个细节，在 std.go 里有一行代码：

```go
var _ Logger = (*stdLogger)(nil)
```

其中 Logger 类型是 log.go 里的：

```go
// log/log.go
// Logger is a logger interface.
type Logger interface {
	Log(level Level, keyvals ...interface{}) error
}
```

这行代码的意思就是**判断 `stdLogger` 是否实现了 `Logger` 接口的所有方法**，给 `stdLogger` 初始化一个 nil，并且忽略掉 Logger 对象。这个方式可以很方便地判断自己的结构体是否实现了对应接口的全部方法，可以放在文件最开始，如果没完全实现接口，代码就会报错。属于一个小技巧。当然也可以这么写，也是一样的：`var _ Logger = &stdLogger{}`

### 示例

再看一下 [example](https://github.com/go-kratos/examples) 库里日志的使用方式。

blog/cmd/blog/main.go：

```go
func main() {
	flag.Parse()
	logger := log.NewStdLogger(os.Stdout)

	cfg := config.New(
		config.WithSource(
			file.NewSource(flagconf),
		),
	)
  
	...
  
	app, cleanup, err := initApp(bc.Server, bc.Data, logger)
	if err != nil {
		panic(err)
	}
	defer cleanup()

	// start and wait for stop signal
	if err := app.Run(); err != nil {
		panic(err)
	}
}
```

blog/cmd/blog/wire_gen.go：

```go
//go:generate go run github.com/google/wire/cmd/wire
//go:build !wireinject
// +build !wireinject
...
// initApp init kratos application.
func initApp(confServer *conf.Server, confData *conf.Data, logger log.Logger) (*kratos.App, func(), error) {
	dataData, cleanup, err := data.NewData(confData, logger)
	if err != nil {
		return nil, nil, err
	}
	articleRepo := data.NewArticleRepo(dataData, logger)
	articleUsecase := biz.NewArticleUsecase(articleRepo, logger)
	blogService := service.NewBlogService(articleUsecase, logger)
	httpServer := server.NewHTTPServer(confServer, logger, blogService)
	grpcServer := server.NewGRPCServer(confServer, logger, blogService)
	app := newApp(logger, httpServer, grpcServer)
	return app, func() {
		cleanup()
	}, nil
}
```

可以看到 main 函数里也是使用了 `log.NewStdLogger`，然后将 `logger` 通过 wire 注入到各个模块中。

这些模块（比如 `data.NewArticleRepo`）里的 logger 参数都是 `log.Logger` 类型，也就是最上面 log.go 里的 `Logger interface`，这样我们的方法**只依赖接口**，怎么实现的不需要管，可以是实现了 Log 方法的标准库，或者其他的库都可以传入，这样想随时替换第三方日志库都可以很方便地在 main 函数里更改，而不需要改太多代码。



## 日志系统

### 日志选型

一个完整的集中式日志系统，需要包含以下几个主要特点：

- 收集－能够采集多种来源的日志数据；
- 传输－能够稳定的把日志数据传输到中央系统；
- 存储－如何存储日志数据；
- 分析－可以支持 UI 分析；
- 警告－能够提供错误报告，监控机制；

开源界鼎鼎大名 ELK stack，分别表示：Elasticsearch , Logstash, Kibana , 它们都是开源软件。新增了一个 FileBeat，它是一个轻量级的日志收集处理工具(Agent)，Filebeat 占用资源少，适合于在各个服务器上搜集日志后传输给 Logstash，官方也推荐此工具。

![elk](/Users/tianyou/Library/Application Support/typora-user-images/image-20220517164737105.png)



### 格式规范

JSON作为日志的输出格式：

- time: 日志产生时间，ISO8601格式；
- level: 日志等级，ERROR、WARN、 INFO、DEBUG；
- app_id: 应用id，用于标示日志来源；
- instance_id: 实例 id，用于区分同一应用不同实例，即 hostname；

![样例格式](/Users/tianyou/Library/Application Support/typora-user-images/image-20220517165243389.png)



或者可以直接参考 [otel 规范](https://github.com/open-telemetry/opentelemetry-specification) 。

### 采集

logstash：

- 监听 tcp/udp
- 适用于通过网络上报日志的方式

filebeat：

- 直接采集本地生成的日志文件
- 适用于日志无法定制化输出的应用

logagent（b 站自研）：

- 物理机部署，监听 unixsocket
- 日志系统提供各种语言 SDK
- 直接读取本地日志文件

容器内日志采集：

基于 overlay2，直接从物理机上查找对应日志文件。

![配置](/Users/tianyou/Library/Application Support/typora-user-images/image-20220517165641111.png)

### 传输

可以使用 Flink + Kafka 的方式。

### 本地文件方案

使用自定义协议，对 SDK 质量、版本升级都有比较高的要求，因此我们长期会使用“本地文件”的方案实现：

采集本地日志文件：位置不限，容器内 or 物理机。

配置自描述：不做中心化配置，配置由 app/paas 自身提供，agent 读取配置并生效。

日志不重不丢：多级队列，能够稳定地处理日志收集过程中各种异常。

可监控：实时监控运行状态。

完善的自我保护机制：限制自身对于宿主机资源的消耗，限制发送速度。