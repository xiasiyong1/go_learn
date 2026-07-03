# 026. map key 和遍历顺序 - 面试追问

## 1. map key 为什么必须可比较？

map 需要用 key 做哈希和相等判断。

```go
m := map[string]int{}
m["go"] = 1
fmt.Println(m["go"])
```

查找 `"go"` 时，运行时要先用它计算哈希，再判断 bucket 里某个 key 是否等于 `"go"`。

slice、map、func 没有语言定义的通用相等关系，所以不能作为 key：

```go
// map[[]int]string{} // 编译错误
```

## 2. 结构体什么时候能做 map key？

所有字段都可比较时，结构体可以做 key。

```go
type Key struct {
	TenantID int64
	UserID   int64
}

m := map[Key]string{}
m[Key{1, 2}] = "Tom"
```

如果加了不可比较字段，就不行：

```go
type BadKey struct {
	TenantID int64
	UserID   int64
	Tags     []string
}

// map[BadKey]string{} // 编译错误
```

公共 key 类型新增字段要谨慎，因为这可能破坏调用方编译。

## 3. `[]byte` 内容想做 key，有哪些方案？

常见方案一：转 string。

```go
m := map[string]int{}
b := []byte("hello")
m[string(b)]++
```

优点是简单；缺点是可能有转换成本。

方案二：固定长度用数组。

```go
var key [32]byte
copy(key[:], sum)

m := map[[32]byte]int{}
m[key]++
```

方案三：多个字段用结构体。

```go
type Key struct {
	Method string
	Path   string
}
```

不要用不规范拼接制造冲突：

```go
key := a + ":" + b
```

如果 `a` 或 `b` 本身包含 `:`，就可能产生歧义。

## 4. 指针做 key 时比较的是地址还是内容？

比较的是地址。

```go
a := &User{ID: 1}
b := &User{ID: 1}

m := map[*User]string{
	a: "first",
	b: "second",
}

fmt.Println(len(m)) // 2
```

如果业务上只看 `ID`，就用 `ID` 做 key：

```go
m := map[int64]string{}
m[a.ID] = "first"
m[b.ID] = "second"

fmt.Println(len(m)) // 1
```

指针 key 适合表达对象身份，不适合表达内容相等。

## 5. map 遍历顺序为什么不能用于稳定输出？

语言不保证 map 遍历顺序。

```go
for k, v := range m {
	fmt.Println(k, v)
}
```

这段代码的输出顺序不能作为测试期望、签名输入或分页依据。

稳定输出要排序：

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

## 6. 需要稳定 JSON、签名或测试输出时应该怎么做？

核心是先定义顺序。

签名：

```go
func Canonical(m map[string]string) string {
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	var b strings.Builder
	for _, k := range keys {
		b.WriteString(k)
		b.WriteByte('=')
		b.WriteString(m[k])
		b.WriteByte('&')
	}
	return b.String()
}
```

测试输出也一样，不要直接比较 range map 拼出来的字符串。先排序，再断言结果。

如果 map 很大，排序是 `O(n log n)`，要结合调用频率评估成本；但正确性上必须先稳定顺序。
