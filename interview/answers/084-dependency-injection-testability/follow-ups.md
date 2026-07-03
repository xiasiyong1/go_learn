# 084. 可测试性 - 面试追问

## 1. Go 为什么通常不需要复杂 DI 框架？

因为 Go 的依赖关系用结构体字段和构造函数就能表达清楚。

```go
type Service struct {
	repo   UserRepo
	mailer Mailer
}

func NewService(repo UserRepo, mailer Mailer) *Service {
	return &Service{repo: repo, mailer: mailer}
}
```

调用方一眼能看到 Service 依赖什么。复杂框架可能把依赖隐藏在运行时容器里，反而不利于初学者理解和排查。

大型项目可以使用生成式工具减少装配样板，但核心仍然是显式依赖关系，而不是魔法注入。

## 2. 接口应该定义在生产者还是使用者？

多数情况下定义在使用者侧。

使用者只需要一个方法：

```go
type UserFinder interface {
	FindUser(ctx context.Context, id string) (User, error)
}
```

真实 repository 可以有很多方法：

```go
type MySQLUserRepo struct{}

func (r *MySQLUserRepo) FindUser(ctx context.Context, id string) (User, error) { return User{}, nil }
func (r *MySQLUserRepo) CreateUser(ctx context.Context, u User) error { return nil }
func (r *MySQLUserRepo) DeleteUser(ctx context.Context, id string) error { return nil }
```

使用者不应该被迫依赖 `CreateUser`、`DeleteUser`。接口越贴近使用者，测试 fake 越小。

## 3. 时间相关逻辑怎么测试？

不要在业务逻辑里直接调用 `time.Now()`，把 clock 注入进去。

```go
type Clock interface {
	Now() time.Time
}

func IsExpired(clock Clock, deadline time.Time) bool {
	return !clock.Now().Before(deadline)
}
```

测试：

```go
type fakeClock struct{ now time.Time }

func (f fakeClock) Now() time.Time { return f.now }

func TestIsExpired(t *testing.T) {
	now := time.Date(2026, 7, 3, 10, 0, 0, 0, time.UTC)
	deadline := now.Add(-time.Second)

	if !IsExpired(fakeClock{now: now}, deadline) {
		t.Fatal("want expired")
	}
}
```

这样测试不会受真实时间影响，也不需要 sleep。

## 4. 如何避免测试依赖真实 HTTP 或数据库？

HTTP 可以用接口或 `httptest`。

接口方式：

```go
type Doer interface {
	Do(*http.Request) (*http.Response, error)
}

type API struct {
	client Doer
}
```

`httptest` 方式：

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte(`{"ok":true}`))
}))
defer srv.Close()
```

数据库可以把 repository 抽成小接口，单元测试用 fake；集成测试再连真实数据库，并用 build tag 或单独测试环境隔离。

## 5. 什么时候不应该抽接口？

没有替换需求时，不要为了“看起来可测试”提前抽接口。

不必要：

```go
type Adder interface {
	Add(int, int) int
}

type Calculator struct{}

func (Calculator) Add(a, b int) int { return a + b }
```

直接测试具体类型更简单：

```go
func TestAdd(t *testing.T) {
	c := Calculator{}
	if got := c.Add(1, 2); got != 3 {
		t.Fatal(got)
	}
}
```

适合抽接口的场景：

- 网络、数据库、文件系统等外部依赖。
- 时间、随机数等不稳定依赖。
- 慢操作或错误路径难构造的依赖。
- 多实现策略确实存在。

接口是为了降低耦合，不是为了增加层数。
