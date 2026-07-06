# 039. race detector - 面试追问

## 1. race detector 发现的“数据竞争”准确定义是什么？

两个 goroutine 访问同一内存位置，至少一个是写，并且没有同步。

```go
var n int

go func() {
	n++
}()

go func() {
	n++
}()
```

`n++` 包含读、加、写，不是原子操作。

修复：

```go
var mu sync.Mutex

mu.Lock()
n++
mu.Unlock()
```

或者：

```go
var n atomic.Int64
n.Add(1)
```

## 2. 为什么没有 race 报告不代表程序并发正确？

因为 race detector 只能检查运行到的路径。

```sh
go test -race ./...
```

如果测试没有覆盖某段并发逻辑，就不会发现那里的 race。

另外，它发现的是数据竞争，不是全部并发设计问题。比如结果顺序、公平性、业务状态机错误，不一定有数据竞争。

## 3. 数据竞争和业务竞态有什么区别？

数据竞争是内存访问缺同步。

业务竞态是结果依赖时序，虽然内存访问可能同步了，但业务仍然不符合预期。

示例：

```go
type Store struct {
	mu sync.Mutex
	n  int
}

func (s *Store) Set(n int) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.n = n
}
```

没有数据竞争，但如果两个请求都能写 `n`，最后谁覆盖谁取决于顺序。race detector 不会判断这是否符合业务。

## 4. race 报告应该怎么看？

重点看两组堆栈：

- Read 或 Write 的当前位置。
- Previous read/write 的位置。
- goroutine 创建位置。

修复时不要只在报错行附近加锁，要找到共享变量的所有访问路径，统一同步模型。

常见问题是只修写不修读：

```go
func Set(v int) {
	mu.Lock()
	defer mu.Unlock()
	x = v
}

func Get() int {
	return x // 仍然 race
}
```

读也要加锁或 atomic。

## 5. 为什么不能用 sleep 修 race？

sleep 不建立 happens-before 关系。

```go
go func() {
	x = 1
}()

time.Sleep(time.Millisecond)
fmt.Println(x)
```

这只是碰巧等了一会儿。调度、机器性能、负载变化后仍可能出问题。

正确同步：

```go
done := make(chan struct{})
go func() {
	x = 1
	close(done)
}()

<-done
fmt.Println(x)
```

channel close 和 receive 建立同步关系。

## 6. race detector 适合放在什么测试或 CI 环节？

适合：

```sh
go test -race ./...
```

也可以对服务构建 race 版本跑集成测试或压测环境：

```sh
go run -race ./cmd/server
```

注意：

- race 版本更慢，占用更多内存。
- 不要用 race 版本的 benchmark 代表真实性能。
- 高并发关键路径应该配合压力测试，提高路径覆盖。
