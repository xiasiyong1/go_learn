# 066. map 操作

## 问题

Go map 的查找、删除、零值返回和 `comma ok` 应该怎么理解？

## 先给结论

访问 `m[k]` 时，如果 key 不存在，Go 会返回 value 类型的零值。也正因为如此，单返回值形式无法区分“key 不存在”和“key 存在但值刚好是零值”。需要区分时必须使用 `v, ok := m[k]`。`delete` 删除不存在的 key 是安全的；nil map 可以读和删，但不能写。

## 1. 查找不存在的 key 会返回零值

```go
scores := map[string]int{
	"alice": 0,
}

fmt.Println(scores["bob"])   // 0
fmt.Println(scores["alice"]) // 0
```

只看返回值，无法知道 `bob` 不存在，还是他的分数就是 0。

这时要用 `comma ok`：

```go
score, ok := scores["bob"]
if !ok {
	fmt.Println("bob not found")
} else {
	fmt.Println("bob score:", score)
}
```

## 2. 什么时候可以利用零值

计数场景非常适合利用零值。

```go
counts := map[string]int{}

for _, word := range []string{"go", "go", "java"} {
	counts[word]++
}

fmt.Println(counts) // map[go:2 java:1]
```

因为不存在的 key 返回 `0`，所以 `counts[word]++` 可以直接工作。

布尔集合也可以利用零值，但要看语义：

```go
seen := map[string]bool{}

if !seen["task-1"] {
	seen["task-1"] = true
}
```

如果你只关心存在性，`map[string]struct{}` 更省语义歧义：

```go
set := map[string]struct{}{}
set["task-1"] = struct{}{}

_, ok := set["task-1"]
fmt.Println(ok)
```

## 3. `delete` 不需要先判断 key 是否存在

```go
m := map[string]int{"a": 1}

delete(m, "a")
delete(m, "missing") // 安全，不会 panic

fmt.Println(m)
```

所以这种代码是多余的：

```go
if _, ok := m[key]; ok {
	delete(m, key)
}
```

只有当你需要根据“是否存在”做其他逻辑时，才需要先查。

## 4. nil map 可以读和删，不能写

```go
var m map[string]int

fmt.Println(m["x"]) // 0
delete(m, "x")      // 安全

// m["x"] = 1       // panic: assignment to entry in nil map
```

要写入 map，必须初始化：

```go
m = make(map[string]int)
m["x"] = 1
```

这一点和 nil slice 不同：nil slice 可以 `append`，nil map 不能直接赋值。

## 5. map 取出来的是值副本

如果 map 的 value 是结构体，不能直接修改字段。

```go
type User struct {
	Name string
	Age  int
}

users := map[string]User{
	"a": {Name: "alice", Age: 18},
}

// users["a"].Age = 19 // 编译失败：cannot assign to struct field in map
```

正确写法是取出、修改、写回：

```go
u := users["a"]
u.Age = 19
users["a"] = u
```

或者把 value 设计成指针：

```go
users2 := map[string]*User{
	"a": {Name: "alice", Age: 18},
}

users2["a"].Age = 19
```

指针写法方便修改，但会引入共享可变状态，需要考虑 nil、并发和对象生命周期。

## 6. 面试时怎么答

可以这样回答：

- `m[k]` 不存在时返回 value 类型零值。
- 如果要区分“不存在”和“存在但值为零值”，使用 `v, ok := m[k]`。
- 计数类场景可以直接利用零值，如 `m[k]++`。
- `delete` 删除不存在 key 是安全的。
- nil map 可以读和 delete，但写入会 panic。
- map value 是结构体时，取出来是副本，修改字段要取出后写回，或存指针。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `m[k]` 和 `v, ok := m[k]` 应该怎么选？
- nil map、空 map 和已经初始化的 map 行为有什么区别？
- 为什么 map 中结构体字段不能直接修改？
- `map[string]bool` 和 `map[string]struct{}` 怎么选？
- 遍历 map 时删除 key 是否安全？
