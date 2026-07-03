# 021. error 处理 - 面试追问

## 1. 为什么 `%w` 比 `%v` 更适合包装错误？

`%v` 只把错误格式化成文本，丢失 unwrap 链。

```go
err := fmt.Errorf("read config: %v", os.ErrNotExist)

fmt.Println(errors.Is(err, os.ErrNotExist)) // false
```

`%w` 会保留原始错误：

```go
err := fmt.Errorf("read config: %w", os.ErrNotExist)

fmt.Println(errors.Is(err, os.ErrNotExist)) // true
```

所以底层错误需要被上层识别时，应该用 `%w`。如果只是输出文本，不需要错误语义，才考虑 `%v`。

## 2. `errors.Is` 和 `errors.As` 分别解决什么问题？

`errors.Is` 判断错误链里是否包含某个目标错误。

```go
var ErrNotFound = errors.New("not found")

err := fmt.Errorf("load user: %w", ErrNotFound)

fmt.Println(errors.Is(err, ErrNotFound)) // true
```

`errors.As` 从错误链里提取某种错误类型。

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Op, pathErr.Path)
}
```

一句话：

- `Is` 适合判断“是不是这个错误语义”。
- `As` 适合拿到“这个错误类型里的结构化字段”。

## 3. 什么时候用哨兵错误，什么时候用自定义错误类型？

哨兵错误适合少量、稳定、只需要判断类别的错误。

```go
var ErrNotFound = errors.New("not found")
```

调用方只关心是不是 not found：

```go
if errors.Is(err, ErrNotFound) {
	return nil
}
```

自定义错误类型适合携带结构化信息：

```go
type RateLimitError struct {
	RetryAfter time.Duration
}

func (e *RateLimitError) Error() string {
	return "rate limited"
}
```

调用方可以提取字段做决策：

```go
var e *RateLimitError
if errors.As(err, &e) {
	time.Sleep(e.RetryAfter)
}
```

## 4. 为什么不建议靠错误字符串做业务判断？

错误字符串是给人看的，不是稳定协议。

不推荐：

```go
if strings.Contains(err.Error(), "timeout") {
	retry()
}
```

文案变化、依赖升级、语言变化都可能破坏判断。

推荐定义稳定错误语义：

```go
var ErrTimeout = errors.New("timeout")

if errors.Is(err, ErrTimeout) {
	retry()
}
```

或者定义接口：

```go
type Temporary interface {
	Temporary() bool
}

var te Temporary
if errors.As(err, &te) && te.Temporary() {
	retry()
}
```

## 5. 错误日志应该每层都打吗？

通常不应该。

底层负责加上下文：

```go
func loadUser(id int64) error {
	if err := queryUser(id); err != nil {
		return fmt.Errorf("query user %d: %w", id, err)
	}
	return nil
}
```

边界层负责记录：

```go
func handler(w http.ResponseWriter, r *http.Request) {
	if err := loadUser(1); err != nil {
		slog.Error("request failed", "err", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}
```

每层都打日志会产生重复日志，反而让排查更难。

## 6. 普通业务错误为什么不应该用 panic 表达？

因为业务错误是预期失败，调用方应该能正常处理。

不推荐：

```go
func ParseAge(s string) int {
	n, err := strconv.Atoi(s)
	if err != nil {
		panic(err)
	}
	return n
}
```

推荐：

```go
func ParseAge(s string) (int, error) {
	n, err := strconv.Atoi(s)
	if err != nil {
		return 0, fmt.Errorf("parse age: %w", err)
	}
	return n, nil
}
```

`panic` 更适合程序员错误、不可恢复的不变量破坏，或者在 goroutine/HTTP 框架边界做兜底 recover。
