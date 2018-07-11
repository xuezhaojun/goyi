# Standard Package Layout 标准的包结构（部分选文）

[原文链接](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1)

> 原文第一部分介绍的是一般使用的包结构方法，第二部分则介绍的是作者推荐的包结构方法，本文仅包含第二部分

## A better approach - 更好的做法

我在我的项目中所使用的4个原则：

1. 根包用于放领域类型（root package）
2. 根据依赖来划分子包
3. 使用一个共享的mock包
4. main 包要整合所有的依赖



这些原则帮助我们解耦包，并且定义一个整个应用内清晰的领域语言。让我们看看具体每一条怎么用。

### #1. Root package is for demain types

你的应用会有一个逻辑的，高层的语言，用于解释数据和进程的交互。这就是你的领域（domain），如果你有一个电子交易应用，那么你的领域应该包括 顾客，账户，充值信用卡，同时要处理库存。如果你是facebook，你的领域就是用户，点赞，关系。这些事和你底层技术实现无关的东西。

我将我的领域类型放在我的根包（root package）下。这个包仅包含简单的数据类型，例如保存用户对象的User结构体，或者UserService这样用于存取用户数据的接口。

看起来就和下面差不多：

```go
package myapp

type User struct {
	ID      int
	Name    string
	Address Address
}

type UserService interface {
	User(id int) (*User, error)
	Users() ([]*User, error)
	CreateUser(u *User) error
	DeleteUser(id int) error
}
```

这会让你的root package非常极简。	你也可以在放一些提供方法的类型，但仅限他们完全依赖于你的领域类型的情况下。举个例子：你可以有一个定期轮询你的UserService的类型。但是，它不应该调用外部服务，或者保存东西到数据库，因为这已经涉及到实现细节了，而root package关系的是domain。

### #2. 根据依赖分子包

在 root package 不能有外部依赖的前提下，我们要将这些依赖放到自包内。按我的包结构方案，子包subpackages 是作为你的领域和你的实现之间的适配器存在的。

例如，你的UserService 可能是 PostgreSQL 支持的。你可以通过向你的应用引入一个 postgres 子包，来提供一个 postgres.UserService 的具体实现：

```go

package postgres

import (
	"database/sql"

	"github.com/benbjohnson/myapp"
	_ "github.com/lib/pq"
)

// UserService represents a PostgreSQL implementation of myapp.UserService.
type UserService struct {
	DB *sql.DB
}

// User returns a user for a given id.
func (s *UserService) User(id int) (*myapp.User, error) {
	var u myapp.User
	row := db.QueryRow(`SELECT id, name FROM users WHERE id = $1`, id)
	if row.Scan(&u.ID, &u.Name); err != nil {
		return nil, err
	}
	return &u, nil
}

// implement remaining myapp.UserService interface...
```

这会把我们对PostgreSQL的依赖独立出来，简化了测试，也让未来迁移到其他的数据库更加的方便。如果你想添加例如 BoltDB一类的，其他数据库实现，这种可拔插的结构正可以满足。

这样也正好让你有办法进行 分层实现 layer implementations。也许你想在 PostgreSQL 前加一个内存的，LRU缓存。那么你可以添加一个实现了UserService的UserCache，把你的PostgreSQL实现包起来。

```go
package myapp

// UserCache wraps a UserService to provide an in-memory cache.
type UserCache struct {
        cache   map[int]*User
        service UserService
}

// NewUserCache returns a new read-through cache for service.
func NewUserCache(service UserService) *UserCache {
        return &UserCache{
                cache: make(map[int]*User),
                service: service,
        }
}

// User returns a user for a given id.
// Returns the cached instance if available.
func (c *UserCache) User(id int) (*User, error) {
	// Check the local cache first.
        if u := c.cache[id]]; u != nil {
                return u, nil
        }

	// Otherwise fetch from the underlying service.
        u, err := c.service.User(id)
        if err != nil {
        	return nil, err
        } else if u != nil {
        	c.cache[id] = u
        }
        return u, err
}
```

在标准库种，这个设计也有。比如 io.Reader 是一个读字节的领域类型，而它的实现则根据依赖分包： tar.Reader， gzip.Reader，multipart.Reader。这些也可以是分层的，比如常见的 os.File 被 bufio.Reader包住，bufio.Reader 被 gzip.Reader 包住，而gzip.Reader则被 tar.Reader 包住。

#### 依赖和依赖之间

你的依赖并不是唯一，有时候你存储了User类型的数据到PostgreSQL中，但是你的金融交易数据则存放在另一个第三方服务中，比如 [Stripe](https://stripe.com/) 中。在这个案例中，我们将Stripe的依赖用一个逻辑领域类型 logical domain type 包起来  -   我们就叫这个类型 TransactionService （交易服务）

通过给 UserService 添加 TransactionService，我们整合了两种依赖。

```go
type UserService struct {
        DB *sql.DB
        TransactionService myapp.TransactionService
}
```

 此时，我们的依赖可以完全通过我们共有的领域语言来交流了。这也意味着我们可以在PostgreSQL替换为MySQL或者将Stripe替换为其他第三方的同时，不影响其他依赖。（这里的myapp.TransactionService是一个interface,和myapp.UserService是一样的，只定义领域语言，不涉及具体实现）

#### 不关针对第三方的包

可能听起来相当怪，但是我对于标准库的依赖，也会用同样的方法处理。比如 net/http 这个包就是一个依赖。我们可以通过添加一个 http 的子包来独立它。

可能有一个和依赖的包名字一样的包看起来很怪，但是这是故意而为之的。一旦你允许你的应用除了http子包外的其他包使用net/http，就会出现名字冲突。重名的好处在于你可以将所有的http代码都放在http子包种。

```go
package http

import (
        "net/http"
        
        "github.com/benbjohnson/myapp"
)

type Handler struct {
        UserService myapp.UserService
}

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
        // handle request
}
```

此时，你的 http.Handler 是作为一个领域和http协议之间的适配器。

### #3. User a shared mock subpackage

