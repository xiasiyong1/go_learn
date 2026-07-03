# 012. 包、可见性和 init

## 问题

Go 的包导入、可见性和 init 执行顺序是怎样的？

## 先给结论

Go 用包组织代码。一个包的公开 API 由导出标识符决定：首字母大写表示包外可见，首字母小写表示只在包内可见。

`init` 是包初始化阶段自动执行的函数。初始化顺序可以记成：

1. 先初始化被导入的包。
2. 再初始化当前包的包级变量。
3. 最后执行当前包的 `init` 函数。
4. `main` 包初始化完成后，才执行 `main.main`。

`init` 适合做少量注册类副作用，不适合放复杂业务启动逻辑。

## 包名、目录名和导入路径

Go 代码文件开头必须声明包名：

```go
package user
```

导入时使用的是模块路径加目录路径：

```go
import "example.com/app/internal/user"
```

包名和目录名通常保持一致，这样读代码最省力：

```text
internal/user     -> package user
internal/order    -> package order
```

不推荐随意不一致：

```text
internal/user     -> package account
```

虽然语法允许，但会让调用方困惑：

```go
import "example.com/app/internal/user"

account.Create()
```

## 可见性：看标识符首字母

Go 没有 `public`、`private` 关键字。是否导出看首字母。

```go
package user

type User struct {
	ID   int64  // 导出字段，包外可访问
	name string // 未导出字段，只能包内访问
}

func New(id int64, name string) User {
	return User{ID: id, name: name}
}

func (u User) Name() string {
	return u.name
}
```

包外可以这样用：

```go
u := user.New(1, "Tom")
fmt.Println(u.ID)
fmt.Println(u.Name())

// fmt.Println(u.name) // 编译错误
```

这个设计可以隐藏内部字段，后续即使改存储方式，也不一定影响调用方。

## 包级变量和 init 的顺序

同一个包里，包级变量会先初始化，然后执行 `init`。

```go
package config

import "fmt"

var defaultPort = initDefaultPort()

func initDefaultPort() int {
	fmt.Println("var init")
	return 8080
}

func init() {
	fmt.Println("init", defaultPort)
}
```

输出顺序一定是：

```text
var init
init 8080
```

如果有依赖包：

```go
package main

import (
	_ "example.com/app/driver"
	"example.com/app/server"
)

func main() {
	server.Run()
}
```

执行顺序是：先初始化 `driver`、`server` 及它们各自依赖的包，再初始化 `main`，最后进入 `main.main`。

## 一个包可以有多个 init

```go
package registry

func init() {
	register("json")
}

func init() {
	register("xml")
}
```

`init` 的限制：

- 不能被显式调用。
- 不能有参数。
- 不能有返回值。
- 一个文件里可以有多个，一个包里也可以有多个。

但是多个 `init` 会让初始化逻辑分散。除非是很小的注册逻辑，否则更推荐显式函数。

## 空白导入用于副作用

空白导入会导入包并执行它的初始化逻辑，但不直接使用它的导出标识符。

典型例子是数据库驱动注册：

```go
import (
	"database/sql"

	_ "github.com/lib/pq"
)

func Open() (*sql.DB, error) {
	return sql.Open("postgres", "postgres://...")
}
```

`github.com/lib/pq` 的 `init` 会把驱动注册到 `database/sql`。业务代码没有直接引用 `pq` 包，所以要用 `_` 表示“我就是为了副作用导入它”。

空白导入要谨慎。代码 review 时应该能回答：

- 这个包的副作用是什么？
- 去掉它会影响什么？
- 有没有更显式的注册方式？

## import cycle 为什么被禁止

Go 禁止循环导入：

```text
user -> order -> user
```

因为包是编译和初始化单元。循环依赖会让编译顺序、初始化顺序和抽象边界都变得不清楚。

常见错误设计：

```text
service/user   导入 service/order
service/order  导入 service/user
```

更好的方向通常是抽出更底层的包：

```text
domain        放 User、Order 等基础类型
service/user  导入 domain
service/order 导入 domain
```

或者让上层组合依赖，而不是两个底层包互相调用。

## init 适合什么，不适合什么

适合：

```go
func init() {
	image.RegisterFormat("png", "\x89PNG\r\n\x1a\n", Decode, DecodeConfig)
}
```

这种注册逻辑小、确定、失败概率低，且本来就是导入即生效。

不适合：

```go
func init() {
	db, err := sql.Open("postgres", os.Getenv("DSN"))
	if err != nil {
		panic(err)
	}
	globalDB = db
}
```

问题很多：

- 错误只能 panic，不能正常返回给调用方。
- 测试很难替换配置。
- 导入包就产生外部连接，副作用太重。
- 初始化顺序不明显。

更好的写法是显式构造：

```go
func NewStore(dsn string) (*Store, error) {
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return nil, err
	}
	return &Store{db: db}, nil
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 包外可见性为什么只看首字母？结构体字段也一样吗？
- 包初始化顺序具体是什么？
- 一个包里多个 `init` 有什么限制和风险？
- 空白导入 `_` 的真实用途是什么？
- 为什么复杂业务初始化不建议放进 `init`？
- import cycle 出现时，通常应该怎么拆包？
