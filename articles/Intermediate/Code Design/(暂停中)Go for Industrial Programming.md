# Go for Industrial Programming

工业级Go

[原文链接](https://peter.bourgon.org/go-for-industrial-programming/)

### 介绍的上下文

（前面有一大段，作者关于提建议和接受建议的分析，没翻）

我今天要讲的东西，都是建立在一个产业级的上下文中，也就是说：

* 在一个初创公司或者企业环境中
* 在一个工程师有来有离开的团队
* 代码不依赖于特定的工程师
* 代码满足高度可变的业务要求

以上这几点也是我记忆中大部分的工作环境。我猜测对于你也是如此。这种情况的最好建议，对于个人项目，大型开源项目等等，则不一定是最好的建议。

让我们来看看要覆盖到的topic，我选择了一些我曾待过的组织中，会阻碍初级和进阶的gopher的问题，这些问题一般都会有很有意思的影响。

#### Table of Contents

1. Structuring your code and repository 结构化你的代码和仓库
2. Program configuration 程序配置
3. The componet graph 组件图
4. Goroutine lifecycle management Goroutine的生命周期管理
5. Observability 观测
6. Testing 测试
7. How much interface do I need 我需要多少接口
8. Cotext use and misusr context的使用和滥用

### Structuring your code and repository 结构化你的代码和仓库

我在同事中，特别是刚接触Go的同事中，经常看到一种情况：

他们会在项目开始的时候，决定好一个古板的项目结构。我认为这样做通常弊大于利。为什么呢？我们首先要回忆前提条件中的第4个，也可能是工业级编程中最重要的一个：

> 工业级代码...允许高度可变的商业需求

唯一不变的就是变化本省，同事商业需求中，唯一的规则就是，这些需求永远不会聚合，只会不断分支，成长和变化。我们的代码也得认清这个现实，而不是自我蒙蔽。我们的库，我们的抽象，我们的代码本身，都应该易于维护，也就是易于修改，易于添加，最终，要易于删除。

有一些特别好的办法。如果你的project有多个二进制，那你最好有一个 cmd/ 的子目录来保存他们。如果你的project非常大，并且包含非Go的代码，如UI asserts 或者 复杂的 构建工具，那么将你的代码独立在pkg/ 子目录下则是个不错的主意。如果你有个包，那么最好不要通过实现的方式 ，而是通过 领域模型 来定位它们。**这也就是指使用 package user,而不是 package models**。 关于包设计，有几个特别好的文章[Ben Johnson’s Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1) 和 [Brian Ketelsen’s GopherCon Russia talk](https://talks.bjk.fyi/bketelsen/gcru18-best#/) 。

当然了，领域 和 实现 并不是总是严格区分的，比如，大型的web应用趋向于不区分 传输 和 核心业务细节。比如 [GoBuffalO](https://gobuffalo.io/en/docs/examples/) 中鼓励如 actions, assets ,models 和 templates 等命名。

Tim Hockin 也建议我们将包以依赖边界区分开。 那也就是，依照 RedisStore, MySqlStore 这样区分，这样可以避免包的用户引入和编译他们不需要的东西。但是依我所见，这种方式对于依赖集中的包，不算是很好的优化。但是对于哪些广泛依赖于第三方的包，则很有意义。

所以这些建议都是有适用范围的。我认为最好的，最通用的建议为，只在你具体需要的时候，再添加结构。我的大部分项目，仍然由区区几个根目录下main包中的文件开始，直到它们扩充到一个文件至少几千行代码。不过我的大部分项目，都没有扩充到这么大。Go的一大好处就是非常的轻量级。我希望尽可能的保留这个特性。

### Program configuration 程序配置

我感觉我不止一次的讲过这个话题，是在这里我还得再讲一遍：**Flags 是最好的配置程序的方式** 因为它们可以自文档化的方式来描述你的程序再运行时需要的配置。

这再工业级别的上下文中，因为操作服务的人，不一定是服务的原作者了。如果用  -h flag 提供控制程序的完整提示，那么一定新的工程师就很容易调整，或者在开发环境运行。比起在文档中搜索相关的环境变量，或者匹配配置文件中的格式要容易的多。

这不意味着永远不用环境变量和配置文件。在一些情况下，使用这两者会更好：环境变量对于无安全的认证，连接字符串都非常的实用，特别是在开发环境中。而配置文件适用于描述大量的配置，同时也相对安全一些。（Flags 会被系统的其他使用者看到，而环境变量很容易忘记）。所以只需保证命令行的配置，有最高的优先级即可。

### The component graph 组件图

工业级别的代码意味着写一次，永远的维护下去。维护就是不断的读代码和重构代码。正因如此，工业界别的编程通常需要压倒性的偏向于可读性，当遇到 容易读 和 容易写 的冲突的时候，我们应该选择前者。

依赖注入对于提升可读性来说，非常的有力。我这里绝对不是指 依赖管理方案，如[facebook-go/inject](https://github.com/facebook-go/inject) 或者 [uber-go/dig](https://github.com/uber-go/dig)  。我指的是一些 能讲依赖项枚举为类型和结构体参数的 简单实现。

以下是一个 基于容器的依赖注入：

```go
func BuildContainer() *dig.Container {  
    container := dig.New()
    container.Provide(NewConfig)
    container.Provide(ConnectDatabase)
    container.Provide(NewPersonRepository)
    container.Provide(NewPersonService)
    container.Provide(NewServer)
    return container
}

func main() {  
    container := BuildContainer()
    if err := container.Invoke(func(server *Server) {
        server.Run()
    }); err != nil {
        panic(err)
    }
}
```

main 方法很小，而 BuildContainer也看起来很简洁。但是 Provide 方法需要通过反射解释它的参数，同时Invoke方法也让我们摸不清楚它需要什么才能完成它的工作。

一个新的公司雇员，需要在不同的上下文之间来回跳转，来理清这些依赖之前如何交互，是否被 server 调用。以上反馈了一个不好的，可读和可写之间的平衡。

对比我们简单优化的版本。如下：

```go
func main() {  
    cfg := GetConfig()
    db, err := ConnectDatabase(cfg.URN)
    if err != nil {
        panic(err)
    }
    repo := NewPersonRepository(db)
    service := NewPersonService(cfg.AccessToken, repo)
    server := NewServer(cfg.ListenAddr, service)
    server.Run()
}
```

main 方法更长了，但是我们增加的明确性。其中每一个组件都按照依赖顺序构建，又对应的错误处理。每一个构造器都将它的所有依赖作为参数，这样新的代码阅读者可以快速的了解到各个组件之间的关系。没有任何东西被掩盖于间接层之下。

如果重构需要一个组件新增依赖，只要简单的在构造器中添加即可。如果又使用构造器的地方没有更新，那么在下一次编译的时候就会提示出来。

-

我关于Go现在编程的准则：

* 依赖清晰
* 无包级别的变量
* 无init方法

### Goroutine 生命周期管理

依照我的经验，对于新手或者中级gopher来说，要面对的最大的问题就是启动，停止，检查goroutine所用到的，不正确或者过于复杂的设计。

我认为关键问题是: goroutine 本身开销很小，用于实现并发的代价很低。我们常说“不知道怎么关，就别开始一个goroutine”，但是这条建议如果没有方法论则非常空洞。而很多样例代码，甚至是Go Programing Language这本书中的引用，都通过会泄漏的，用了就忘的goroutine，全局变量和一些基本代码审查都不会通过的模式来解释并发。

我的同事所写的大部分goroutine都是吗，定义好的并发算法中，不主要的几个步骤。它们被用于构造和管理长期任务，但是又不明确何时会关闭。我认为这些例子需要更加强力的约束。

想象一下，go的关键字有所不同，我们必须要提供一个停止方案才能启动一个goroutine。事实上，我们正是要这么做。

这也是我再 [run](https://github.com/oklog/run)中，坚持做的事情，以下是其中最重要的方法：

```go
func (g *Group) Add(execute func() error, interrupt func(error))
```

Add 会运行一个goroutine，同事也会跟踪goroutine，在必要时中断它。比如我经常一个例子，我需要启动几个永远执行的goroutine方法，然后添加个goroutine来监听Ctrl-C, 然后关闭所有东西。

```go
ctx, cancel := context.WithCancel(context.Background())
g.Add(func() error {
    c := make(chan os.Signal, 1)
    signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
    select {
    case sig := <-c:
        return fmt.Errorf("received signal %s", sig)
    case <-ctx.Done():
        return ctx.Err()
    }
}, func(error) {
    cancel()
})
```

### Observability 观测

在讨论挂观测之前，我们来说说工业级代码的特殊性：我们写的代码会运行在服务器上，受理各种请求，不会被中断。这与交付客户的代码，和客户无感知的代码，是有很大的区别的。

我基本同意Charity一开始说的，我认同在分布式系统中，没有什么经济有效的方式进行全面的集成测试和冒烟测试。对于很多场景来说，好的观测比好的测试更重要，好的观测让我们可以快速部署和回滚，在有限的时间内恢复。

问题是，一个可观测的系统是怎样的？我觉得没有唯一的答案，也没有一个包可以一举解决这个问题。很多人跃跃欲试，但是花落谁家还未可知。当我们等待一个最佳的解决方案的时候，我们可以做些什么呢？



![](https://peter.bourgon.org/go-for-industrial-programming/venn.png)

Metrics, logging, tracing 都是可以观测数据，告知了我们生成，运输，和消费信号的方式。

todo 这篇文章的生僻东西太多，先不看了