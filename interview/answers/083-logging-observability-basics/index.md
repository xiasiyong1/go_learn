# 083. 日志

## 问题

Go 服务日志应该怎么打？结构化日志和 context 有什么关系？

## 先给结论

日志不是越多越好，而是要在边界处记录能帮助定位问题的事件。生产服务优先使用结构化日志，用稳定字段表达请求、用户、资源、依赖、耗时和错误。`context` 适合携带 request id、trace id 这类请求范围元数据，但不应该被当作普通参数袋。

## 1. 结构化日志比拼字符串更容易检索

不推荐：

```go
log.Printf("user %s create order %s failed: %v", userID, orderID, err)
```

推荐结构化字段：

```go
slog.ErrorContext(ctx, "create order failed",
	"user_id", userID,
	"order_id", orderID,
	"err", err,
)
```

结构化日志的优势是机器可以按字段检索和聚合，例如查某个 `order_id` 的所有失败日志。

## 2. 错误通常在边界层记录一次

底层函数返回带上下文的 error：

```go
func (r Repo) LoadUser(ctx context.Context, id string) (User, error) {
	user, err := r.db.GetUser(ctx, id)
	if err != nil {
		return User{}, fmt.Errorf("load user %s: %w", id, err)
	}
	return user, nil
}
```

HTTP 边界统一记录：

```go
func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	user, err := h.repo.LoadUser(r.Context(), r.URL.Query().Get("id"))
	if err != nil {
		slog.ErrorContext(r.Context(), "request failed", "err", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	_ = json.NewEncoder(w).Encode(user)
}
```

不要每层都打同一个错误，否则日志里会出现多条重复噪音。

## 3. context 适合传请求范围元数据

定义私有 key，避免冲突：

```go
type contextKey string

const requestIDKey contextKey = "request_id"

func WithRequestID(ctx context.Context, id string) context.Context {
	return context.WithValue(ctx, requestIDKey, id)
}

func RequestID(ctx context.Context) string {
	v, _ := ctx.Value(requestIDKey).(string)
	return v
}
```

写日志时取出来：

```go
slog.InfoContext(ctx, "request started",
	"request_id", RequestID(ctx),
	"path", path,
)
```

不要把业务参数、数据库连接、logger 全部塞进 context。context value 适合跨 API 边界传请求元数据。

## 4. 日志级别要表达关注度

```go
slog.Debug("cache miss", "key", key)
slog.Info("server started", "addr", addr)
slog.Warn("downstream slow", "service", "payment", "cost", cost)
slog.Error("downstream failed", "service", "payment", "err", err)
```

常见判断：

- Debug：开发或临时排查。
- Info：关键生命周期和业务事件。
- Warn：可恢复但需要关注。
- Error：请求或任务失败。

日志级别不是指标系统。QPS、错误率、延迟应该进 metrics。

## 5. 敏感字段必须脱敏

错误示例：

```go
slog.Info("login", "password", password, "token", token)
```

正确做法：

```go
slog.Info("login",
	"user_id", userID,
	"token", "***",
)
```

对结构体输出也要小心：

```go
type User struct {
	ID       string
	Password string
}

slog.Info("user", "user", user) // 可能泄露 Password
```

更稳的是只输出必要字段。

## 6. 高频日志要采样或降级

循环里打错误日志可能把日志系统打爆。

```go
for _, item := range items {
	if err := process(item); err != nil {
		slog.Error("process item failed", "err", err) // 高频时很危险
	}
}
```

可以聚合后记录摘要：

```go
failed := 0
for _, item := range items {
	if err := process(item); err != nil {
		failed++
	}
}

if failed > 0 {
	slog.Warn("batch finished with failures", "failed", failed, "total", len(items))
}
```

需要逐条排查时再加采样或 debug 级别。

## 7. 面试时怎么答

可以这样回答：

- 日志要记录事件和上下文，不是到处 print。
- 生产服务优先结构化日志，字段要稳定可检索。
- 底层返回带上下文的 error，边界层统一记录，避免重复日志。
- context 可传 request id、trace id 等请求元数据。
- 敏感字段必须脱敏。
- 高频路径要控制日志量，指标和 trace 不能用日志替代。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么不要每一层都打印同一个 error？
- 结构化日志字段应该怎么选？
- request id 应该怎么进入日志？
- 日志、指标、trace 分别解决什么问题？
- 如何避免日志泄露敏感信息？
