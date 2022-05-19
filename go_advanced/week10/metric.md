# 指标

## 监控

监控：

- 延迟、流量、错误、饱和度（4 个黄金指标）
- 长尾问题（一些耗时特别长的请求需要单独看是什么问题）
- 依赖资源 (Client/Server 's view)（分别从客户端和服务端的视角查看，比如请求耗时，可以看出网络是否是耗时点）

链路追踪 (Google Dapper)：

- jaeger
- zipkin

日志：

- traceid关联

指标：

- Prometheus + Granfana



涉及到 net、cache、db、rpc 等资源类型的基础库，首先监控维度 4 个黄金指标：

- 延迟（耗时，需要区分正常还是异常）
- 流量（需要覆盖来源，即：caller）
- 错误（覆盖错误码或者 HTTP Status Code）
- 饱和度（服务容量有多“满”）

系统层面

- CPU，Memory，IO，Network，TCP/IP 状态等，FD（等其他）
- Kernel：Context Switch
- Runtime：各类 GC、Mem 内部状态等

服务监控

- 线上打开 Profiling 的端口；
- 使用服务发现找到节点信息，以及提供快捷的方式快速可以 WEB 化查看进程的 Profiling 信息（火焰图等）；
- watchdog，使用内存、CPU 等信号量触发自动采集，能够在发生异常情况时迅速采集数据，不用人工干预，减少成本；

