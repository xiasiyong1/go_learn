# 009. nil 切片和空切片

## 问题

nil 切片和空切片有什么区别？

## 先给结论

nil 切片和空切片的长度都是 0，都可以 `range`，也都可以 `append`。主要区别在于：

- nil 切片等于 `nil`，空切片不等于 `nil`。
- nil 切片没有底层数组；空切片通常有一个非 nil 的数据指针，但不能依赖这个细节写业务逻辑。
- JSON 编码不同：nil slice 是 `null`，empty slice 是 `[]`。
- API 语义可能不同：nil 可以表示“未设置”，empty 可以表示“已设置但为空”。

```go
var a []int
b := []int{}
c := make([]int, 0)

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false
fmt.Println(c == nil) // false

fmt.Println(len(a), len(b), len(c)) // 0 0 0
```

## 行为上有哪些相同点

都可以取长度：

```go
var a []int
b := []int{}

fmt.Println(len(a)) // 0
fmt.Println(len(b)) // 0
```

都可以遍历：

```go
for _, v := range a {
	fmt.Println(v) // 不会执行
}

for _, v := range b {
	fmt.Println(v) // 不会执行
}
```

都可以追加：

```go
a = append(a, 1)
b = append(b, 1)

fmt.Println(a) // [1]
fmt.Println(b) // [1]
```

所以在内部业务逻辑里，如果只是表达“没有元素”，通常用 `len(s) == 0`，不要用 `s == nil`。

## JSON 编码差异

这是面试和工程里最常见的差异。

```go
var a []int
b := []int{}

aj, _ := json.Marshal(a)
bj, _ := json.Marshal(b)

fmt.Println(string(aj)) // null
fmt.Println(string(bj)) // []
```

如果对外 API 要求空列表返回 `[]`，就要确保字段是空切片而不是 nil 切片。

```go
type Response struct {
	Items []string `json:"items"`
}

func List() Response {
	return Response{
		Items: []string{},
	}
}
```

编码结果：

```json
{"items":[]}
```

如果 `Items` 是 nil，结果会是：

```json
{"items":null}
```

## `omitempty` 的影响

`omitempty` 会把长度为 0 的 slice 省略掉。nil slice 和 empty slice 都会被省略。

```go
type Response struct {
	Items []string `json:"items,omitempty"`
}

var a = Response{Items: nil}
var b = Response{Items: []string{}}

aj, _ := json.Marshal(a)
bj, _ := json.Marshal(b)

fmt.Println(string(aj)) // {}
fmt.Println(string(bj)) // {}
```

如果接口契约要求一定返回 `items: []`，就不要给这个字段加 `omitempty`。

## 什么时候返回 nil slice

内部函数里，返回 nil slice 往往最简单，也符合零值习惯。

```go
func FindEven(nums []int) []int {
	var out []int
	for _, n := range nums {
		if n%2 == 0 {
			out = append(out, n)
		}
	}
	return out
}
```

调用方可以直接：

```go
for _, n := range FindEven(nums) {
	fmt.Println(n)
}
```

没有偶数也不会 panic。

## 什么时候返回 empty slice

对外 API、前端接口、SDK 返回值通常更倾向 empty slice，因为调用方看到 `[]` 更像“空列表”，不需要额外处理 `null`。

```go
func ListUsers() []User {
	users := queryUsers()
	if users == nil {
		return []User{}
	}
	return users
}
```

更好的做法是在边界层统一处理，而不是让业务层每个函数都强行 `make([]T, 0)`：

```go
func writeUsers(w http.ResponseWriter, users []User) error {
	if users == nil {
		users = []User{}
	}
	return json.NewEncoder(w).Encode(struct {
		Users []User `json:"users"`
	}{
		Users: users,
	})
}
```

这样内部逻辑保持自然，外部契约保持稳定。

## nil 和 empty 可以表达不同语义

有时 nil 和 empty 都有业务意义：

```go
type QueryResult struct {
	Loaded bool
	Items  []Item
}
```

这里更推荐显式加 `Loaded` 字段，而不是只靠 `Items == nil` 表达“未加载”。

如果确实约定 nil 表示未加载、empty 表示已加载但为空，要把这个约定写进 API 文档和测试里。

```go
var notLoaded []Item = nil
loadedEmpty := []Item{}

fmt.Println(notLoaded == nil)  // true
fmt.Println(loadedEmpty == nil) // false
```

## reflect.DeepEqual 的差异

有些测试会踩这个坑：

```go
var a []int
b := []int{}

fmt.Println(reflect.DeepEqual(a, b)) // false
```

如果业务只关心元素是否为空，测试里应该比较长度或使用更符合语义的断言。

```go
if len(got) != 0 {
	t.Fatalf("expected empty result, got %v", got)
}
```

如果 API 契约要求必须返回 empty slice 而不是 nil slice，那才应该明确断言 `got != nil`。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- nil slice 和 empty slice 在 `len`、`cap`、`append`、`range` 上有什么区别？
- JSON 中为什么一个是 `null`，一个是 `[]`？
- `omitempty` 会如何处理 nil slice 和 empty slice？
- 为什么内部逻辑常用 nil slice，而对外 API 常用 empty slice？
- `reflect.DeepEqual(nilSlice, emptySlice)` 为什么是 false？
- 什么时候应该用 `s == nil`，什么时候应该用 `len(s) == 0`？
