# 015. 结构体基础

## 问题

结构体值拷贝、匿名字段和 tag 分别怎么理解？

## 先给结论

结构体是字段的聚合值，赋值、传参、返回时默认按值拷贝。这个拷贝是浅拷贝：字段本身会被复制，但字段里如果包含 slice、map、指针、接口等引用类数据，底层对象仍然可能共享。

匿名字段是组合，不是传统继承。它会带来字段和方法提升，但不会让两个类型变成父子关系。

tag 是字段上的字符串元数据，编译器不理解业务含义，`encoding/json`、ORM、校验库等通常通过反射读取它。

## 结构体值拷贝是浅拷贝

普通字段会复制：

```go
type Point struct {
	X int
	Y int
}

p1 := Point{X: 1, Y: 2}
p2 := p1

p2.X = 100

fmt.Println(p1.X) // 1
fmt.Println(p2.X) // 100
```

但是引用类字段的底层数据可能共享：

```go
type User struct {
	Name string
	Tags []string
}

u1 := User{Name: "Tom", Tags: []string{"go"}}
u2 := u1

u2.Tags[0] = "java"

fmt.Println(u1.Tags[0]) // java
fmt.Println(u2.Tags[0]) // java
```

`u2 := u1` 复制了 slice 描述符，但两个描述符指向同一个底层数组。

如果要深拷贝，需要显式复制引用字段：

```go
func CloneUser(u User) User {
	tags := append([]string(nil), u.Tags...)
	return User{
		Name: u.Name,
		Tags: tags,
	}
}
```

map 字段也一样：

```go
type Config struct {
	Labels map[string]string
}

func CloneConfig(c Config) Config {
	labels := make(map[string]string, len(c.Labels))
	for k, v := range c.Labels {
		labels[k] = v
	}
	return Config{Labels: labels}
}
```

## 含锁或状态的结构体不要随便拷贝

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func Bad(c Counter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

这里复制了 `Counter`，锁和计数都是副本，不能保护原对象。

应该传指针：

```go
func Good(c *Counter) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

可以用工具检查：

```sh
go vet -copylocks ./...
```

## 匿名字段是组合，不是继承

```go
type Logger struct{}

func (Logger) Print(msg string) {
	fmt.Println(msg)
}

type Service struct {
	Logger
	Name string
}

func main() {
	s := Service{Name: "api"}
	s.Print("start")
}
```

`Service` 匿名嵌入了 `Logger`，所以可以直接写 `s.Print("start")`。这叫方法提升。

但 `Service` 不是 `Logger` 的子类：

```go
func UseLogger(l Logger) {}

s := Service{}
// UseLogger(s) // 编译错误
UseLogger(s.Logger)
```

Go 的复用方式是组合。匿名字段只是让访问语法更短，不是继承关系。

## 匿名字段可能暴露过多 API

如果匿名嵌入导出类型，它的方法也可能被提升到外层类型上，成为外层类型 API 的一部分。

```go
type Client struct{}

func (Client) Close() error {
	return nil
}

type Service struct {
	Client
}
```

调用方可以写：

```go
var s Service
_ = s.Close()
```

如果你不希望 `Service` 对外暴露 `Close`，就不要匿名嵌入，改成具名字段：

```go
type Service struct {
	client Client
}
```

组合能复用行为，但公共结构体的匿名字段要谨慎，因为它会影响对外 API。

## tag 是字符串元数据

```go
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name,omitempty"`
}
```

`json:"id"` 不是编译器规则，而是 `encoding/json` 通过反射读取 tag 后决定字段名。

可以自己读 tag：

```go
func PrintJSONTags(v any) {
	t := reflect.TypeOf(v)
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		fmt.Println(f.Name, f.Tag.Get("json"))
	}
}
```

tag 写错，编译通常不会报错：

```go
type User struct {
	ID int64 `json: "id"` // 格式不规范，go vet 会提示
}
```

运行前最好用：

```sh
go vet ./...
```

## 结构体是否可比较

所有字段都可比较时，结构体才可比较。

```go
type Point struct {
	X int
	Y int
}

fmt.Println(Point{1, 2} == Point{1, 2}) // true
```

包含 slice、map、func 等不可比较字段时，结构体不可比较：

```go
type User struct {
	Tags []string
}

// fmt.Println(User{} == User{}) // 编译错误
```

如果需要比较这类结构体，要明确比较规则，例如用 `reflect.DeepEqual`、`slices.Equal`，或者自己写比较函数。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 结构体赋值是深拷贝还是浅拷贝？
- 匿名字段为什么不是继承？
- 方法提升和字段提升会带来什么 API 风险？
- tag 是谁解析的？写错为什么编译还能过？
- 什么样的结构体可以用 `==` 比较？
- 含 `sync.Mutex` 的结构体为什么不能复制？
