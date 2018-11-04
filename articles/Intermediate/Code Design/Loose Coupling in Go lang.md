# Loose Coupling in Go lang

> golang中做到松耦合

[原文地址](https://8thlight.com/blog/javier-saldana/2015/02/06/loose-coupling-in-go-lang.html)

Golang提供了很多让我们实现松耦合设计的特性。这篇blog中，我将会介绍两个达成松耦合的方法 [interfaces](https://8thlight.com/blog/javier-saldana/2015/02/06/loose-coupling-in-go-lang.html#interfaces)  和 [higher-order functions](https://8thlight.com/blog/javier-saldana/2015/02/06/loose-coupling-in-go-lang.html#higher-order-functions) 。

### 1. Interfaces

在 Go 中，我们可以通过定义一个方法集来定义一个interface：

```go
type /* name of interface */ interface {
     // method definitions, that is without implementation
     // e.g. sum(a, b int) int
}
```

假设我们再完成一个象棋程序，我们希望定义一个interface来代表在多个I/O设备下的游戏。我们需要一个方法从用户那里获取到input输入，然后另一个方法代表输出：

```go
type IO interface {
    Read() string
    Write(message string)
}
```

当我们调用`Read`时，我们希望能获取到一个字符串，而当调用`Write`时，我们传入一条信息。在Go中，任何有了这两个方法的值都隐式的实现了接口。比如：

```go
type CommandLine struct{}

func (cli CommandLine) Read() string {
    var input string

    fmt.Scanf("%s\n", &input)

    return input
}

func (cli CommandLine) Write(output string) {
    fmt.Printf("%v", output)
}
```

这对于测试来说非常有效。我们可以创建一个FakeIO（假IO）,并且拥有对其input和output的完全控制，用于校验输入输出的内容。

```go
type FakeIO struct {
    Input string
    Output string
}

func (io FakeIO) Read() string {
    return io.Input
}

func (io FakeIO) Write(output string) {
    io.Output = output
}
```

现在我们可以在**测试**中替换对I/O的调用。假设我们想要玩家在我们开始游戏的时候输入他的名字。

```go
func (player Player) GetName(io IO) {
     io.Write("What's your name?")
     player.Name = io.Read()
}

func TestGetPlayerName(t *testing.T) {
    var io FakeIO

    io.Input = "Javier"
    player.GetName(io)

    if name := player.Name; name != "Javier" {
        t.Errorf("Expected name to be %#v, but was %#v",
            "Javier", name)
    }
}
```

同时这样让检查游戏的output变得简单，我们可以通过检查`io.Output`的值来直接判断。

接口创建了我们的程序需要的通用信息集。只要数据能对的上接口，我们就可以安全的替换系统内的各个元素。在这个列子中，如果我们想创建一个GUI替换掉命令行，只要使用一个具有 `Read()` 和 `Write()` 方法的GUI类型就可以，我们的游戏部分代码无需改变，而且在Go里实现interface可是相当的简单。

### 2. Higher-order functions

> 在数学和计算机领域，一个高阶函数是一个至少含有一下两个中一个特性的函数：
>
> 1. 将一个或者多个function作为input
>
> 2. 将一个function作为output

Go 是静态类型的，也就是说可以将函数的使用作为依赖注入。还拿我们的象棋游戏说事，考虑与电脑对战的问题。

当一个人类玩家的回合，我们必须 将 象棋棋盘 和 IO interface 传入，来获取移动指令的输入，有可能是GUI的一次鼠标点击，也有可能是命令行的一次输入：

```go
player.Move(io, chessBoard)
```

而当电脑回合时，我们就不再需要IOinterface了，它可以基于当前的状态自己计算出下一步怎么走。

```go
computer.Move(chessBoard)
```

这样游戏循环可能会看起来如下:

```go
for !chessBoard.HasCheckMate() {
    player.Move(io, chessBoard)
    computer.Move(chessBoard)
}
```

你可能注意到了，上面代码的问题是，我们没在电脑移动之前检验游戏状态。如果人类已经达成胜利条件，电脑还是会尝试移动。我们可以通过一个死循环来修正：

```go
for {
    if !chessBoard.HasCheckMate() {
        player.Move(io, chessBoard)
    }

    if !chessBoard.HasCheckMate() {
        computer.Move(chessBoard)
    }
}
```

但是我一点都不想要代码重复，（考虑到我们之后可能还会添加第三用户，第四用户等）

如果将玩家（不分人类还是电脑）都可以保存在player结构体中，然后每轮循环获取玩家。那么我们的代码就可以写成以下这样：

```go
for !chessBoard.HasCheckMate() {
    getCurrentPlayer().Move(...)
}
```

仅保存玩家的引用是不够的，毕竟，每一类玩家的`Move`方法的参数都不一样。如果我们真的想上述代码成功运行，我们可以尝试几种策略，但是我（作者）不会在这个blog中介绍。我会使用go的匿名方法来存储`Move`,而非player:

```go
var playerMove = func(chessBoard chess.Board) {
    player.Move(io, chessBoard)
}

var computerMove = func(chessBoard chess.Board) {
    computer.Move(chessBoard)
}

var moves = []func(chessBoard chess.Board) {
    playerMove,
    computerMove,
}
```

匿名函数值接受chessBoard作为参数，不同类型玩家的`Move`可以保存于同一个数组中：

最终，我们的游戏循环会变成这样：

```go
for !chessBoard.HasCheckMate() {
    moveCurrentPlayer(chessBoard)
    changeTurn()
}
```

然后调用当前玩家的`move`会相当简答：

```go
func moveCurrentPlayer(chessBoard chess.Board) {
    moves[0](chessBoard)
}
```

要更新玩家轮次，只要循环一下 `moves`的内容即可。

高阶函数让我们可以将不同的接口整合为相同的格式。现在我们可以在不改变游戏逻辑的情况下，添加更多的玩家了，`Move`方法的参数差别不再成问题，因为我们用匿名方法将它们包住了。

最后要说，Go真清爽，开箱即用，简单强大。基础包有很多功能，无需重复造轮子，Go将functions作为一等公民对待，这是我目前最喜欢的特性！