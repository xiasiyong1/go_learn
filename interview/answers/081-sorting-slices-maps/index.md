# 081. 排序

## 问题

Go 中 slice 排序、map 稳定输出和自定义比较应该怎么做？

## 先给结论

slice 可以原地排序，map 不能直接排序，因为 map 遍历顺序不稳定。需要稳定输出时，先取出 key 排序，再按 key 访问 map。自定义比较函数必须满足一致性：相等时返回 0 或 false，不能写成“<=”这种会破坏排序约定的逻辑。

## 1. 排序会修改原 slice

```go
nums := []int{3, 1, 2}
slices.Sort(nums)

fmt.Println(nums) // [1 2 3]
```

如果调用方还需要原顺序，先复制。

```go
nums := []int{3, 1, 2}
sorted := slices.Clone(nums)
slices.Sort(sorted)

fmt.Println(nums)   // [3 1 2]
fmt.Println(sorted) // [1 2 3]
```

面试时要主动说：排序通常是原地操作，是否能改原 slice 是 API 设计的一部分。

## 2. 自定义排序

按年龄排序：

```go
type User struct {
	Name string
	Age  int
}

users := []User{
	{Name: "a", Age: 20},
	{Name: "b", Age: 18},
}

slices.SortFunc(users, func(a, b User) int {
	return cmp.Compare(a.Age, b.Age)
})
```

多字段排序：

```go
slices.SortFunc(users, func(a, b User) int {
	if n := cmp.Compare(a.Age, b.Age); n != 0 {
		return n
	}
	return cmp.Compare(a.Name, b.Name)
})
```

如果使用旧版本 Go，也可以用 `sort.Slice`：

```go
sort.Slice(users, func(i, j int) bool {
	return users[i].Age < users[j].Age
})
```

## 3. 比较函数不能写错

错误写法：

```go
sort.Slice(nums, func(i, j int) bool {
	return nums[i] <= nums[j] // 错：相等时也返回 true
})
```

比较函数应表达严格小于。

```go
sort.Slice(nums, func(i, j int) bool {
	return nums[i] < nums[j]
})
```

对 `slices.SortFunc`，相等时应该返回 0。

```go
slices.SortFunc(users, func(a, b User) int {
	return cmp.Compare(a.Age, b.Age)
})
```

比较逻辑不一致时，排序结果不可预期。

## 4. 稳定排序和不稳定排序

普通排序不保证相等元素保持原相对顺序。稳定排序会保留相等元素的原顺序。

```go
type Item struct {
	Group string
	Name  string
}

items := []Item{
	{Group: "a", Name: "first"},
	{Group: "a", Name: "second"},
}

sort.SliceStable(items, func(i, j int) bool {
	return items[i].Group < items[j].Group
})
```

如果相同 Group 内的原顺序有业务意义，用稳定排序。

## 5. map 要稳定输出，先排序 key

不要依赖 map 遍历顺序。

```go
m := map[string]int{
	"b": 2,
	"a": 1,
	"c": 3,
}

for k, v := range m {
	fmt.Println(k, v) // 顺序不保证
}
```

稳定输出：

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
slices.Sort(keys)

for _, k := range keys {
	fmt.Println(k, m[k])
}
```

这在测试、签名、缓存 key、配置输出里非常重要。

## 6. 数字字符串排序要注意语义

字符串排序是字典序：

```go
s := []string{"2", "10", "1"}
slices.Sort(s)
fmt.Println(s) // [1 10 2]
```

如果业务要按数字排序，就转换成数字比较。

```go
slices.SortFunc(s, func(a, b string) int {
	ai, _ := strconv.Atoi(a)
	bi, _ := strconv.Atoi(b)
	return cmp.Compare(ai, bi)
})

fmt.Println(s) // [1 2 10]
```

真实代码里要处理转换错误，不要像示例里忽略。

## 7. 面试时怎么答

可以这样回答：

- slice 排序通常原地修改，保留原顺序要先复制。
- 基础类型可用 `slices.Sort`，结构体用 `slices.SortFunc` 或 `sort.Slice`。
- 比较函数必须一致，相等时不能返回“小于”。
- 需要保留相等元素原顺序时用稳定排序。
- map 遍历顺序不稳定，稳定输出要先取 key、排序、再访问 map。
- 字符串排序和数字排序语义不同。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么排序前有时要先复制 slice？
- `sort.Slice` 的 less 函数为什么不能写 `<=`？
- 稳定排序解决什么问题？
- map 为什么不能直接排序？
- 排序 map 输出时有哪些性能和正确性取舍？
