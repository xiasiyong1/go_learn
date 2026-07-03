# 070. 时间

## 问题

Go 中 `time.Time`、`time.Duration`、时区和单调时间应该怎么理解？

## 先给结论

`time.Time` 表示一个时间点，`time.Duration` 表示两个时间点之间的时间段。Go 时间题不能只会 `time.Now()`，还要知道格式化布局、时区、UTC、本地时间、零值、单调时钟和 `==` 比较的坑。

## 1. `time.Time` 和 `time.Duration` 的区别

时间点：

```go
now := time.Now()
fmt.Println(now)
```

时间段：

```go
timeout := 1500 * time.Millisecond
fmt.Println(timeout.Seconds()) // 1.5
```

计算两个时间点之间的间隔：

```go
start := time.Now()
doWork()
cost := time.Since(start)
fmt.Println("cost:", cost)
```

面试里要说清楚：超时、重试间隔、定时器用 `time.Duration`；业务发生时间、创建时间、过期时间用 `time.Time`。

## 2. Go 的时间格式化布局不是 `%Y-%m-%d`

Go 使用固定参考时间 `2006-01-02 15:04:05` 作为布局。

```go
t := time.Date(2026, 7, 3, 9, 8, 7, 0, time.UTC)

fmt.Println(t.Format("2006-01-02 15:04:05")) // 2026-07-03 09:08:07
fmt.Println(t.Format("2006/01/02"))          // 2026/07/03
```

错误写法：

```go
fmt.Println(t.Format("YYYY-MM-DD")) // YYYY-MM-DD
```

如果从其他语言迁移过来，这个点非常容易错。

## 3. 解析时间时要明确时区

`time.Parse` 默认按 UTC 解析不带时区的字符串。

```go
t, err := time.Parse("2006-01-02 15:04:05", "2026-07-03 10:00:00")
if err != nil {
	panic(err)
}
fmt.Println(t.Location()) // UTC
```

如果字符串表示的是某个本地时区时间，用 `ParseInLocation`。

```go
loc, _ := time.LoadLocation("Asia/Shanghai")
t, err := time.ParseInLocation("2006-01-02 15:04:05", "2026-07-03 10:00:00", loc)
if err != nil {
	panic(err)
}
fmt.Println(t.Location()) // Asia/Shanghai
```

服务内部通常用 UTC 存储和传输，展示时再转用户时区。

```go
stored := t.UTC()
display := stored.In(loc)
```

## 4. `time.Time` 的零值要小心

`time.Time{}` 是零值，表示公元 1 年，不是当前时间，也不是 nil。

```go
var t time.Time

fmt.Println(t.IsZero()) // true
fmt.Println(t.Year())   // 1
```

如果时间字段是可选的，常见做法是用 `*time.Time` 或额外状态表达“没有值”。

```go
type User struct {
	DeletedAt *time.Time
}
```

## 5. 不要随便用 `==` 比较时间

`time.Time` 里可能包含 Location 和单调时钟信息。业务上比较两个时间是否表示同一瞬间，应该用 `Equal`。

```go
t1 := time.Date(2026, 7, 3, 10, 0, 0, 0, time.UTC)
loc, _ := time.LoadLocation("Asia/Shanghai")
t2 := t1.In(loc)

fmt.Println(t1 == t2)    // false，Location 不同
fmt.Println(t1.Equal(t2)) // true，表示同一瞬间
```

如果要排序，用 `Before`、`After`。

```go
if deadline.Before(time.Now()) {
	return errors.New("expired")
}
```

## 6. 单调时间用于测量间隔

`time.Now()` 返回的时间可能带单调时钟读数。用 `time.Since` 或 `Sub` 测耗时，可以避免系统时间被调整带来的影响。

```go
start := time.Now()
time.Sleep(10 * time.Millisecond)
fmt.Println(time.Since(start))
```

但序列化、格式化、数据库存储时，单调时钟信息不会保留。不要把它当业务字段依赖。

## 7. 面试时怎么答

可以这样回答：

- `time.Time` 是时间点，`time.Duration` 是时间段。
- Go 格式化布局使用 `2006-01-02 15:04:05`，不是 `%Y-%m-%d`。
- 内部存储和传输优先 UTC，展示时转用户时区。
- 解析无时区字符串要明确使用 `Parse` 还是 `ParseInLocation`。
- 时间零值用 `IsZero` 判断。
- 比较同一瞬间用 `Equal`，测耗时用 `time.Since`。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 Go 时间格式化用 `2006-01-02`？
- `Parse` 和 `ParseInLocation` 有什么区别？
- 为什么 `time.Time` 不建议直接用 `==` 比较？
- `time.Duration` 为什么不要用裸整数表示？
- 单调时间解决什么问题，又不能解决什么问题？
