# 082. 配置 - 面试追问

## 1. `os.Getenv` 和 `os.LookupEnv` 有什么区别？

`Getenv` 只返回字符串，未设置和设置为空字符串都会得到 `""`。

```go
v := os.Getenv("API_KEY")
if v == "" {
	fmt.Println("empty or missing")
}
```

`LookupEnv` 能区分是否设置。

```go
v, ok := os.LookupEnv("API_KEY")
if !ok {
	return fmt.Errorf("API_KEY is required")
}
if v == "" {
	return fmt.Errorf("API_KEY must not be empty")
}
```

必填配置优先用 `LookupEnv`，否则你无法给出准确错误。

## 2. 配置解析失败时为什么不能用零值继续启动？

因为零值可能是危险配置。

```go
timeout, _ := time.ParseDuration(os.Getenv("TIMEOUT"))
client := &http.Client{Timeout: timeout} // 解析失败时 timeout 是 0，可能表示无超时
```

正确做法是启动失败：

```go
timeout, err := time.ParseDuration(raw)
if err != nil {
	return Config{}, fmt.Errorf("invalid TIMEOUT %q: %w", raw, err)
}
```

配置错误是部署错误，应该尽早暴露，而不是让服务带着错误配置运行。

## 3. 配置来源优先级应该怎么设计？

常见优先级：

```text
命令行参数 > 环境变量 > 配置文件 > 默认值
```

示例：

```go
cfg := DefaultConfig()

if fileCfg, err := LoadFile(path); err == nil {
	cfg = Merge(cfg, fileCfg)
}
if v, ok := os.LookupEnv("ADDR"); ok {
	cfg.Addr = v
}
if *addrFlag != "" {
	cfg.Addr = *addrFlag
}
```

关键不是固定哪种优先级，而是：

- 规则统一。
- 文档清楚。
- 启动日志能输出脱敏后的最终结果。
- 测试覆盖优先级冲突。

## 4. 敏感配置如何安全打印？

不要直接 `%+v` 打印完整配置。

```go
log.Printf("config=%+v", cfg) // 可能泄露密码
```

给配置定义脱敏输出：

```go
type Config struct {
	Addr string
	DSN  string
}

func (c Config) LogFields() map[string]any {
	return map[string]any{
		"addr": c.Addr,
		"dsn":  "***",
	}
}
```

或者只打印非敏感字段：

```go
log.Printf("addr=%s", cfg.Addr)
```

敏感信息包括 password、token、cookie、私钥、DSN 中的账号密码等。

## 5. 动态配置如何保证并发安全？

不要多个 goroutine 直接读写同一个可变配置结构。

错误示例：

```go
var cfg Config

go func() {
	cfg.Timeout = time.Second // 写
}()

go func() {
	fmt.Println(cfg.Timeout) // 读，数据竞争
}()
```

用不可变快照：

```go
var current atomic.Value // Config

func GetConfig() Config {
	return current.Load().(Config)
}

func SetConfig(c Config) {
	current.Store(c)
}
```

如果 Config 内部有 map/slice，更新前要深拷贝，避免读者拿到后看到后续修改。
