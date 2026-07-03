# 067. 结构体嵌入 - 面试追问

## 1. 为什么说嵌入不是继承？

因为外层类型和内层类型仍然是两个独立类型，不能互相替代。

```go
type User struct {
	Name string
}

type Admin struct {
	User
}

func needUser(u User) {}

func main() {
	a := Admin{User: User{Name: "alice"}}

	// needUser(a) // 编译失败
	needUser(a.User)
}
```

继承通常意味着子类可以替代父类。Go 的嵌入只提供组合和选择器提升，不提供这种类型替代关系。

## 2. 方法提升会不会让外层类型实现接口？

会。外层类型的方法集可能包含嵌入字段提升上来的方法。

```go
type Runner interface {
	Run() error
}

type Worker struct{}

func (Worker) Run() error {
	return nil
}

type Service struct {
	Worker
}

var _ Runner = Service{}
```

如果嵌入的是指针类型，方法集还要结合值类型和指针类型判断。

```go
type Worker struct{}

func (*Worker) Run() error {
	return nil
}

type Service struct {
	*Worker
}

var _ Runner = Service{}  // 可以：Service 含有 *Worker 嵌入字段
var _ Runner = &Service{} // 也可以
```

真实项目里，如果依赖提升方法实现接口，最好写编译期断言，避免后续改字段时悄悄破坏实现关系。

## 3. 字段名冲突时 Go 怎么处理？

Go 不会猜，短选择器会报歧义。

```go
type ConfigA struct {
	Timeout int
}

type ConfigB struct {
	Timeout int
}

type App struct {
	ConfigA
	ConfigB
}

func main() {
	app := App{}

	// _ = app.Timeout // ambiguous selector
	_ = app.ConfigA.Timeout
	_ = app.ConfigB.Timeout
}
```

如果两个字段表达的是不同业务概念，用普通命名字段往往更清楚。

```go
type App struct {
	readConfig  ConfigA
	writeConfig ConfigB
}
```

## 4. 什么时候应该用嵌入，什么时候应该用普通字段？

适合嵌入：外层确实想暴露内层的行为，并且这个行为稳定、很小。

```go
type BufferLogger struct {
	bytes.Buffer
}

func (b *BufferLogger) Log(s string) {
	b.WriteString(s)
	b.WriteByte('\n')
}
```

不适合嵌入：内层只是实现细节，不希望调用方直接使用。

```go
type Service struct {
	client *http.Client // 命名字段，不把 client 的方法提升出去
}
```

判断标准：调用方是否应该知道并直接使用内层能力。如果不应该，就不要嵌入。

## 5. 为什么嵌入 `sync.Mutex` 后不能随便复制结构体？

锁的状态被复制后，两个结构体会持有两个锁状态，但它们可能保护同一份或本应同一份数据，语义会混乱。

```go
type Counter struct {
	sync.Mutex
	n int
}

func main() {
	c1 := Counter{}
	c1.Lock()

	c2 := c1 // 复制了已经使用过的 Mutex

	c1.Unlock()
	_ = c2
}
```

正确做法是使用指针传递含锁对象，不在使用后复制它。

```go
func inc(c *Counter) {
	c.Lock()
	defer c.Unlock()
	c.n++
}
```

可以用工具辅助检查：

```bash
go vet -copylocks ./...
```
