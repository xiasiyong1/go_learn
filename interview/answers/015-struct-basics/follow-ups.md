# 015. 结构体基础 - 面试追问

## 1. 结构体赋值是深拷贝还是浅拷贝？

是字段级别的值拷贝，也就是浅拷贝。

```go
type Profile struct {
	Name string
	Tags []string
}

a := Profile{Name: "Tom", Tags: []string{"go"}}
b := a

b.Name = "Jerry"
b.Tags[0] = "java"

fmt.Println(a.Name)    // Tom
fmt.Println(a.Tags[0]) // java
```

`Name` 是字符串字段，赋值后 `a.Name` 不受 `b.Name` 影响。`Tags` 是 slice 字段，拷贝的是 slice 描述符，底层数组仍然共享。

要深拷贝就手动复制引用字段：

```go
func CloneProfile(p Profile) Profile {
	tags := append([]string(nil), p.Tags...)
	return Profile{Name: p.Name, Tags: tags}
}
```

## 2. 匿名字段为什么不是继承？

匿名字段是组合。外层类型拥有一个内层字段，字段和方法可以被提升，但外层类型不是内层类型的子类。

```go
type Engine struct{}

func (Engine) Start() {}

type Car struct {
	Engine
}

var c Car
c.Start() // 方法提升
```

但不能把 `Car` 当成 `Engine` 传入：

```go
func UseEngine(e Engine) {}

// UseEngine(c) // 编译错误
UseEngine(c.Engine)
```

如果要表达能力，应该用接口：

```go
type Starter interface {
	Start()
}

func UseStarter(s Starter) {}

UseStarter(c) // 可以，因为 Car 的方法集中有 Start
```

## 3. 方法提升和字段提升会带来什么 API 风险？

匿名嵌入的导出方法可能变成外层类型的公开 API。

```go
type DB struct{}

func (DB) Close() error {
	return nil
}

type Repository struct {
	DB
}
```

调用方可以直接：

```go
var r Repository
_ = r.Close()
```

如果后续你想移除或替换 `DB`，`Repository.Close` 也可能变成兼容性负担。

如果只是内部复用，具名未导出字段更稳：

```go
type Repository struct {
	db DB
}
```

面试回答要说明：匿名嵌入减少代码，但会扩大外层类型的方法集。

## 4. tag 是谁解析的？写错为什么编译还能过？

tag 本质上是结构体字段上的字符串。编译器只检查它是合法字符串字面量，不理解 `json`、`db`、`validate` 的业务规则。

```go
type User struct {
	ID int64 `json:"id"`
}
```

`encoding/json` 会通过反射读取：

```go
t := reflect.TypeOf(User{})
f, _ := t.FieldByName("ID")
fmt.Println(f.Tag.Get("json")) // id
```

写错 tag，编译可能仍然通过：

```go
type User struct {
	ID int64 `json: "id"`
}
```

但工具可以检查很多常见错误：

```sh
go vet ./...
```

## 5. 什么样的结构体可以用 `==` 比较？

所有字段都可比较时，结构体才可比较。

```go
type Point struct {
	X int
	Y int
}

fmt.Println(Point{1, 2} == Point{1, 2}) // true
```

含 slice、map、func 字段时不可比较：

```go
type User struct {
	Tags []string
}

// User{} == User{} // 编译错误
```

如果要比较 slice 字段，可以自己定义规则：

```go
func EqualUser(a, b User) bool {
	return slices.Equal(a.Tags, b.Tags)
}
```

不要用 `reflect.DeepEqual` 偷懒替代业务规则，尤其是 nil slice 和 empty slice 是否相等这类问题，要看你的契约。

## 6. 含 `sync.Mutex` 的结构体为什么不能复制？

因为复制会把锁状态也复制出去，锁保护的对象关系就不清楚了。

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

`Inc` 使用值接收者，会复制 `Counter`。锁住的是副本，原对象没有被修改。

应该使用指针接收者：

```go
func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

检查命令：

```sh
go vet -copylocks ./...
```
