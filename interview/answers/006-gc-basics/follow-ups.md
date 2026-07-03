# 006. Go GC 基础 - 面试追问

## 1. GC 如何判断一个对象可以回收？

GC 判断的是可达性。

从全局变量、goroutine 栈、寄存器和 runtime 内部结构这些根对象出发，沿着指针能访问到的对象就是可达对象；不可达对象才可以回收。

```go
var global []*User

func Save() {
	u := &User{Name: "Tom"}
	global = append(global, u)
}
```

`Save` 返回后，局部变量 `u` 不在了，但 `global` 还引用着这个 `User`，所以它仍然可达。

如果你想让它可回收，必须移除引用：

```go
func Clear() {
	for i := range global {
		global[i] = nil
	}
	global = global[:0]
}
```

面试里要强调：GC 不是根据“业务上还用不用”来判断，而是根据“从根对象还能不能找到”来判断。

## 2. 分配率和存活堆对 GC 压力有什么不同影响？

分配率高，意味着程序不断制造新对象，GC 需要更频繁地工作。

```go
func Handle(items []string) {
	for _, item := range items {
		_ = []byte(item) // 热路径中频繁分配
	}
}
```

存活堆高，意味着每轮 GC 后仍然有很多对象活着，后续标记和扫描成本会更高。

```go
var cache = map[string][]byte{}

func Put(key string, value []byte) {
	cache[key] = value
}
```

这两个问题的优化方向不同：

- 分配率高：减少临时对象、预分配、复用 buffer。
- 存活堆高：控制缓存大小、缩短引用生命周期、清理不再需要的引用。

所以面试里不要笼统说“减少内存”。要先判断是分配太快，还是活得太久。

## 3. 为什么小切片可能导致大数组不能释放？

slice 只是一个描述符，里面有指向底层数组的指针、长度和容量。返回小切片时，可能仍然引用原来的大数组。

```go
func Prefix(data []byte) []byte {
	return data[:8]
}
```

如果 `data` 来自一个 50 MB 文件，`Prefix` 返回的 8 字节切片仍然指向那块 50 MB 底层数组。

如果这个 prefix 会长期保存，应该复制：

```go
func Prefix(data []byte) []byte {
	p := make([]byte, 8)
	copy(p, data[:8])
	return p
}
```

这个问题的关键是生命周期。短时间使用的小切片不一定要复制；长期保存的小切片引用大数组，才容易造成内存迟迟不能释放。

## 4. `sync.Pool` 适合什么场景，不适合什么场景？

适合复用临时对象，尤其是创建成本较高、在请求结束后就能重置再用的对象。

```go
var pool = sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}

func Format(v any) string {
	buf := pool.Get().(*bytes.Buffer)
	buf.Reset()
	defer pool.Put(buf)

	fmt.Fprint(buf, v)
	return buf.String()
}
```

不适合：

- 需要确定生命周期的资源，比如连接、文件、锁。
- 有复杂状态且容易忘记重置的对象。
- 没有性能瓶颈证据的普通业务对象。

还要知道：`sync.Pool` 里的对象可能在 GC 时被丢弃，所以不能依赖它保存重要状态。

## 5. 逃逸到堆上一定是坏事吗？

不一定。

逃逸只是说明对象生命周期超过了当前栈帧，或者编译器无法证明它可以安全放在栈上。

```go
func NewUser(name string) *User {
	return &User{Name: name}
}
```

这里 `User` 返回给调用方，放到堆上是合理的。

真正需要关注的是热点路径上大量不必要逃逸：

```go
func Values(nums []int) []any {
	values := make([]any, 0, len(nums))
	for _, n := range nums {
		values = append(values, n)
	}
	return values
}
```

这里把 `int` 装进 `any`，可能带来额外成本。但是否值得优化，要用 benchmark 判断：

```sh
go test -bench BenchmarkValues -benchmem
go build -gcflags=-m ./...
```

面试回答要避免两个极端：既不要完全无视逃逸，也不要看到 `escapes to heap` 就立刻重写代码。

## 6. 看到内存上涨时，如何区分缓存、泄漏和正常分配？

可以按三个问题排查：

1. 停止流量后，内存和 goroutine 是否回落？
2. heap profile 里哪些对象还活着？
3. 这些对象是否被预期的数据结构引用？

缓存增长示例：

```go
var cache = map[string][]byte{}

func Get(key string) []byte {
	if v, ok := cache[key]; ok {
		return v
	}
	v := load(key)
	cache[key] = v
	return v
}
```

如果没有容量上限或淘汰策略，缓存会一直增长。这不是 GC 回收不了，而是 `cache` 仍然引用着数据。

泄漏更常见的特征是：请求结束后，按理说应该释放的对象仍然被某个全局容器、后台 goroutine、channel、闭包或 timer 引用。

排查命令可以是：

```sh
go tool pprof http://localhost:6060/debug/pprof/heap
```

在 pprof 里重点看 `inuse_space` 和 `inuse_objects`，因为它们代表当前仍然存活的对象；`alloc_space` 更适合看累计分配热点。
