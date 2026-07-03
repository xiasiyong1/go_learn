# 069. nil - 面试追问

## 1. nil slice 和 empty slice 在代码和 JSON 里有什么区别？

代码里它们都可以 `len`、`range`、`append`。

```go
var a []int
b := []int{}

fmt.Println(len(a), len(b)) // 0 0

a = append(a, 1)
b = append(b, 1)
```

但它们和 nil 比较不同：

```go
var a []int
b := []int{}

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
```

JSON 表现也不同：

```go
var a []int
b := []int{}

aj, _ := json.Marshal(a)
bj, _ := json.Marshal(b)

fmt.Println(string(aj)) // null
fmt.Println(string(bj)) // []
```

内部逻辑通常用 `len(s) == 0` 判断空集合；对外 API 要根据契约决定返回 `null` 还是 `[]`。

## 2. 为什么 nil map 不能写，但 nil slice 可以 append？

nil slice 的 `append` 会创建新的底层数组并返回新切片。

```go
var s []int
s = append(s, 1)
fmt.Println(s) // [1]
```

nil map 没有底层哈希表，赋值时没有地方放 key/value。

```go
var m map[string]int

// m["x"] = 1 // panic

m = make(map[string]int)
m["x"] = 1
```

面试回答要强调：slice 的 append 返回新值，map 的赋值是对现有哈希表写入。

## 3. nil channel 在 select 里有什么实际用途？

可以动态禁用某个 case。

```go
var out chan<- int
if hasValue {
	out = realOut
}

select {
case out <- value:
	hasValue = false
case v := <-in:
	value = v
	hasValue = true
}
```

当 `out == nil` 时，发送分支不会被选中。这个技巧常用于状态机式 select，但要写得清楚，否则很容易变成死锁。

危险写法：

```go
var ch chan int

select {
case <-ch:
	// 永远不会执行
}
```

如果 select 里所有 case 都是 nil channel 且没有 default，就会永久阻塞。

## 4. nil func 应该如何设计默认行为？

如果回调是可选的，调用前判断 nil：

```go
type Hook func(error)

func run(h Hook) {
	if h != nil {
		h(nil)
	}
}
```

如果回调在业务流程中一定会被调用，可以在构造阶段提供空函数，减少调用处判断。

```go
type Worker struct {
	onDone func()
}

func NewWorker(onDone func()) Worker {
	if onDone == nil {
		onDone = func() {}
	}
	return Worker{onDone: onDone}
}
```

不要直接调用 nil func：

```go
var f func()
// f() // panic
```

## 5. 为什么 `var err error = (*MyErr)(nil)` 不等于 nil？

接口值包含动态类型和动态值。下面的 `err` 有动态类型 `*MyErr`，所以接口本身不是 nil。

```go
type MyErr struct{}

func (*MyErr) Error() string { return "bad" }

func main() {
	var e *MyErr = nil
	var err error = e

	fmt.Println(e == nil)   // true
	fmt.Println(err == nil) // false
}
```

正确的错误返回方式是没有错误时直接返回 nil 接口：

```go
func do(ok bool) error {
	if ok {
		return nil
	}
	return &MyErr{}
}
```

这类问题在自定义 error、接口返回值和 mock 中非常常见。
