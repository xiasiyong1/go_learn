# 070. 时间 - 面试追问

## 1. 为什么 Go 时间格式化用 `2006-01-02`？

Go 用一个固定参考时间表达布局：`Mon Jan 2 15:04:05 MST 2006`。你在布局里怎么写这个参考时间，输出就按相同形状生成。

```go
t := time.Date(2026, 7, 3, 9, 8, 7, 0, time.UTC)

fmt.Println(t.Format("2006-01-02"))          // 2026-07-03
fmt.Println(t.Format("2006-01-02 15:04:05")) // 2026-07-03 09:08:07
fmt.Println(t.Format("15:04"))               // 09:08
```

错误示例：

```go
fmt.Println(t.Format("YYYY-MM-DD")) // YYYY-MM-DD
```

Go 不使用 `%Y` 这类占位符，所以这类字符串会被当成普通文本。

## 2. `Parse` 和 `ParseInLocation` 有什么区别？

`Parse` 解析不带时区的字符串时，默认得到 UTC 时间。

```go
t, _ := time.Parse("2006-01-02 15:04:05", "2026-07-03 10:00:00")
fmt.Println(t.Location()) // UTC
```

`ParseInLocation` 会把不带时区的字符串解释为指定地点的本地时间。

```go
loc, _ := time.LoadLocation("Asia/Shanghai")
t, _ := time.ParseInLocation("2006-01-02 15:04:05", "2026-07-03 10:00:00", loc)
fmt.Println(t.Location()) // Asia/Shanghai
```

如果用户输入的是“北京时间 10 点”，就不应该用 `Parse` 默认为 UTC。

## 3. 为什么 `time.Time` 不建议直接用 `==` 比较？

`==` 会比较结构体所有字段，包括 Location 和可能存在的单调时间信息。业务上判断两个时间是否表示同一瞬间，用 `Equal`。

```go
utc := time.Date(2026, 7, 3, 2, 0, 0, 0, time.UTC)
loc, _ := time.LoadLocation("Asia/Shanghai")
sh := utc.In(loc)

fmt.Println(utc == sh)     // false
fmt.Println(utc.Equal(sh)) // true
```

判断先后顺序用：

```go
if a.Before(b) {
	fmt.Println("a is earlier")
}
```

只有你确实想比较 Time 结构体内部所有信息时，才考虑 `==`。

## 4. `time.Duration` 为什么不要用裸整数表示？

裸整数没有单位，读者不知道是秒、毫秒还是纳秒。

```go
timeout := 5 // 5 是什么单位？
_ = timeout
```

明确写单位：

```go
timeout := 5 * time.Second
interval := 200 * time.Millisecond

_ = timeout
_ = interval
```

函数签名也应该表达单位：

```go
func wait(d time.Duration) {
	time.Sleep(d)
}

wait(500 * time.Millisecond)
```

这比 `func wait(ms int)` 更不容易误用。

## 5. 单调时间解决什么问题，又不能解决什么问题？

它解决“测量时间间隔时系统墙上时间被调整”的问题。

```go
start := time.Now()
doWork()
fmt.Println(time.Since(start))
```

`time.Since` 会尽量使用单调时钟读数来计算耗时。

但它不能用于业务时间存储。格式化、JSON、数据库写入都会丢掉单调时钟信息。

```go
now := time.Now()
b, _ := now.MarshalJSON()

var decoded time.Time
_ = decoded.UnmarshalJSON(b)
```

反序列化后的 `decoded` 只保留墙上时间，不保留原来的单调时钟读数。业务过期时间、创建时间、账单时间仍然应该按明确时区和时间点处理。
