# 083. 日志 - 面试追问

## 1. 为什么不要每一层都打印同一个 error？

因为会制造重复噪音，让排查者以为发生了多次错误。

不推荐：

```go
func repo() error {
	err := errors.New("db failed")
	slog.Error("repo failed", "err", err)
	return err
}

func service() error {
	if err := repo(); err != nil {
		slog.Error("service failed", "err", err)
		return err
	}
	return nil
}
```

更好的方式：底层加上下文，边界层记录。

```go
func repo() error {
	if err := query(); err != nil {
		return fmt.Errorf("query user: %w", err)
	}
	return nil
}

func handler(w http.ResponseWriter, r *http.Request) {
	if err := repo(); err != nil {
		slog.ErrorContext(r.Context(), "request failed", "err", err)
	}
}
```

## 2. 结构化日志字段应该怎么选？

选择能帮助检索和定位的字段。

```go
slog.ErrorContext(ctx, "payment request failed",
	"request_id", requestID,
	"user_id", userID,
	"order_id", orderID,
	"provider", "stripe",
	"cost_ms", cost.Milliseconds(),
	"err", err,
)
```

字段命名要稳定。不要今天叫 `userID`，明天叫 `uid`，后天叫 `user_id`。字段稳定性会直接影响日志查询和告警规则。

## 3. request id 应该怎么进入日志？

入口 middleware 生成或读取 request id，放入 context，后续日志从 context 取。

```go
func requestID(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		id := r.Header.Get("X-Request-ID")
		if id == "" {
			id = newRequestID()
		}
		ctx := WithRequestID(r.Context(), id)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

日志：

```go
slog.InfoContext(ctx, "load user",
	"request_id", RequestID(ctx),
	"user_id", userID,
)
```

真实项目也可能由 OpenTelemetry trace id 统一串联，但思路一样：入口注入，链路传递，日志输出。

## 4. 日志、指标、trace 分别解决什么问题？

日志回答“发生了什么具体事件”。

```go
slog.Error("db query failed", "sql", "select user", "err", err)
```

指标回答“整体趋势怎么样”。

```text
http_requests_total
http_request_duration_seconds
db_errors_total
```

trace 回答“一次请求经过哪些步骤、每步耗时多少”。

面试里可以这样说：日志用于细节，指标用于告警和趋势，trace 用于链路和耗时。不要用日志硬算 QPS，也不要指望指标告诉你某个订单为什么失败。

## 5. 如何避免日志泄露敏感信息？

第一，不记录敏感字段。

```go
slog.Info("user login", "user_id", userID)
```

第二，必须记录时脱敏。

```go
func maskToken(token string) string {
	if len(token) <= 6 {
		return "***"
	}
	return token[:3] + "***" + token[len(token)-3:]
}
```

第三，不直接打印完整结构体。

```go
// 不推荐
slog.Info("request", "body", req)

// 推荐
slog.Info("request", "user_id", req.UserID, "action", req.Action)
```

敏感信息包括密码、token、cookie、身份证、手机号、邮箱、银行卡、私钥、DSN 等。公共日志规范里应该明确这些字段。
