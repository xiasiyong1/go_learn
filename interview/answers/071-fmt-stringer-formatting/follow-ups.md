# 071. 格式化 - 面试追问

## 1. `%v`、`%+v`、`%#v`、`%T` 分别适合什么场景？

用一个结构体就能看出区别。

```go
type User struct {
	Name string
	Age  int
}

u := User{Name: "alice", Age: 18}

fmt.Printf("%v\n", u)  // {alice 18}
fmt.Printf("%+v\n", u) // {Name:alice Age:18}
fmt.Printf("%#v\n", u) // main.User{Name:"alice", Age:18}
fmt.Printf("%T\n", u)  // main.User
```

调试接口动态类型时，`%T` 很有用：

```go
var v any = []int{1, 2}
fmt.Printf("%T\n", v) // []int
```

调试不可见字符时，用 `%q`：

```go
fmt.Printf("%q\n", "a\nb") // "a\nb"
```

## 2. `Stringer` 为什么会导致递归？

因为 `fmt` 发现一个值实现了 `String() string` 后，会调用它。你在 `String` 里再用 `%v` 打印自己，就又触发 `String`。

```go
type User struct {
	Name string
}

func (u User) String() string {
	return fmt.Sprintf("%v", u) // 递归
}
```

正确写法是打印字段：

```go
func (u User) String() string {
	return fmt.Sprintf("User(%s)", u.Name)
}
```

或者转换成没有 `String` 方法的别名类型：

```go
type rawUser User

func (u User) String() string {
	return fmt.Sprintf("%+v", rawUser(u))
}
```

## 3. `%w` 和 `%v` 包装 error 有什么区别？

`%w` 会保留错误链，`errors.Is/As` 可以继续识别底层错误。

```go
base := os.ErrNotExist
err := fmt.Errorf("open config: %w", base)

fmt.Println(errors.Is(err, os.ErrNotExist)) // true
```

`%v` 只把错误格式化成字符串，错误链断开。

```go
base := os.ErrNotExist
err := fmt.Errorf("open config: %v", base)

fmt.Println(errors.Is(err, os.ErrNotExist)) // false
```

所以业务代码需要保留错误语义时，用 `%w`；只是展示文本时才用 `%v`。

## 4. 为什么日志里不能随便打印结构体？

第一，可能泄露敏感字段。

```go
type Config struct {
	User     string
	Password string
}

cfg := Config{User: "root", Password: "secret"}
fmt.Printf("%+v\n", cfg) // Password 会直接输出
```

第二，字段太多时日志不可检索。结构化日志更适合生产排查：

```go
logger.Info("login failed",
	"user", userID,
	"reason", reason,
)
```

如果类型可能被日志打印，`String()` 里要主动脱敏：

```go
func (c Config) String() string {
	return fmt.Sprintf("Config{User:%s Password:***}", c.User)
}
```

## 5. 热点路径里为什么要少用 `fmt.Sprintf`？

`fmt.Sprintf` 是通用格式化工具，要处理反射、接口和多种格式动词，通常会有更多开销。

```go
s := fmt.Sprintf("%d:%s", id, name)
_ = s
```

简单整数转字符串用 `strconv` 更直接：

```go
s := strconv.Itoa(id) + ":" + name
```

大量拼接用 `strings.Builder`：

```go
var b strings.Builder
b.WriteString(strconv.Itoa(id))
b.WriteByte(':')
b.WriteString(name)
s := b.String()
```

不要凭感觉优化；如果怀疑 fmt 是热点，用 benchmark 和 pprof 验证。
