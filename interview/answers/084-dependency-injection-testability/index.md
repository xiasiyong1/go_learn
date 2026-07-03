# 084. 可测试性

## 问题

Go 中依赖注入应该怎么做？如何让代码更容易测试？

## 先给结论

Go 不需要复杂 DI 框架也能写出可测试代码。核心做法是：依赖通过构造函数或函数参数显式传入，外部系统通过小接口隔离，时间、随机数、网络、数据库等不可控因素可以替换成 fake。目标不是“为了接口而接口”，而是让业务逻辑可以在单元测试里稳定验证。

## 1. 不要在业务代码里直接创建外部依赖

不推荐：

```go
func Notify(userID string) error {
	resp, err := http.Get("https://api.example.com/users/" + userID)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	return nil
}
```

这个函数测试时必须打真实网络，难以覆盖超时、500、返回体异常等场景。

改成依赖注入：

```go
type UserClient interface {
	LoadUser(ctx context.Context, id string) (User, error)
}

type Service struct {
	users UserClient
}

func NewService(users UserClient) *Service {
	return &Service{users: users}
}

func (s *Service) Notify(ctx context.Context, userID string) error {
	user, err := s.users.LoadUser(ctx, userID)
	if err != nil {
		return err
	}
	_ = user
	return nil
}
```

## 2. 接口定义在使用方，并且尽量小

如果业务只需要查询用户，就不要依赖一个巨大的 repository。

```go
type UserFinder interface {
	FindUser(ctx context.Context, id string) (User, error)
}

type Handler struct {
	users UserFinder
}
```

测试 fake 很简单：

```go
type fakeUsers struct {
	user User
	err  error
}

func (f fakeUsers) FindUser(ctx context.Context, id string) (User, error) {
	return f.user, f.err
}
```

接口越小，fake 越容易写，测试越聚焦。

## 3. 时间要可替换

不推荐业务逻辑里直接 `time.Now()`。

```go
func Expired(deadline time.Time) bool {
	return time.Now().After(deadline)
}
```

测试会依赖当前时间。改成注入 clock：

```go
type Clock interface {
	Now() time.Time
}

type realClock struct{}

func (realClock) Now() time.Time { return time.Now() }

func Expired(clock Clock, deadline time.Time) bool {
	return clock.Now().After(deadline)
}
```

测试：

```go
type fakeClock struct {
	now time.Time
}

func (f fakeClock) Now() time.Time { return f.now }
```

这样边界时间、过期、未过期都能稳定测试。

## 4. 函数参数注入适合简单依赖

不是所有依赖都要抽接口。简单函数依赖可以直接传函数。

```go
func Retry(ctx context.Context, sleep func(time.Duration), fn func() error) error {
	for i := 0; i < 3; i++ {
		if err := fn(); err == nil {
			return nil
		}
		sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("retry failed")
}
```

测试里传空 sleep，避免真的等待。

```go
err := Retry(context.Background(), func(time.Duration) {}, func() error {
	attempts++
	if attempts < 2 {
		return errors.New("fail")
	}
	return nil
})
```

## 5. 全局变量会污染测试

不推荐：

```go
var defaultClient = &http.Client{Timeout: time.Second}

func Fetch(url string) error {
	_, err := defaultClient.Get(url)
	return err
}
```

测试如果修改全局 client，可能影响其他测试，尤其是 `t.Parallel()`。

更好的方式是结构体持有依赖：

```go
type Fetcher struct {
	client *http.Client
}

func (f Fetcher) Fetch(ctx context.Context, url string) error {
	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	resp, err := f.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	return nil
}
```

## 6. 不要过度抽象

如果一个类型只有一个实现，测试也不需要替换它，不一定要先抽接口。

```go
type Calculator struct{}

func (Calculator) Add(a, b int) int {
	return a + b
}
```

为它提前抽接口没有收益：

```go
type Calculator interface {
	Add(int, int) int
}
```

接口应该由真实替换需求驱动：外部依赖、复杂状态、慢操作、不可控时间、随机数、网络、数据库。

## 7. 面试时怎么答

可以这样回答：

- 依赖注入的目标是让业务逻辑不直接依赖真实外部系统和全局状态。
- Go 常用构造函数、接口、函数参数做显式注入。
- 接口定义在使用方，并保持最小。
- 时间、随机数、HTTP、数据库等不可控依赖要能替换。
- 全局变量会让测试顺序相关，尽量避免。
- 不要为了测试过度抽象，只有真实替换需求时才抽接口。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- Go 为什么通常不需要复杂 DI 框架？
- 接口应该定义在生产者还是使用者？
- 时间相关逻辑怎么测试？
- 如何避免测试依赖真实 HTTP 或数据库？
- 什么时候不应该抽接口？
