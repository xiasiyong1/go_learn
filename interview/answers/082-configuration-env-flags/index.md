# 082. 配置

## 问题

Go 服务中的配置、环境变量和命令行参数应该怎么管理？

## 先给结论

配置管理不是简单 `os.Getenv`。一个靠谱的 Go 服务应该集中加载配置，定义强类型结构，处理默认值、来源优先级、类型转换、必填校验和敏感信息脱敏。业务代码不应该到处直接读环境变量，否则配置来源和测试都会变乱。

## 1. 环境变量都是字符串

```go
port := os.Getenv("PORT")
fmt.Printf("%T %q\n", port, port) // string
```

如果配置是数字、布尔、时间段，都要解析。

```go
raw := os.Getenv("PORT")
port, err := strconv.Atoi(raw)
if err != nil {
	return fmt.Errorf("invalid PORT %q: %w", raw, err)
}
```

不要让解析失败后默默使用零值。

```go
port, _ := strconv.Atoi(os.Getenv("PORT")) // 错：失败时 port 变 0
```

`0` 可能是非法端口，也可能造成监听随机端口，问题很隐蔽。

## 2. 用 `LookupEnv` 区分未设置和空字符串

`os.Getenv` 不能区分“没设置”和“设置为空字符串”。

```go
fmt.Println(os.Getenv("NAME")) // 未设置和空字符串都返回 ""
```

用 `LookupEnv`：

```go
value, ok := os.LookupEnv("NAME")
if !ok {
	fmt.Println("NAME is not set")
} else {
	fmt.Println("NAME =", value)
}
```

这对密码、开关、可选字段很重要。

## 3. 集中加载成强类型配置

推荐定义配置结构：

```go
type Config struct {
	Addr        string
	DatabaseDSN string
	Timeout     time.Duration
	Debug       bool
}
```

集中加载：

```go
func LoadConfig() (Config, error) {
	cfg := Config{
		Addr:    ":8080",
		Timeout: 3 * time.Second,
	}

	if v, ok := os.LookupEnv("ADDR"); ok {
		cfg.Addr = v
	}

	dsn, ok := os.LookupEnv("DATABASE_DSN")
	if !ok || dsn == "" {
		return Config{}, fmt.Errorf("DATABASE_DSN is required")
	}
	cfg.DatabaseDSN = dsn

	if v, ok := os.LookupEnv("TIMEOUT"); ok {
		d, err := time.ParseDuration(v)
		if err != nil {
			return Config{}, fmt.Errorf("invalid TIMEOUT: %w", err)
		}
		cfg.Timeout = d
	}

	if v, ok := os.LookupEnv("DEBUG"); ok {
		b, err := strconv.ParseBool(v)
		if err != nil {
			return Config{}, fmt.Errorf("invalid DEBUG: %w", err)
		}
		cfg.Debug = b
	}

	return cfg, nil
}
```

业务代码接收 `Config`，不要自己读环境变量。

## 4. 命令行参数适合本地工具和启动参数

```go
addr := flag.String("addr", ":8080", "listen address")
timeout := flag.Duration("timeout", 3*time.Second, "request timeout")
flag.Parse()

cfg := Config{
	Addr:    *addr,
	Timeout: *timeout,
}
```

服务中常见优先级是：命令行参数 > 环境变量 > 配置文件 > 默认值。具体优先级不重要，重要的是团队要统一并写清楚。

## 5. 敏感配置不能完整打印

错误示例：

```go
log.Printf("config: %+v", cfg) // 可能打印 DATABASE_DSN、token、password
```

提供脱敏输出：

```go
func (c Config) SafeString() string {
	return fmt.Sprintf("Config{Addr:%q Timeout:%s Debug:%v DatabaseDSN:***}",
		c.Addr, c.Timeout, c.Debug)
}

log.Println(cfg.SafeString())
```

或者只输出非敏感摘要。

## 6. 动态配置要考虑并发可见性

如果配置运行时会更新，不能让多个 goroutine 随便读写同一个结构体。

简单方式：用 `atomic.Value` 保存不可变快照。

```go
var current atomic.Value // stores Config

current.Store(cfg)

func CurrentConfig() Config {
	return current.Load().(Config)
}

func UpdateConfig(next Config) {
	current.Store(next)
}
```

前提是 `Config` 当作不可变值使用，不要在取出后修改内部共享 map/slice。

## 7. 面试时怎么答

可以这样回答：

- 环境变量都是字符串，必须解析和校验。
- 用 `LookupEnv` 区分未设置和空字符串。
- 启动时集中加载成强类型 Config，业务模块通过依赖注入拿配置。
- 配置要有默认值、必填校验、范围校验和明确优先级。
- 敏感配置不能完整打印，要脱敏。
- 动态配置要考虑并发安全，常见做法是不可变快照加 atomic。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `os.Getenv` 和 `os.LookupEnv` 有什么区别？
- 配置解析失败时为什么不能用零值继续启动？
- 配置来源优先级应该怎么设计？
- 敏感配置如何安全打印？
- 动态配置如何保证并发安全？
