# 069. nil

## 问题

Go 中哪些类型可以是 nil？不同 nil 值的行为有什么区别？

## 先给结论

`nil` 只适用于指针、slice、map、channel、func、interface 这几类可以表示“没有引用目标”的类型。不同 nil 类型行为差异很大：nil slice 能 append，nil map 不能写，nil channel 会永久阻塞，nil func 调用会 panic，interface 还存在“接口本身不为 nil，但内部值为 nil”的陷阱。

## 1. 哪些类型可以和 nil 比较

可以为 nil 的类型：

```go
var p *int
var s []int
var m map[string]int
var ch chan int
var fn func()
var v any

fmt.Println(p == nil, s == nil, m == nil, ch == nil, fn == nil, v == nil)
```

基础值类型不能和 nil 比较：

```go
var n int
var str string
var st struct{}

// fmt.Println(n == nil)   // 编译失败
// fmt.Println(str == nil) // 编译失败
// fmt.Println(st == nil)  // 编译失败
```

所以 nil 不是“所有类型的空值”，它只属于特定类型。

## 2. nil slice 可以正常使用一部分操作

nil slice 可以 `len`、`cap`、`range`，也可以 `append`。

```go
var s []int

fmt.Println(s == nil) // true
fmt.Println(len(s))   // 0

for _, v := range s {
	fmt.Println(v) // 不会执行
}

s = append(s, 1)
fmt.Println(s) // [1]
```

这也是为什么函数返回空列表时，很多内部场景可以直接返回 nil slice。

但 JSON 序列化会区分 nil slice 和空 slice：

```go
var a []int
b := []int{}

aj, _ := json.Marshal(a)
bj, _ := json.Marshal(b)

fmt.Println(string(aj)) // null
fmt.Println(string(bj)) // []
```

对外 API 要按契约选择。

## 3. nil map 可以读和删，但不能写

```go
var m map[string]int

fmt.Println(m["x"]) // 0
delete(m, "x")      // 安全

// m["x"] = 1       // panic: assignment to entry in nil map
```

写 map 前必须初始化：

```go
m = make(map[string]int)
m["x"] = 1
```

这和 slice 的行为不同：slice 的 append 可以分配底层数组，map 赋值不能自动创建哈希表。

## 4. nil channel 会永久阻塞

对 nil channel 发送或接收都会永久阻塞。

```go
var ch chan int

// ch <- 1  // 永久阻塞
// <-ch     // 永久阻塞
```

但这个特性在 `select` 里有用：nil channel 的 case 永远不会被选中，可以动态禁用某个分支。

```go
var out chan<- int

if ready {
	out = realOut
}

select {
case out <- value:
	fmt.Println("sent")
case <-ctx.Done():
	return ctx.Err()
}
```

如果 `ready` 为 false，发送 case 就被禁用。

## 5. nil func 调用会 panic

```go
var fn func()

// fn() // panic: call of nil function
```

回调函数要么调用前判断 nil：

```go
if fn != nil {
	fn()
}
```

要么在构造时提供空实现：

```go
type Option struct {
	OnDone func()
}

func NewOption() Option {
	return Option{OnDone: func() {}}
}
```

## 6. interface 的 nil 陷阱

接口值由动态类型和动态值组成。只有两者都为空，接口才等于 nil。

```go
type MyErr struct{}

func (*MyErr) Error() string { return "bad" }

func returnsTypedNil() error {
	var e *MyErr = nil
	return e
}

func main() {
	err := returnsTypedNil()
	fmt.Println(err == nil) // false
}
```

这里 `err` 的动态类型是 `*MyErr`，动态值是 nil，所以接口本身不是 nil。

## 7. 面试时怎么答

可以这样回答：

- nil 只适用于指针、slice、map、channel、func、interface 等类型。
- nil slice 可以 append，nil map 不能写。
- nil channel 收发会永久阻塞，在 select 中可用于禁用 case。
- nil func 调用会 panic。
- interface 要区分接口本身为 nil 和内部动态值为 nil。
- 不同 nil 类型边界完全不同，不能只背“nil 是空”。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- nil slice 和 empty slice 在代码和 JSON 里有什么区别？
- 为什么 nil map 不能写，但 nil slice 可以 append？
- nil channel 在 select 里有什么实际用途？
- nil func 应该如何设计默认行为？
- 为什么 `var err error = (*MyErr)(nil)` 不等于 nil？
