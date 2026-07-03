# 066. map 操作 - 面试追问

## 1. `m[k]` 和 `v, ok := m[k]` 应该怎么选？

如果业务上只需要值，并且零值就是合理默认值，可以直接用 `m[k]`。

```go
counts := map[string]int{}
counts["go"]++

fmt.Println(counts["go"])   // 1
fmt.Println(counts["java"]) // 0，合理默认值
```

如果零值也是合法业务值，就必须用 `comma ok`。

```go
ages := map[string]int{
	"baby": 0,
}

age, ok := ages["bob"]
if !ok {
	fmt.Println("not found")
} else {
	fmt.Println(age)
}
```

判断标准不是类型，而是业务语义：零值能不能代表“默认值”。

## 2. nil map、空 map 和已经初始化的 map 行为有什么区别？

nil map 没有底层哈希表。

```go
var nilMap map[string]int
fmt.Println(nilMap == nil) // true
fmt.Println(nilMap["x"])   // 0
delete(nilMap, "x")        // 安全

// nilMap["x"] = 1         // panic
```

空 map 已经初始化，可以写。

```go
empty := map[string]int{}
fmt.Println(empty == nil) // false

empty["x"] = 1
fmt.Println(empty)
```

`make` 创建的 map 也是已初始化 map：

```go
m := make(map[string]int, 100)
m["x"] = 1
```

容量参数只是提示，不是长度，也不能用 `len` 看到容量。

## 3. 为什么 map 中结构体字段不能直接修改？

map 查找返回的是 value 的副本，不是稳定地址。map 扩容时元素位置可能变化，所以 Go 不允许你直接取 map 元素字段地址或修改字段。

```go
type Counter struct {
	N int
}

m := map[string]Counter{
	"a": {N: 1},
}

// m["a"].N++ // 编译失败
```

取出、修改、写回：

```go
c := m["a"]
c.N++
m["a"] = c
```

如果频繁修改，可以存指针：

```go
mp := map[string]*Counter{
	"a": {N: 1},
}

mp["a"].N++
```

但指针 map 要处理 key 不存在时的 nil 问题：

```go
if c := mp["missing"]; c != nil {
	c.N++
}
```

## 4. `map[string]bool` 和 `map[string]struct{}` 怎么选？

如果 value 的真假有业务含义，用 `map[string]bool`。

```go
enabled := map[string]bool{
	"feature-a": false, // 明确配置为 false
}

v, ok := enabled["feature-a"]
fmt.Println(v, ok) // false true
```

如果只表示集合存在性，用 `map[string]struct{}` 更准确。

```go
set := map[string]struct{}{
	"alice": {},
}

_, ok := set["alice"]
fmt.Println(ok)
```

`struct{}` 不占额外数据空间，更重要的是语义清楚：只关心 key 是否存在。

## 5. 遍历 map 时删除 key 是否安全？

在同一个 goroutine 中，遍历 map 时删除 key 是允许的。

```go
m := map[string]int{
	"a": 1,
	"b": 2,
	"c": 3,
}

for k, v := range m {
	if v%2 == 1 {
		delete(m, k)
	}
}

fmt.Println(m)
```

但遍历时新增 key，不应该依赖新 key 是否会被遍历到。

```go
for k := range m {
	m[k+"x"] = 100 // 是否在本轮遍历到，不要依赖
}
```

并发场景是另一个问题：一个 goroutine 遍历，另一个 goroutine 写 map，是数据竞争，可能直接 fatal error。需要锁、`sync.Map` 或改变所有权模型。
