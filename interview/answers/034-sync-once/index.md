# 034. sync.Once

## 问题

`sync.Once` 适合解决什么问题？

## 先给结论

`sync.Once` 保证某段函数在并发环境下最多执行一次。常见场景是惰性初始化单例、全局配置、昂贵资源。

关键点：

- `Once` 的语义是“最多执行一次”，不是“成功执行一次”。
- `Do` 里的函数 panic 后，`Once` 也会认为已经执行过。
- `Once` 不提供重置能力。
- 初始化可能失败并且需要重试时，不要直接用普通 `Once` 包住失败逻辑。

## 基本用法

```go
var once sync.Once
var client *Client

func GetClient() *Client {
	once.Do(func() {
		client = NewClient()
	})
	return client
}
```

多个 goroutine 同时调用 `GetClient` 时，只有一个 goroutine 会执行初始化函数。其他 goroutine 会等初始化完成，然后看到初始化后的结果。

## 初始化带错误时要小心

常见写法：

```go
var once sync.Once
var client *Client
var initErr error

func GetClient() (*Client, error) {
	once.Do(func() {
		client, initErr = NewClient()
	})
	return client, initErr
}
```

这个写法的语义是：第一次初始化失败后，后续调用不会重试，只会继续返回第一次的错误。

如果这就是你想要的，比如配置错误不可恢复，可以接受。

如果希望下次再试，就不能这样写。可以用锁显式控制：

```go
type ClientProvider struct {
	mu     sync.Mutex
	client *Client
}

func (p *ClientProvider) Get() (*Client, error) {
	p.mu.Lock()
	defer p.mu.Unlock()

	if p.client != nil {
		return p.client, nil
	}

	c, err := NewClient()
	if err != nil {
		return nil, err
	}
	p.client = c
	return c, nil
}
```

这里初始化失败不会缓存错误，下一次调用还会重试。

## panic 后也不会再执行

```go
var once sync.Once

func Init() {
	once.Do(func() {
		panic("failed")
	})
}
```

如果 `Do` 里的函数 panic，这次 `Do` 没有正常返回，但 `Once` 仍然会认为这段函数已经执行过。后续调用不会再执行它。

所以不要把“可能 panic 后再重试”的逻辑放进 `sync.Once`。

## 带参数初始化要谨慎

错误方向：

```go
var once sync.Once
var client *Client

func GetClient(dsn string) *Client {
	once.Do(func() {
		client = NewClientWithDSN(dsn)
	})
	return client
}
```

第一次调用传入的 `dsn` 决定了全局 client。后续调用即使传不同参数也无效。

更清楚的做法是配置来源固定：

```go
func NewClientProvider(dsn string) *ClientProvider {
	return &ClientProvider{dsn: dsn}
}

type ClientProvider struct {
	once   sync.Once
	dsn    string
	client *Client
	err    error
}

func (p *ClientProvider) Get() (*Client, error) {
	p.once.Do(func() {
		p.client, p.err = NewClientWithDSN(p.dsn)
	})
	return p.client, p.err
}
```

参数在构造 provider 时确定，而不是由第一次调用偷偷决定。

## 测试里不要依赖全局 Once 可重置

全局 Once 会污染测试：

```go
var once sync.Once
var cfg Config
```

一个测试初始化后，另一个测试很难换配置。

更好的方式是封装成实例：

```go
type Loader struct {
	once sync.Once
	cfg  Config
	err  error
	path string
}

func NewLoader(path string) *Loader {
	return &Loader{path: path}
}

func (l *Loader) Config() (Config, error) {
	l.once.Do(func() {
		l.cfg, l.err = LoadConfig(l.path)
	})
	return l.cfg, l.err
}
```

每个测试创建自己的 `Loader`，互不影响。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `sync.Once` 保证的是成功一次还是最多一次？
- `Do` 里的函数返回错误时，后续会不会重试？
- `Do` 里的函数 panic 后，后续会不会重试？
- 带参数的初始化为什么容易被第一次调用污染？
- 测试中全局 Once 有什么问题？
- 什么时候不用 Once，改用显式初始化更好？
