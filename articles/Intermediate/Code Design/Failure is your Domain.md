# Failure is your Domain

[原文链接](https://middlemost.com/failure-is-your-domain/)

> 错误也属于你的应用领域

错误处理是Go这门语言的核心，但同时Go并没有规定怎么做错误处理，真实悖论啊。虽然社区有提供一些提高和标准化错误错误的方式，但是大多都在应用领域这一块缺失了。也就是说，其实你的error本应该和`Customer`和`Order`一样的类型。

应用类型，终端用户，操作者 - 形形色色的角色要求有不同的error。本文探讨了对于不同类型的消费角色对应的不同目的的error，以及我们如何能简单高效的满足不同角色的需求。

本文包含了很多[Standard Package Layout](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)一文中，关于应用领域&项目设计内容，可以先读一下那个。

### Why We Err

Errors,核心就是，一种简单的，告诉你为什么事情发展和预期不符的方式。Go将errors分为两份部分，`panic`和`error`。一个`panic`在你无法预料的情况发生，比如访问错误的内存。典型的，一个panic对于我们的应用来说是不可恢复的，灾难性的，我们只是简单的通知操作者来修复以下bug。

相对而言，一个`error`,则是在我们能预期有些东西除了问题，这也是本文的关注点。

### Types of errors

我们可以将error划分为两类，定义好的error & 没有定义的error。

一个定义好的error，具有特别的API，比如从`os.Open()` 中返回的 `os.IsNotExit()` error, 这些错误让我们可以在已知预期的情况下，管理我们的应用流。

一个未定义的错误，是无API的，我们也没有办法深度的处理它。这种情况有可能是文档太简陋导致的，也有可能是在我们集成之后，APIs有新增了一些错误条件。

### Who Consumes Our Errors

关于error最神奇的部分是，对于不同的consumer，它们也应该是不同的。在任何给定的系统中，我们至少都有三种consumer角色 --  应用（application），终端用户(end user)，操作者(operator)

#### Application role

你的应用本身就是你错误处理的第一道防线，你的应用代码可以很快的从error状态恢复，并且不用半夜给任何人打电话。但是应用的错误处理最不灵活，并且只能处理定义好的错误状态。

举例，你的浏览器收到了一个`301`重定向码，同时将你导航到一个新的地址。这对于大部分用户来说都是无缝的操作，它能做到这样是应为HTTP对于错误码具有非常明确的定义。

#### End user role

如果你的应用没有办法处理这个error，那么最好你的终端用户可以收到这个问题，你的终端用户可以看到错误状态类似于`your debit card is declined（你的信用卡超支了）`，而且可以非常灵活的处理这个问题。

不像application角色，终端用户需要的是 人类易读 的信息，可以帮助他们处理问题。

这些用户仍然限制于定义好的错误，因为解释未定义的错误有可能会让危及你系统的安全性。比如：一个`postgres`error 可能会展示处 查询 和 表信息的 的细节，有些攻击者可能会利用这些信息。当面对未定义的error，最好简单的告诉用户和技术支持联系即可。

#### Operator role

最终，也是最后的防线，及时系统的操作者 - 开发或者运维。这些人非常理解系统，可以处理任何类型的错误。

作为这类角色，你一般希望能看到愈多的信息越好。除了错误码和人类可读的信息，一个逻辑堆栈跟踪可以帮助操作者理解程序流程。

### Our Baseline Application Error Type

我们需要：

* 错误码 - error code - application 
* 可读信息 - human-readble message - end user
* 堆栈信息 - logic stack trace - operator

建立在这样的基础上，我们就可以建立简单的错误类型来处理大部分的 application error了。

```go
package myapp

// Error defines a standard application error.
type Error struct {
	// Machine-readable error code.
	Code    string

	// Human-readable message.
	Message string

	// Logical operation and nested error.
	Op      string
	Err     error
}
```

注意：这个方案是在[Error handling in Upspin](https://commandcenter.blogspot.com/2017/12/error-handling-in-upspin.html)一文中衍生出来的，做了一些改进，我们之后会讨论。

这是我们的基础错误类型。`Code` he `Message` 字段是为我们的application 和 user 提供的，相对的，`OP`和 `Err` 字段允许我们将error 链化，这样我们可以为我们的operator提供堆栈跟踪的信息。

一个容易被忽略的重要细节是，我们的error是定义在root package下的 -- `myapp.Error`。这一点非常重要，因为它会成为我们domain language(领域语言)的一部分。这样避免了表达上的重复问题 `比如：errors.Error`

每一个包都是唯一的，所以这个基本类型还需要根据你的业务和场景来使用。你可能还需要为定义一些特殊的错误类型，但是对于大多数的error的使用场景来说是足够的。

### Error Management by Role

我们的`Error`类型为我们开了一个好头，但是我们需要添加一些简单功能，来让它可用。让我们考虑几个不同橘色的不同场景。在以下的例子中，我们假设一个应用 `myaqq`, 在其领域内定义了一个 `UserService` 接口。

```go
package myapp

// UserService represents a service for managing users.
type UserService interface {
	// Returns a user by ID.
	FindUserByID(ctx context.Context, id int) (*User, error)

	// Creates a new user.
	CreateUser(ctx context.Context, user *User) error
}
```

让我们看看每一个角色是怎么管理它的错误状态的。

#### Application role

我们的应用角色一般只关心简单的错误码。比如我们的程序试图通过 ID 获取到 `User`,然后它收到了一个`not found` 错误，它就可以使用email地址再次做请求。

##### 定义我们的错误码

虽然定义细粒度的错误码很诱人，但是通用的错误码更容易管理。有好几个定义通用错误码的系统的代码可以借鉴，我个人最喜欢的是 Http 和 gRPC

* [HTTP/1.1: Status Code Definitions](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)
* [gRPC Canonical Error Codes](https://github.com/grpc/grpc-go/blob/v1.12.0/codes/codes.go)

它们都包含的错误码数量众多，我则从更加简单的代码开始，然后根据需求扩展。我发现以下这些是一个不错的起点：

```##go
// Application error codes.
const (
	ECONFLICT   = "conflict"   // action cannot be performed
	EINTERNAL   = "internal"   // internal error
	EINVALID    = "invalid"    // validation failed
	ENOTFOUND   = "not_found"  // entity does not exist
)
```

##### 将错误码翻译到我们的领域domain

这些错误码对于我们的应用来说都是特定的，所以当我们与外部库交互的时候，我们必须将这些errors转到为我们的领域错误码。比如，我们的application实现了我们的`UserService`，我们将需要将`sql.ErrNoRows` 转化为一个 `ENOTFOUND`码。如果不做这样的翻译，我们的application如果同时也用 non-SQL 数据库实现了，就无法识别 `sql.ErrNoRows` 然后会奔溃。

```go
package postgres

// FindUserByID returns a user by ID. Returns ENOTFOUND if user does not exist.
func (s *UserService) FindUserByID(ctx context.Context, id int) (*myapp.User, error) {
	var user myapp.User
	if err := s.QueryRowContext(ctx, `
		SELECT id, username
		FROM users
		WHERE id = $1
	`,
		id
	).Scan(
		&user.ID,
		&user.Username,
	); err == sql.ErrNoRows {
		return nil, &myapp.Error{Code: myapp.ENOTFOUND}
	} else if err {
		return nil, err
	}
	return &user, nil
}
```

这么做可以让我们给我们的application返回一个与`UserService`如何实现无关的错误类型。

##### 更高效的使用错误码

要做到这一点，我们有两个问题。第一，我们的function返回的是`error`而不适`*myapp.Error`，所以我们需要在需要访问`Error.Code`的时候，做类型断言。这让人很烦。第二，我们的`Error.Err`字段允许我们回溯errors，所以我们最顶层的error可能没有包含错误码，需要我们递归的查找它。

我们可以通过一个简单的function来解决这两个问题：

1. `nil`时，不返回任何error code
2. 查找 error 链，直到找到定义 `Code`的地方
3. 如果没有任何code被定义，就返回一个internal error code（`EINTERNAL`）

以下则是实现：

```go
// ErrorCode returns the code of the root error, if available. Otherwise returns EINTERNAL.
func ErrorCode(err error) string {
	if err == nil {
		return ""
	} else if e, ok := err.(*Error); ok && e.Code != "" {
		return e.Code
	} else if ok && e.Err != nil {
		return ErrorCode(e.Err)
	}
	return EINTERNAL
}
```

现在我们调用代码的就是以下这个样子：

```go
user, err := userService.FindUserByID(ctx, 100)
if myapp.ErrorCode(err) == myapp.ENOTFOUND {
	// retry another method of finding our user
} else if err != nil {
	return err
}
```

### End user role

我们的终端用户期望可行的，可读的信息。可能还包含一些其他的问题，比如国际化等，但是我们先只关心基础的东西。

##### 例子使用

字段验证时用户终端信息的一个完美例子。我们这里检查`UserService`中的新用户，是否具有新的用户名，并且唯一。

```go
package postgres

// CreateUser creates a new user in the system.
// Returns EINVALID if the username is blank or already exists.
// Returns ECONFLICT if the username is already in use.
func (s *UserService) CreateUser(ctx context.Context, user *myapp.User) error {
	// Validate username is non-blank.
	if user.Username == "" {
		return &myapp.Error{Code: myapp.EINVALID, Message: "Username is required."}
	}

	// Verify user does not already exist.
	if s.usernameInUse(user.Username) {
		return &myapp.Error{
			Code: myapp.ECONFLICT,
			Message: "Username is already in use. Please choose a different username.",
		}
	}

	...
}
```

##### 高效的利用错误信息

和之前说的错误码问题类似，我们需要一个工具型的方法，我们可以实现一个类似于`ErrorCode`,并满足以下条件的方法：

1. nil 返回空
2. 查找Error.Err 链直到找到Message
3. 如果没有定义message，返回一个通用的error信息

以下是实现：

```go
// ErrorMessage returns the human-readable message of the error, if available.
// Otherwise returns a generic error message.
func ErrorMessage(err error) string {
	if err == nil {
		return ""
	} else if e, ok := err.(*Error); ok && e.Message != "" {
		return e.Message
	} else if ok && e.Err != nil {
		return ErrorMessage(e.Err)
	}
	return "An internal error has occurred. Please contact technical support."
}
```

现在我们的user可以看到的信息为:

```go
if msg := ErrorMessage(err); msg != "" {
	fmt.Printf("ERROR: %s\n", msg)
}
```

### Operator role

最终，我们需要讨论如何将这一切信息附加在一个堆栈trace上，给我们的operator来调式问题。Go已经提供了一个类似的方法，`error.Error()` ，来打印和error相关的信息。

##### Logical stack traces

很多operator都对堆栈跟踪很熟悉。他们会打印出错误发生时，调用栈中每一个方法。你可以在`panic`的时候看到。

但是很多时候，堆栈跟踪 都太过头了，我们只需要里面的一小部分就可以理解error产生的上下文。堆栈跟踪只包括我们作为开发者，认为描述程序流最重要的那一层。我们将通过`Op`和`Err`字段将errors包起来提供上下文。

##### 实现 Error()

我们的 `myapp.Error.Error()` 方法，应该为operator返回一个错误字符串。虽然没有特定的格式，但是我有以下建议：

1. 首先显示堆栈跟踪的信息，它会提供剩余信息的上下文，它让我们可以找出error的行数。
2. 在最后显示 `Code` 和 `Message`
3. 只打印一行，这样grep起来容易

```go
// Error returns the string representation of the error message.
func (e *Error) Error() string {
	var buf bytes.Buffer

	// Print the current operation in our stack, if any.
	if e.Op != "" {
		fmt.Fprintf(&buf, "%s: ", e.Op)
	}

	// If wrapping an error, print its Error() message.
	// Otherwise print the error code & message.
	if e.Err != nil {
		buf.WriteString(e.Err.Error())
	} else {
		if e.Code != "" {
			fmt.Fprintf(&buf, "<%s> ", e.Code)
		}
		buf.WriteString(e.Message)
	}
	return buf.String()
}
```

这个实现假定了 `Err` 不能和 `Code`或`Message`并存，我发现这么做在实际中很有用。

我们看一下下面的例子：

##### 举例

回头看我们的`CreateUser()`方法，假设我们要为我们的用户创建一些额外的角色。我们可以利用`Op`和`Error`字段来讲嵌套包起来。

```go
// CreateUser creates a new user in the system with a default role.
func (s *UserService) CreateUser(ctx context.Context, user *myapp.User) error {
	const op = "UserService.CreateUser"

	// Perform validation...

	// Insert user record.
	if err := s.insertUser(ctx, user); err != nil {
		return &myapp.Error{Op: op, Err: err}
	}

	// Insert default role.
	if err := s.attachRole(ctx, user.ID, "default"); err != nil {
		return &myapp.Error{Op: op, Err: err}
	}
	return nil
}
```

```go
// insertUser inserts the user into the database.
func (s *UserService) insertUser(ctx context.Context, user *myapp.User) error {
	const op = "insertUser"
	if _, err := s.db.Exec(`INSERT INTO users...`); err != nil {
		return &myapp.Error{Op: op, Err: err}
	}
	return nil
}
```

```go
// attachRole inserts a role record for a user in the database
func (s *UserService) attachRole(ctx context.Context, id int, role string) error {
	const op = "attachRole"
	if _, err := s.db.Exec(`INSERT roles...`); err != nil {
		return &myapp.Error{Op: op, Err: err}
	}
	return nil
}
```

我们假设在我们的attachRole()中，收到了Postgres的语法错误，如下：

```go
syntax error at or near "INSERT"
```

没有上下文，我们不会直到这个错误实在我们的`insertUser()`方法还是`attachRole()`中发生的。

但是由于我们有打包错误信息，最终错误信息会展示如下：

```go
UserService.CreateUser: attachRole: syntax error at or near "INSERT"
```

这让我们更快的找到问题并修复。

### Comparison with other approaches

本文介绍了我是使用的错误处理方法，但是这不是唯一的，我想讨论以下其他的方案已经为什么有所区别，会很有帮助（这里仅节选讨论pkg/errors的部分）

##### pkg/errors

[pkg/errors](https://github.com/pkg/errors) 这个项目以一个库的方式，提供了 error 打包。这种处理error打包的方式很不错，而且会提供很多error需要的信息，但是我也发现了一些在使用上的问题：

* 通过第三方库来引入你的error处理，会将你的应用领域延伸
* error打包让开发者可以从根问题信息开始，但还是需要一些类型断言。但是本文`ErrorCode()`方法只要在调用的地方加一行简单的判断即可
* 它解决了错误打包问题，但是对于其他的一些方面并没有帮助

### Feedback

我指出问题不是让你们不用这些。没有什么方案可以适用于所有场景。我添加这些对比只是为了帮助开发者做出自己的选择。

### Conclusion

错误处理是你的应用设计中十分重要的一个部分，通常会因为不同角色的consumer变得复杂起来。通过使用错误码，错误信息，和堆栈信息，我们可以满足各种角色的需求。通过整合`Error`到我们的领域，我们让我们的应用的各个部分在问题出现的时候可以通过通用的语言进行沟通。