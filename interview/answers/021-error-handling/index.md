# 021. error 处理

## 问题

Go 为什么推荐显式返回 error？`errors.Is/As` 和 wrapping 怎么用？

## 先给结论

Go 推荐显式返回 `error`，是为了让失败路径出现在函数签名里，让调用方必须处理。

`error` 本质是接口：

```go
type error interface {
	Error() string
}
```

实际项目里，好的错误处理不只是 `if err != nil { return err }`，还要做到：

- 给错误加上下文。
- 保留原始错误语义。
- 调用方不要靠字符串判断错误。
- 在合适边界把错误映射成重试、HTTP 状态码、用户提示或日志。

## 基础写法：立即处理错误

```go
func LoadConfig(path string) ([]byte, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, err
	}
	return data, nil
}
```

Go 不用异常表达普通失败。文件不存在、参数非法、网络超时、数据库返回空结果，这些都应该通过 `error` 返回。

## 加上下文，但不要丢失原始错误

不推荐：

```go
if err != nil {
	return fmt.Errorf("load config failed: %v", err)
}
```

这里 `%v` 只是把错误转成字符串，调用方无法用 `errors.Is` 找回原始错误。

推荐：

```go
if err != nil {
	return fmt.Errorf("load config %s: %w", path, err)
}
```

`%w` 会包装错误，形成 unwrap 链。

调用方可以判断：

```go
data, err := LoadConfig("app.yaml")
if errors.Is(err, os.ErrNotExist) {
	return defaultConfig(), nil
}
if err != nil {
	return nil, err
}
_ = data
```

这样既有上下文，又保留了机器可判断的错误语义。

## 不要用字符串判断错误

错误示例：

```go
if strings.Contains(err.Error(), "not exist") {
	// fragile
}
```

问题是错误文案可能变化、语言可能变化、底层依赖可能变化。

更好的做法是用 `errors.Is` 判断稳定错误：

```go
if errors.Is(err, os.ErrNotExist) {
	// file missing
}
```

或者用 `errors.As` 提取错误类型：

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Op, pathErr.Path)
}
```

## 哨兵错误适合稳定语义

```go
var ErrNotFound = errors.New("not found")

func FindUser(id int64) (*User, error) {
	if id <= 0 {
		return nil, ErrNotFound
	}
	return &User{ID: id}, nil
}
```

调用方：

```go
u, err := FindUser(id)
if errors.Is(err, ErrNotFound) {
	return nil, nil
}
if err != nil {
	return nil, err
}
return u, nil
```

如果要加上下文：

```go
return nil, fmt.Errorf("find user %d: %w", id, ErrNotFound)
```

哨兵错误的代价是：一旦对外暴露，它就成了 API 契约，不要随便改。

## 自定义错误类型适合结构化信息

```go
type ValidationError struct {
	Field string
	Msg   string
}

func (e *ValidationError) Error() string {
	return e.Field + ": " + e.Msg
}

func ValidateUser(u User) error {
	if u.Name == "" {
		return &ValidationError{Field: "name", Msg: "required"}
	}
	return nil
}
```

调用方用 `errors.As`：

```go
var ve *ValidationError
if errors.As(err, &ve) {
	return http.StatusBadRequest, ve.Field
}
```

自定义类型适合携带字段、错误码、是否可重试等结构化信息。

## 日志应该在边界打

不推荐每层都打日志：

```go
func repo() error {
	err := query()
	if err != nil {
		log.Println("repo:", err)
		return err
	}
	return nil
}

func service() error {
	err := repo()
	if err != nil {
		log.Println("service:", err)
		return err
	}
	return nil
}
```

一条失败可能刷出多条重复日志。

更常见的做法是：底层加上下文并返回，边界统一记录。

```go
func repo() error {
	if err := query(); err != nil {
		return fmt.Errorf("query user: %w", err)
	}
	return nil
}

func handler(w http.ResponseWriter, r *http.Request) {
	if err := service(); err != nil {
		slog.Error("request failed", "err", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
	}
}
```

## 什么时候可以忽略错误

可以忽略，但要明确原因。

```go
_ = os.Remove(tmpFile) // best effort cleanup
```

如果这个错误影响业务，就不能吞：

```go
if err := rows.Close(); err != nil {
	return fmt.Errorf("close rows: %w", err)
}
```

经验是：忽略错误时，读代码的人应该能看出这是有意的，不是漏处理。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 `%w` 比 `%v` 更适合包装错误？
- `errors.Is` 和 `errors.As` 分别解决什么问题？
- 什么时候用哨兵错误，什么时候用自定义错误类型？
- 为什么不建议靠错误字符串做业务判断？
- 错误日志应该每层都打吗？
- 普通业务错误为什么不应该用 panic 表达？
