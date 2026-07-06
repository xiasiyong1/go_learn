# 039. race detector

## 问题

Go race detector 能发现什么？不能发现什么？

## 先给结论

race detector 能发现数据竞争：两个 goroutine 访问同一内存位置，至少一个是写，并且它们之间没有同步。

它不能证明程序没有并发 bug，因为：

- 只有执行到的代码路径才可能被发现。
- 它发现的是数据竞争，不是所有业务竞态。
- 开启 race 后程序会变慢、占用更多内存，不适合用它的性能结果判断真实性能。

## 一个能被发现的 race

```go
func TestRace(t *testing.T) {
	var n int
	var wg sync.WaitGroup

	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			n++
		}()
	}

	wg.Wait()
}
```

运行：

```sh
go test -race
```

`n++` 不是原子操作，两个 goroutine 并发读写同一个变量，会被报告。

修复可以用锁：

```go
var mu sync.Mutex

mu.Lock()
n++
mu.Unlock()
```

或者 atomic：

```go
var n atomic.Int64
n.Add(1)
```

## 数据竞争不等于所有竞态条件

下面没有数据竞争，但有业务竞态：

```go
func Book(seats chan struct{}) bool {
	select {
	case <-seats:
		return true
	default:
		return false
	}
}
```

如果业务要求严格公平、按请求顺序订座，channel 的并发安全不能保证公平顺序。race detector 不会告诉你“业务顺序不符合预期”。

再比如多个请求都合法，但最终业务状态不符合产品规则，这需要状态机测试、并发压力测试或更清楚的同步设计。

## 没报 race 不等于没问题

race detector 只能检查实际运行到的路径。

```sh
go test -race ./...
```

如果测试没有覆盖某个并发路径，它就不会发现那里有 race。

对于服务，可以构建 race 版本跑集成测试或压测：

```sh
go test -race ./...
go run -race ./cmd/server
```

注意 race 版本性能开销明显，不要用它的延迟和吞吐代表真实线上性能。

## 如何读 race 报告

报告通常会给出：

- 当前读或写的位置。
- 之前冲突访问的位置。
- goroutine 创建位置。

定位时不要只看第一行，要找到两个访问同一个变量的堆栈，并问：

- 这个变量应该由谁保护？
- 是否所有读写都经过同一把锁或同一个 channel？
- 是否有普通读混在 atomic 写旁边？

## 常见误用

atomic 写、普通读：

```go
var n int64

func Inc() {
	atomic.AddInt64(&n, 1)
}

func Get() int64 {
	return n // 错误
}
```

正确：

```go
func Get() int64 {
	return atomic.LoadInt64(&n)
}
```

靠 sleep 修并发：

```go
time.Sleep(time.Millisecond)
```

sleep 只是改变时序，不建立正确同步关系。要用锁、channel、WaitGroup、atomic 或 context。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- race detector 发现的“数据竞争”准确定义是什么？
- 为什么没有 race 报告不代表程序并发正确？
- 数据竞争和业务竞态有什么区别？
- race 报告应该怎么看？
- 为什么不能用 sleep 修 race？
- race detector 适合放在什么测试或 CI 环节？
