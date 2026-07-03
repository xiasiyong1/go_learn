# 009. nil 切片和空切片 - 面试追问

## 1. nil slice 和 empty slice 在 `len`、`cap`、`append`、`range` 上有什么区别？

大多数普通操作没有区别。

```go
var a []int
b := []int{}

fmt.Println(a == nil) // true
fmt.Println(b == nil) // false

fmt.Println(len(a), cap(a)) // 0 0
fmt.Println(len(b), cap(b)) // 0 0

for range a {
	panic("不会执行")
}
for range b {
	panic("不会执行")
}

a = append(a, 1)
b = append(b, 1)

fmt.Println(a) // [1]
fmt.Println(b) // [1]
```

真正的区别主要在 `== nil`、序列化、反射比较和 API 语义上。

## 2. JSON 中为什么一个是 `null`，一个是 `[]`？

因为 encoding/json 会保留 nil slice 和非 nil 空 slice 的状态。

```go
var nilSlice []int
emptySlice := []int{}

a, _ := json.Marshal(nilSlice)
b, _ := json.Marshal(emptySlice)

fmt.Println(string(a)) // null
fmt.Println(string(b)) // []
```

放在结构体里也一样：

```go
type Response struct {
	Items []int `json:"items"`
}

json.Marshal(Response{Items: nil})     // {"items":null}
json.Marshal(Response{Items: []int{}}) // {"items":[]}
```

对外接口如果承诺返回数组，通常应该返回 `[]`，避免客户端同时处理 `null` 和数组两种类型。

## 3. `omitempty` 会如何处理 nil slice 和 empty slice？

都会省略，因为 slice 长度都是 0。

```go
type Response struct {
	Items []int `json:"items,omitempty"`
}

a, _ := json.Marshal(Response{Items: nil})
b, _ := json.Marshal(Response{Items: []int{}})

fmt.Println(string(a)) // {}
fmt.Println(string(b)) // {}
```

所以如果接口要求空列表也必须出现：

```json
{"items":[]}
```

字段就不要加 `omitempty`，并且编码前要保证它是 empty slice：

```go
func Normalize(items []int) []int {
	if items == nil {
		return []int{}
	}
	return items
}
```

## 4. 为什么内部逻辑常用 nil slice，而对外 API 常用 empty slice？

内部逻辑里 nil slice 更简单：

```go
func Collect(nums []int) []int {
	var out []int
	for _, n := range nums {
		if n > 0 {
			out = append(out, n)
		}
	}
	return out
}
```

没有正数时返回 nil slice，但调用方仍然可以 `len`、`range`、`append`。

对外 API 关注契约稳定。很多前端或其他语言客户端更希望空列表就是 `[]`：

```go
func WriteJSON(w http.ResponseWriter, items []Item) error {
	if items == nil {
		items = []Item{}
	}
	return json.NewEncoder(w).Encode(struct {
		Items []Item `json:"items"`
	}{
		Items: items,
	})
}
```

边界层统一转换，比业务层到处 `make([]T, 0)` 更清晰。

## 5. `reflect.DeepEqual(nilSlice, emptySlice)` 为什么是 false？

因为它会区分 nil slice 和非 nil empty slice。

```go
var a []int
b := []int{}

fmt.Println(reflect.DeepEqual(a, b)) // false
```

测试时要按业务语义选择断言。

如果只关心没有元素：

```go
if len(got) != 0 {
	t.Fatalf("expected no items, got %v", got)
}
```

如果关心 JSON 契约必须是 `[]`：

```go
if got == nil {
	t.Fatal("expected empty slice, got nil")
}
```

这道追问的重点是：测试应该表达契约，而不是无意识地把实现细节固定住。

## 6. 什么时候应该用 `s == nil`，什么时候应该用 `len(s) == 0`？

判断“有没有元素”，用 `len(s) == 0`。

```go
if len(users) == 0 {
	return "no users"
}
```

判断“有没有初始化”或“是否未加载”，才考虑 `s == nil`。

```go
type Result struct {
	Items []Item
}

func (r Result) Loaded() bool {
	return r.Items != nil
}
```

不过更清楚的设计通常是显式字段：

```go
type Result struct {
	Loaded bool
	Items  []Item
}
```

这样比让调用方猜 `nil` 的业务含义更稳。
