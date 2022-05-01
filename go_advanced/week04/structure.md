# 工程项目架构

## 项目目录结构

### 标准的 Go 项目结构

#### /cmd 

一般采用 `/cmd/appname/main.go` 的形式进行组织。

- 首先 cmd 目录下一般是项目的主干目录。
- 这个目录下的文件**不应该有太多的代码，不应该包含业务逻辑**。

#### /internal

internal 目录下的包，**不允许被其他项目导入**，这是在 Go 1.4 当中引入的 feature，会在编译时执行。

- 所以我们一般会把项目文件夹放置到 internal 当中，例如 `/internal/app`。
- 如果是**可以被其他项目导入的包我们一般会放到 /pkg 目录下**。
- 如果是我们项目内部进行共享的包，而不期望外部共享，我们可以放到 `/internal/pkg` 当中。
- 注意 internal 目录的限制并不局限于顶级目录，在任何目录当中都是生效的。

#### /pkg

一般而言，我们在 pkg 目录下放置**可以被外部程序安全导入的包**，对于不应该被外部程序依赖的包我们应该放置到 `internal` 目录下， `internal` 目录会有编译器进行强制验证。

- pkg 目录下的包一般会按照功能进行区分，例如 `/pkg/cache` 、 `/pkg/conf` 等。
- 如果你的目录结构比较简单，内容也比较少，其实也可以不使用 `pkg` 目录，直接把上面的这些包放在最上层即可。
- 一般而言我们应用程序 app 在最外层会包含很多文件，例如 `.gitlab-ci.yml` `makefile` `.gitignore` 等等，这种时候顶层目录会很多并且会有点杂乱，建议还是放到 `/pkg` 目录比较好。

可参考文章 [I'll take pkg over internal](https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/)

#### Kit Project Layout

kit 库其实也就是一些基础库。

- 每一个公司正常来说应该**有且仅有一个基础库项目**。
- kit 库一般会包含一些常用的公共的方法，例如缓存，配置等等，比较典型的例子就是 [go-kit](https://github.com/go-kit/kit)。
- kit 库必须具有的特点：
  - 统一
  - 标准库方式布局
  - 高度抽象
  - 支持插件
  - 尽量减少依赖
  - 持续维护

### 服务目录结构

#### /api

API 协议定义目录，主要存放 `app_api.proto` protobuf 文件，以及生成的 go 文件。我们通常把 api 文档直接在 proto 文件中描述。

#### /configs

配置文件模板或默认配置。

#### /test

额外的外部测试应用程序和测试数据。你可以随时根据需求构造 /test 目录。对于较大的项目，有一个数据子目录是有意义的。例如，你可以使用 /test/data 或 /test/testdata (如果你需要忽略目录中的内容)。请注意，Go 还会忽略以“.”或“_”开头的目录或文件，因此在如何命名测试数据目录方面有更大的灵活性。

### 服务类型

微服务中的 app 服务类型分为 4 类：interface、service、job、admin。

- interface: 对外的 BFF 服务，接受来自用户的请求，比如暴露了 HTTP/gRPC 接口。
- service: 对内的微服务，仅接受来自内部其他服务或者网关的请求，比如暴露了 gRPC 接口只对内服务。
- admin：区别于 service，更多是面向运营测的服务，通常数据权限更高，隔离带来更好的代码级别安全。
- job: 流式任务处理的服务，上游一般依赖 message broker。
- task: 定时任务，类似 cronjob，部署到 task 托管平台中。

![image-20220429181915897](/Users/tianyou/Library/Application Support/typora-user-images/image-20220429181915897.png)

cmd 应用目录负责程序的: 启动、关闭、配置初始化等。

### 服务布局

#### 项目 v1 布局

![image-20220429182312560](/Users/tianyou/Library/Application Support/typora-user-images/image-20220429182312560.png)

项目的依赖路径为: model -> dao -> service -> api，model struct 串联各个层，直到 api 需要做 DTO 对象转换。

- model: 放对应“存储层”的结构体，是对存储的一一隐射。
- dao: 数据读写层，数据库和缓存全部在这层统一处理，包括 cache miss 处理。
- service: 组合各种数据访问来构建业务逻辑。
- server: 依赖 proto 定义的服务作为入参，提供快捷的启动服务全局方法。
- api: 定义了 API proto 文件，和生成的 stub 代码，它生成的 interface，其实现者在 service 中。
- service 的方法签名因为实现了 API 的 接口定义，DTO 直接在业务逻辑层直接使用了，更有 dao 直接使用，最简化代码。
- DO(Domain Object): 领域对象，就是从现实世界中抽象出来的有形或无形的业务实体。缺乏 DTO -> DO 的对象转换。

##### v1 存在的问题

- 没有 DTO 对象，model 中的对象贯穿全局，所有层都有。
  - model 层的数据不是每个接口都需要的，这个时候会有一些问题。

#### 项目 v2 布局

<img src="/Users/tianyou/Library/Application Support/typora-user-images/image-20220429182510788.png" alt="image-20220429182510788" style="zoom:50%;" />

app 目录下有 api、cmd、configs、internal 目录，目录里一般还会放置 README、CHANGELOG、OWNERS。

internal: 是为了避免有同业务下有人跨目录引用了内部的 biz、data、service 等内部 struct。

- biz: 业务逻辑的组装层，类似 DDD 的 domain 层，data 类似 DDD 的 repo，repo 接口在这里定义，使用**依赖倒置**的原则。
- data: 业务数据访问，包含 cache、db 等封装，实现了 biz 的 repo 接口。我们可能会把 data 与 dao 混淆在一起，data 偏重业务的含义，它所要做的是将领域对象重新拿出来，我们去掉了 DDD 的 infra 层。
- service: 实现了 api 定义的服务层，类似 DDD 的 application 层，处理 DTO 到 biz 领域实体的转换(DTO -> DO)，同时协同各类 biz 交互，但是不应处理复杂逻辑。

PO(Persistent Object): 持久化对象，它跟持久层（通常是关系型数据库）的数据结构形成一一对应的映射关系，如果持久层是关系型数据库，那么数据表中的每个字段（或若干个）就对应 PO 的一个（或若干个）属性。https://github.com/facebook/ent

示例可以参考 [kratos v2 的 example](https://github.com/go-kratos/examples/tree/main/blog)

布局图如下：

![image-20220429183038959](/Users/tianyou/Library/Application Support/typora-user-images/image-20220429183038959.png)







## 参考文献

1. [Go 进阶训练营-极客时间](https://u.geekbang.org/subject/go?utm_source=lailin.xyz&utm_medium=lailin.xyz)
2. [golang-standards/project-layout · GitHub](https://github.com/golang-standards/project-layout/blob/master/README_zh.md)
3. [Package Oriented Design](https://www.ardanlabs.com/blog/2017/02/package-oriented-design.html)
4. [Go 1.4 Release Notes - The Go Programming Language](https://golang.org/doc/go1.4#internalpackages)
5. [I’ll take pkg over internal](https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/)
6. https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1