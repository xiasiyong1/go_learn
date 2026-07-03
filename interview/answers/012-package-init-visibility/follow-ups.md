# 012. 包、可见性和 init - 面试追问

## 1. 包外可见性为什么只看首字母？结构体字段也一样吗？

Go 用标识符首字母决定是否导出。大写导出，小写不导出。结构体字段、函数、类型、变量、常量、方法都一样。

```go
package user

type User struct {
	ID   int64
	name string
}

func New(id int64, name string) User {
	return User{ID: id, name: name}
}

func (u User) Name() string {
	return u.name
}
```

包外代码只能访问导出的 `ID` 和 `Name`：

```go
u := user.New(1, "Tom")
fmt.Println(u.ID)
fmt.Println(u.Name())

// fmt.Println(u.name) // 编译错误
```

面试里可以补一句：未导出字段常用于保护不变量，调用方只能通过方法访问，包内部仍然可以自由调整实现。

## 2. 包初始化顺序具体是什么？

顺序是：依赖包先初始化，当前包后初始化；同一个包内先初始化包级变量，再执行 `init`。

```go
package a

import "fmt"

var X = initX()

func initX() int {
	fmt.Println("a var")
	return 1
}

func init() {
	fmt.Println("a init")
}
```

```go
package main

import (
	"fmt"
	"example.com/app/a"
)

var Y = initY()

func initY() int {
	fmt.Println("main var")
	return 2
}

func init() {
	fmt.Println("main init", a.X)
}

func main() {
	fmt.Println("main")
}
```

输出顺序会是：

```text
a var
a init
main var
main init 1
main
```

核心规则：`main.main` 是最后才开始执行的。

## 3. 一个包里多个 `init` 有什么限制和风险？

限制：

- 不能显式调用。
- 不能带参数。
- 不能返回值。
- 一个包里可以有多个。

```go
func init() {
	register("json")
}

func init() {
	register("xml")
}
```

风险是初始化逻辑分散，读代码时很难看出导入一个包到底发生了什么。

如果初始化可能失败，`init` 也不适合：

```go
func init() {
	if err := loadConfig(); err != nil {
		panic(err)
	}
}
```

更好的方式是显式返回错误：

```go
func NewApp(path string) (*App, error) {
	cfg, err := LoadConfig(path)
	if err != nil {
		return nil, err
	}
	return &App{Config: cfg}, nil
}
```

## 4. 空白导入 `_` 的真实用途是什么？

空白导入表示只需要这个包的初始化副作用，不直接引用它的名字。

```go
import (
	"database/sql"

	_ "github.com/lib/pq"
)

db, err := sql.Open("postgres", dsn)
```

`github.com/lib/pq` 被导入后，它的 `init` 会注册 postgres 驱动。业务代码调用的是 `database/sql`，所以没有直接使用 `pq` 包名。

如果忘记空白导入，可能会出现运行时错误：

```text
sql: unknown driver "postgres"
```

面试回答要强调：`_` 不是“消除未使用 import 报错”的技巧，而是在声明“我需要这个包的副作用”。

## 5. 为什么复杂业务初始化不建议放进 `init`？

因为 `init` 没有参数和返回值，导入包就自动执行，测试和错误处理都不清晰。

不推荐：

```go
var db *sql.DB

func init() {
	var err error
	db, err = sql.Open("mysql", os.Getenv("DSN"))
	if err != nil {
		panic(err)
	}
}
```

问题：

- 配置来源被隐藏。
- 测试很难注入临时数据库或 fake。
- 初始化失败只能 panic。
- 只要导入包就产生连接副作用。

推荐：

```go
type Store struct {
	db *sql.DB
}

func NewStore(dsn string) (*Store, error) {
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		return nil, err
	}
	return &Store{db: db}, nil
}
```

这样调用方能决定什么时候初始化、如何处理错误、测试时如何替换。

## 6. import cycle 出现时，通常应该怎么拆包？

循环依赖通常说明包边界不清楚。

错误结构：

```text
user   -> order
order  -> user
```

如果两个包只是共享基础类型，可以抽到更底层的包：

```text
domain -> 定义 User、Order、ID 等基础类型
user   -> 导入 domain
order  -> 导入 domain
```

如果是两个服务互相调用，通常应该让更高层负责编排：

```text
app    -> 导入 user 和 order，组合业务流程
user   -> 不导入 order
order  -> 不导入 user
```

如果只是为了使用一个小接口，可以把接口放在使用方一侧：

```go
package order

type UserGetter interface {
	GetUser(id int64) (User, error)
}
```

Go 里接口隐式实现，很多循环依赖都可以通过“接口放在消费方”来拆开。
