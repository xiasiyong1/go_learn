# 026. map key 和遍历顺序

## 问题

map 的 key 为什么必须可比较？遍历顺序为什么是随机的？

## 先给结论

map key 必须可比较，因为哈希表需要对 key 做两件事：

- 根据 key 计算哈希，定位可能的 bucket。
- 在 bucket 内判断某个 key 是否等于目标 key。

slice、map、func 没有语言定义的通用相等关系，所以不能作为 map key。

map 遍历顺序不保证稳定。语言故意不承诺顺序，避免程序依赖运行时内部实现。需要稳定输出时，自己排序 key。

## 什么类型能做 key

可以：

```go
map[string]int{}
map[int64]string{}
map[[16]byte]User{}
```

字段都可比较的结构体也可以：

```go
type UserKey struct {
	TenantID int64
	UserID   int64
}

visits := map[UserKey]int{}
visits[UserKey{TenantID: 1, UserID: 2}]++
```

不可以：

```go
// map[[]byte]int{}        // slice 不可比较
// map[map[string]int]int{} // map 不可比较
// map[func()]int{}        // func 不可比较
```

结构体中只要有不可比较字段，就不能做 key：

```go
type BadKey struct {
	ID   int64
	Tags []string
}

// map[BadKey]int{} // 编译错误
```

## slice 内容想做 key 怎么办

如果是 `[]byte`，可以转成 string：

```go
func Count(chunks [][]byte) map[string]int {
	m := make(map[string]int)
	for _, chunk := range chunks {
		m[string(chunk)]++
	}
	return m
}
```

如果长度固定，可以用数组：

```go
var id [16]byte
copy(id[:], rawID)

m := map[[16]byte]User{}
m[id] = user
```

如果是多个字段，优先结构体 key：

```go
type CacheKey struct {
	Method string
	Path   string
	Lang   string
}
```

不要随便用字符串拼接：

```go
key := method + ":" + path + ":" + lang
```

如果字段本身可能包含分隔符，就可能冲突。要么用结构体 key，要么使用明确的编码规则。

## 指针做 key 比较的是地址

```go
a := &User{ID: 1}
b := &User{ID: 1}

m := map[*User]string{}
m[a] = "a"
m[b] = "b"

fmt.Println(len(m)) // 2
```

`a` 和 `b` 内容一样，但地址不同，所以是两个 key。

如果业务语义是按 `ID` 判断同一个用户，key 应该用 ID：

```go
m := map[int64]string{}
m[a.ID] = "a"
m[b.ID] = "b"

fmt.Println(len(m)) // 1
```

## map 遍历顺序不稳定

```go
m := map[string]int{
	"a": 1,
	"b": 2,
	"c": 3,
}

for k, v := range m {
	fmt.Println(k, v)
}
```

不要依赖输出顺序。即使某次运行看起来固定，也不是语言保证。

需要稳定输出：

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
sort.Strings(keys)

for _, k := range keys {
	fmt.Println(k, m[k])
}
```

这对测试、签名、缓存 key、日志稳定输出都很重要。

## 不要用 map 遍历做分页或签名

错误示例：

```go
func Sign(m map[string]string) string {
	var b strings.Builder
	for k, v := range m {
		b.WriteString(k)
		b.WriteString("=")
		b.WriteString(v)
		b.WriteString("&")
	}
	return hash(b.String())
}
```

同一个 map 可能得到不同字符串，签名就不稳定。

正确做法：

```go
func Sign(m map[string]string) string {
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	var b strings.Builder
	for _, k := range keys {
		b.WriteString(k)
		b.WriteString("=")
		b.WriteString(m[k])
		b.WriteString("&")
	}
	return hash(b.String())
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- map key 为什么必须可比较？
- 结构体什么时候能做 map key？
- `[]byte` 内容想做 key，有哪些方案？
- 指针做 key 时比较的是地址还是内容？
- map 遍历顺序为什么不能用于稳定输出？
- 需要稳定 JSON、签名或测试输出时应该怎么做？
