# Go Proverbs - Go 箴言

[原文链接](https://go-proverbs.github.io/) 原文来自于 Rob_Pike 的  [talk at Gopherfest SV 2015 (video)](https://www.youtube.com/watch?v=PAAkCSZUG1c) 

# Simple, Poetic,Pithy 

>  简单，诗意，精练

### Don't communicate by sharing memory,share memory by communicating.

不要通过分享内存来通信，要通过通信来分享内存

### Concurrency is not parallelism.（concurrency is better）

并发非并行

go最为人称道的，就是它是为并发设计的。不包含goroutines和channels的介绍都是不完整的。

但是当人们听到**并发**(concurrency)这个单词的时，他们往往想的是**并行**(parallelism),一个和并发相关，但是非常不同的概念。

在编程中，**并发指多个进程独立执行中，而并行是指同时执行的（可能具有相关性的）计算**。

**并发**是指 同时**处理**很多东西，**并行**是指同时**执行**很多东西。

[以上来自](https://blog.golang.org/concurrency-is-not-parallelisms)

并发的目的是更好的结构，而不是并行。

实现了并发，并不一实现并行（并行是与处理器有关的，只有一个处理器，那永远都没法并行）

### Channels orchestrate; mutexes serialize.

mutex粒度很细，很小，趋向于序列化执行。给一个变量加上mutex，也就意味着同时只有一个操作可以在这个变量上执行。

而channels和goroutines则是在更大的蓝图上编排的，是粗粒度的。

### The bigger the interface, the weaker the abstraction.

接口越大，抽象越弱

### Make the zero value userful.

让0值有意义

### interface{} says nothing.

empty interface 不表示任何东西，当你写出这种空接口的，一定要好好想想，能不能用接口代替

### Gofmt's style is no one's favorite, yet gofmt is everyone's favorite.

你可以有自己的代码风格，但是最后记得运行 gofmt (偷笑)

### A little copying is better than a little dependency.

如果copy几行代码就行，那就别非要引入库啦

### Syscall must always be guarded with build tags

系统调用必须始终记得使用构建标记

### Cgo must always be guarded with build tags.

Cgo 必须使用构建标记

### Cgo is not go.

cgo 不是 go

### With the unsafe package there are no guarantees.

使用unsafe这个包，没有任何保证

### Clear is better than Clever

清楚比聪明重要

### Reflection is never clear

反射不是给你准备的（再次偷笑），新手就应该原理空接口，原理反射

### Errors are values

错误也是一种值

### Don't just check errors, handle them gracefully.

不要光检查错误，你可以做的更多

### Design the architecture, name the components, document the details.

设计架构，好好起组件名，把细节写清楚

### Documentation is for users.

文档是面向用户的

### Don't Panic

不要panic