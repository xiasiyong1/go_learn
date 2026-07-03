# 024. 泛型

## 问题

Go 泛型解决什么问题？什么时候不应该用泛型？

## 先给结论

泛型解决的是“同一套逻辑适用于一组类型”的问题，让代码在保留静态类型检查的同时减少重复。

适合泛型的场景：

- 容器和集合操作。
- 通用算法。
- 类型安全的缓存、队列、栈。
- 对多个类型做相同操作，并且操作能用约束清楚表达。

不适合泛型的场景：

- 只是为了炫技。
- 业务逻辑本来就不重复。
- 需要动态派发行为时，接口更自然。
- 约束写得比业务逻辑还复杂。

## 泛型函数

```go
func Index[T comparable](items []T, target T) int {
	for i, item := range items {
		if item == target {
			return i
		}
	}
	return -1
}
```

调用：

```go
fmt.Println(Index([]int{1, 2, 3}, 2))
fmt.Println(Index([]string{"go", "java"}, "go"))
```

`T comparable` 表示 `T` 必须支持 `==`，否则函数体里的 `item == target` 不能编译。

如果写成 `T any`：

```go
func BadIndex[T any](items []T, target T) int {
	for i, item := range items {
		// if item == target { // 编译错误
		// 	return i
		// }
		_ = item
	}
	return -1
}
```

约束不是装饰，它决定函数体里可以对 `T` 做什么操作。

## 泛型类型

```go
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) {
	s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
	if len(s.items) == 0 {
		var zero T
		return zero, false
	}

	last := len(s.items) - 1
	v := s.items[last]
	s.items = s.items[:last]
	return v, true
}
```

使用：

```go
var ints Stack[int]
ints.Push(1)

v, ok := ints.Pop()
fmt.Println(v, ok)
```

这里不需要 `any` 加类型断言，编译器知道 `Stack[int]` 只能放 `int`。

## 类型集合和 `~`

约束可以限制底层类型。

```go
type Integer interface {
	~int | ~int8 | ~int16 | ~int32 | ~int64
}

func Add[T Integer](a, b T) T {
	return a + b
}
```

`~int` 表示“底层类型是 int 的类型”也可以传入。

```go
type UserID int

fmt.Println(Add(UserID(1), UserID(2))) // 可以
```

如果约束写成 `int` 而不是 `~int`，`UserID` 这种新类型就不能传入。

约束过窄会限制复用，约束过宽又无法保证函数体里的操作。好的约束应该刚好表达函数需要的能力。

## 泛型、接口、any 的区别

泛型保留静态类型：

```go
func First[T any](items []T) (T, bool) {
	if len(items) == 0 {
		var zero T
		return zero, false
	}
	return items[0], true
}
```

接口适合行为抽象：

```go
type Reader interface {
	Read([]byte) (int, error)
}

func CopyAll(r Reader) ([]byte, error) {
	return io.ReadAll(r)
}
```

`any` 适合确实未知的动态值：

```go
func LogValue(key string, value any) {
	slog.Info("value", key, value)
}
```

不要把三者混在一起：

- 想复用类型安全算法，用泛型。
- 想依赖行为，用接口。
- 想接收未知值，用 `any`。

## 什么时候不要用泛型

只有两个简单类型，重复代码可能更清楚：

```go
func ParseUserID(s string) (UserID, error) {
	n, err := strconv.ParseInt(s, 10, 64)
	return UserID(n), err
}

func ParseOrderID(s string) (OrderID, error) {
	n, err := strconv.ParseInt(s, 10, 64)
	return OrderID(n), err
}
```

没必要为了合并这两段写出难懂的类型参数和约束。

业务流程也不适合强行泛型化：

```go
func ApproveInvoice(inv Invoice) error
func ApproveExpense(exp Expense) error
```

如果两者规则不同，名字相似不代表应该抽泛型。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 泛型和 `any` 最大区别是什么？
- 约束里的 `comparable` 有什么作用？
- `~int` 和 `int` 作为约束有什么区别？
- 泛型什么时候比接口更合适？
- 什么时候不应该为了减少重复使用泛型？
- 公共泛型 API 的约束为什么要尽量小？
