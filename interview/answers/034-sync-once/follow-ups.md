# 034. sync.Once - 面试追问

## 1. `sync.Once` 保证的是成功一次还是最多一次？

保证最多执行一次。

```go
var once sync.Once
var n int

for i := 0; i < 10; i++ {
	once.Do(func() {
		n++
	})
}

fmt.Println(n) // 1
```

它不关心你的初始化是否“业务上成功”，只关心函数是否已经执行过。

## 2. `Do` 里的函数返回错误时，后续会不会重试？

如果你把错误保存到外部变量里，后续不会重试。

```go
var once sync.Once
var cfg Config
var err error

func GetConfig() (Config, error) {
	once.Do(func() {
		cfg, err = LoadConfig()
	})
	return cfg, err
}
```

第一次 `LoadConfig` 失败后，后续 `GetConfig` 还是返回同一个错误。

如果需要失败后重试，用锁自己控制：

```go
type Loader struct {
	mu  sync.Mutex
	cfg *Config
}

func (l *Loader) Get() (*Config, error) {
	l.mu.Lock()
	defer l.mu.Unlock()

	if l.cfg != nil {
		return l.cfg, nil
	}
	cfg, err := LoadConfig()
	if err != nil {
		return nil, err
	}
	l.cfg = &cfg
	return l.cfg, nil
}
```

## 3. `Do` 里的函数 panic 后，后续会不会重试？

不会。

```go
var once sync.Once

func Init() {
	once.Do(func() {
		panic("boom")
	})
}
```

第一次调用 panic 后，`once` 仍然认为函数已经执行过。后续调用不会再执行初始化函数。

所以如果初始化可能 panic 且你希望恢复后重试，不要直接用 `sync.Once` 表达这个逻辑。

## 4. 带参数的初始化为什么容易被第一次调用污染？

因为 `Once` 只执行第一次调用传入的函数。

```go
func GetClient(dsn string) *Client {
	once.Do(func() {
		client = NewClient(dsn)
	})
	return client
}
```

第一次调用传 `"test"`，后续传 `"prod"` 也不会生效。

更好的做法是把参数放到构造阶段：

```go
type Provider struct {
	once sync.Once
	dsn  string
	c    *Client
}

func NewProvider(dsn string) *Provider {
	return &Provider{dsn: dsn}
}
```

这样第一次调用不会偷偷决定全局配置。

## 5. 测试中全局 Once 有什么问题？

全局 `Once` 一旦执行，后续测试很难重置状态。

```go
var once sync.Once
var cfg Config
```

测试 A 加载了一个配置，测试 B 想换配置时，`once.Do` 不会再执行。

更好的设计是用实例：

```go
func TestConfig(t *testing.T) {
	loader := NewLoader("testdata/config.yaml")
	cfg, err := loader.Config()
	_ = cfg
	_ = err
}
```

每个测试有自己的 loader 和 once。

## 6. 什么时候不用 Once，改用显式初始化更好？

当初始化需要参数、可能失败并重试、需要关闭资源、需要热更新或多租户隔离时，显式初始化更好。

```go
func NewServer(cfg Config) (*Server, error) {
	db, err := OpenDB(cfg.DSN)
	if err != nil {
		return nil, err
	}
	return &Server{db: db}, nil
}
```

`Once` 适合生命周期简单、全局唯一、初始化结果稳定的对象。复杂资源管理不要为了懒加载把生命周期藏起来。
