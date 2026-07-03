# 006. Go GC 基础

## 问题

Go 的垃圾回收大致怎么工作？写代码时哪些行为会增加 GC 压力？

## 先给结论

Go 的 GC 主要是并发标记清扫。它回收的是“从根对象出发不可达”的堆对象，不是程序员主观上认为“不再使用”的对象。

写代码时真正影响 GC 压力的通常是三件事：

- 分配率：单位时间内分配了多少对象、多少字节。
- 存活堆：GC 后仍然可达的对象有多少。
- 指针扫描成本：对象里有多少指针需要 GC 扫描。

面试时不要只说“Go 有 GC，不用手动释放”。更好的回答是：先说明 GC 怎么判断对象可回收，再说明哪些代码会制造堆分配和长期引用，最后说明如何用 benchmark、pprof、gctrace 验证。

## GC 回收的是什么

GC 从根对象开始找所有可达对象。常见根对象包括：

- 全局变量。
- 当前 goroutine 栈上的变量。
- 寄存器里保存的指针。
- runtime 内部结构持有的指针。

从这些根对象沿着指针能找到的对象，就是可达对象。可达对象不会被回收；找不到的对象才会在清扫阶段回收。

这就是为什么“我后面不用了”不等于“GC 可以回收”：

```go
var cache = map[string][]byte{}

func Load(id string) []byte {
	data := make([]byte, 10<<20) // 10 MB
	cache[id] = data
	return data
}
```

只要 `cache` 还引用着 `data`，这 10 MB 就仍然可达，GC 不会回收。问题不是 GC 不工作，而是程序还保留着引用。

## Go GC 大致怎么工作

可以按四个阶段理解：

1. 标记准备：短暂停顿，启动本轮标记。
2. 并发标记：用户 goroutine 继续运行，GC 同时从根对象开始标记可达对象。
3. 标记终止：短暂停顿，完成剩余标记工作。
4. 清扫：回收未标记对象占用的内存，供后续分配复用。

并发标记期间，用户代码还在修改指针关系。为了保证标记正确，Go 使用写屏障协助 GC 维护对象可达性。

面试讲到这里就足够了。除非面试官继续深挖，否则初学者不用背复杂术语；但要知道 GC 不是“停下整个程序慢慢扫完”，而是尽量把大部分工作和用户代码并发执行。

## 哪些代码会增加 GC 压力

### 1. 热路径频繁创建临时对象

错误示例：

```go
func Build(items []string) string {
	s := ""
	for _, item := range items {
		s += item + ","
	}
	return s
}
```

字符串不可变，循环里反复 `+` 会创建很多临时字符串。

更好的写法：

```go
func Build(items []string) string {
	var b strings.Builder
	for _, item := range items {
		b.WriteString(item)
		b.WriteByte(',')
	}
	return b.String()
}
```

如果能估计容量，还可以预分配：

```go
func Build(items []string) string {
	var size int
	for _, item := range items {
		size += len(item) + 1
	}

	var b strings.Builder
	b.Grow(size)
	for _, item := range items {
		b.WriteString(item)
		b.WriteByte(',')
	}
	return b.String()
}
```

### 2. slice 扩容太多

```go
func IDs(users []User) []int64 {
	var ids []int64
	for _, u := range users {
		ids = append(ids, u.ID)
	}
	return ids
}
```

如果知道最终长度，可以直接预分配：

```go
func IDs(users []User) []int64 {
	ids := make([]int64, 0, len(users))
	for _, u := range users {
		ids = append(ids, u.ID)
	}
	return ids
}
```

这不只是少几次扩容，也会减少临时底层数组的分配。

### 3. 小切片引用大数组

这是“对象仍然可达”的典型例子。

```go
func Header(data []byte) []byte {
	return data[:16]
}
```

如果 `data` 是 100 MB，返回的 16 字节切片仍然引用同一个底层数组。只要返回值还活着，整个 100 MB 数组都可能无法回收。

如果只需要保留小片段，可以复制出来：

```go
func Header(data []byte) []byte {
	header := make([]byte, 16)
	copy(header, data[:16])
	return header
}
```

这会多一次小分配，但可以释放大数组引用。面试时要说明这是取舍：不是所有切片都要复制，只有当小切片长期存活且底层数组很大时才值得。

### 4. 不必要的装箱到 interface

把具体值传给接口，有时会导致额外分配，尤其是值逃逸到堆上时。

```go
func Log(v any) {
	fmt.Println(v)
}

func Handle(n int) {
	Log(n)
}
```

这不代表“不能用接口”，而是要知道接口、闭包、返回指针、把局部变量放进全局容器，都可能让对象逃逸。可以用逃逸分析观察：

```sh
go build -gcflags=-m ./...
```

看到 `escapes to heap` 时，不要立刻改代码；先结合 benchmark 判断它是否真在热点路径。

### 5. 长生命周期容器持有引用

```go
type Queue struct {
	items []*Request
}

func (q *Queue) Pop() *Request {
	x := q.items[0]
	q.items = q.items[1:]
	return x
}
```

这个写法可能让旧底层数组仍然引用已弹出的 `*Request`，导致对象不能及时回收。

可以在弹出时清掉引用：

```go
func (q *Queue) Pop() *Request {
	x := q.items[0]
	q.items[0] = nil
	q.items = q.items[1:]
	return x
}
```

对于长期存在的队列、缓存、连接池，这类引用残留比一次小分配更值得关注。

## `sync.Pool` 什么时候用

`sync.Pool` 可以复用临时对象，降低分配率。典型场景是复用 `bytes.Buffer`。

```go
var bufferPool = sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func Encode(v any) string {
	buf := bufferPool.Get().(*bytes.Buffer)
	buf.Reset()
	defer bufferPool.Put(buf)

	_ = json.NewEncoder(buf).Encode(v)
	return buf.String()
}
```

但它不是普通对象池：

- 池里的对象可能在任意 GC 后被清掉。
- 放回池前必须重置状态。
- 不适合管理连接、文件这类需要确定关闭的资源。
- 没有性能数据时，不要为了“看起来省内存”滥用。

## 不要靠手动 `runtime.GC()` 解决问题

手动触发 GC 通常不是好办法：

```go
func Bad() {
	// 做完一批任务后强制 GC
	runtime.GC()
}
```

它可能增加 CPU 和停顿，掩盖真正问题。更合理的路径是：

- 找出高分配路径。
- 判断对象是否真的需要长期存活。
- 减少不必要分配或缩短引用生命周期。
- 必要时再调 `GOGC` 或内存限制参数。

## 如何验证

常用工具：

```sh
go test -bench . -benchmem ./...
go test -run '^$' -bench BenchmarkBuild -benchmem
go build -gcflags=-m ./...
GODEBUG=gctrace=1 go test ./...
go tool pprof http://localhost:6060/debug/pprof/heap
```

看指标时不要只看“内存大不大”，要区分：

- 分配总量高：可能是临时对象太多。
- 存活堆高：可能是缓存、长生命周期对象或泄漏。
- GC 频繁：可能是分配率高，也可能是 `GOGC` 设置过低。
- 暂停时间高：要结合堆大小、对象图和运行时指标分析。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- GC 如何判断一个对象可以回收？
- 分配率和存活堆对 GC 压力有什么不同影响？
- 为什么小切片可能导致大数组不能释放？
- `sync.Pool` 适合什么场景，不适合什么场景？
- 逃逸到堆上一定是坏事吗？
- 看到内存上涨时，如何区分缓存、泄漏和正常分配？
