# 081. 排序 - 面试追问

## 1. 为什么排序前有时要先复制 slice？

因为排序会原地修改 slice。

```go
func SortedCopy(nums []int) []int {
	out := slices.Clone(nums)
	slices.Sort(out)
	return out
}
```

如果函数名叫 `SortUsers`，调用方可能接受它修改原 slice；如果函数名叫 `SortedUsers`，通常更像返回新结果。API 命名要和是否修改原数据一致。

错误示例：

```go
func Top(nums []int) int {
	slices.Sort(nums) // 悄悄改变调用方的 nums
	return nums[len(nums)-1]
}
```

更稳：

```go
func Top(nums []int) int {
	cp := slices.Clone(nums)
	slices.Sort(cp)
	return cp[len(cp)-1]
}
```

## 2. `sort.Slice` 的 less 函数为什么不能写 `<=`？

less 要表达严格小于。相等时应返回 false。

错误：

```go
sort.Slice(nums, func(i, j int) bool {
	return nums[i] <= nums[j]
})
```

当两个元素相等时，`less(i, j)` 和 `less(j, i)` 都可能为 true，排序算法的基本假设被破坏。

正确：

```go
sort.Slice(nums, func(i, j int) bool {
	return nums[i] < nums[j]
})
```

`slices.SortFunc` 里，相等返回 0：

```go
slices.SortFunc(users, func(a, b User) int {
	return cmp.Compare(a.Age, b.Age)
})
```

## 3. 稳定排序解决什么问题？

稳定排序保证“比较相等”的元素保持原来的相对顺序。

```go
items := []Item{
	{Group: "a", Name: "first"},
	{Group: "b", Name: "x"},
	{Group: "a", Name: "second"},
}

sort.SliceStable(items, func(i, j int) bool {
	return items[i].Group < items[j].Group
})
```

排序后两个 Group 为 `a` 的元素仍保持 `first` 在 `second` 前面。

如果相等元素内部顺序无所谓，普通排序即可；如果 UI 展示、分页、日志输出依赖原顺序，用稳定排序。

## 4. map 为什么不能直接排序？

map 是哈希表，语言不保证遍历顺序。排序算法需要可索引的序列，而 map 不是序列。

错误思路：

```go
// sort(m) // 不存在这种操作
```

正确做法是排序 key：

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

如果 key 是结构体，要定义清楚排序字段。

## 5. 排序 map 输出时有哪些性能和正确性取舍？

正确性方面，签名、缓存 key、测试 golden 文件必须稳定排序。

```go
func StableString(m map[string]string) string {
	keys := make([]string, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	slices.Sort(keys)

	var b strings.Builder
	for _, k := range keys {
		b.WriteString(k)
		b.WriteByte('=')
		b.WriteString(m[k])
		b.WriteByte(';')
	}
	return b.String()
}
```

性能方面，排序 key 是 O(n log n)，还要额外分配 key slice。小 map 无所谓，大 map 高频输出要考虑缓存排序结果、改变数据结构，或避免在热路径生成稳定文本。
