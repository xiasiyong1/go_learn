# 031. select、超时和取消

## 问题

`select` 的执行规则是什么？如何处理超时和取消？

## 先给结论

`select` 用来在多个 channel 操作之间等待。

规则要点：

- 只有 channel 发送、接收、`default` 可以出现在 case 中。
- 如果多个 case 同时就绪，会伪随机选择一个。
- 如果没有 case 就绪且没有 `default`，`select` 会阻塞。
- 如果有 `default` 且没有其他 case 就绪，会执行 `default`。
- nil channel 的 case 永远不会就绪，可用来动态禁用 case。

超时和取消通常用 `context`、`time.After`、`time.Timer` 配合 `select` 实现。

## 基本 select

```go
select {
case v := <-ch:
	fmt.Println("got", v)
case <-ctx.Done():
	return ctx.Err()
}
```

这段代码等待两件事：

- 从 `ch` 收到数据。
- context 被取消或超时。

哪个先发生，就执行哪个分支。

## 多个 case 同时 ready，不能依赖顺序

```go
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)

ch1 <- 1
ch2 <- 2

select {
case v := <-ch1:
	fmt.Println("ch1", v)
case v := <-ch2:
	fmt.Println("ch2", v)
}
```

两个 case 都 ready 时，不能依赖一定先选 `ch1`。如果业务需要优先级，就不要直接依赖 `select` 顺序，要显式写优先逻辑。

例如先尝试高优先级：

```go
select {
case v := <-high:
	handle(v)
default:
	select {
	case v := <-high:
		handle(v)
	case v := <-low:
		handle(v)
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

这类写法要谨慎，优先级逻辑越复杂越应该单独封装和测试。

## default 会让 select 非阻塞

```go
select {
case job := <-jobs:
	handle(job)
default:
	fmt.Println("no job")
}
```

没有 job 时会立即执行 `default`。

常见错误是忙轮询：

```go
for {
	select {
	case job := <-jobs:
		handle(job)
	default:
	}
}
```

这个循环在没有任务时会空转，占满 CPU。

如果要等待，就不要加 `default`：

```go
for {
	select {
	case job := <-jobs:
		handle(job)
	case <-ctx.Done():
		return
	}
}
```

如果确实要周期性检查，用 ticker：

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
	select {
	case job := <-jobs:
		handle(job)
	case <-ticker.C:
		reportIdle()
	case <-ctx.Done():
		return
	}
}
```

## 单次超时可以用 time.After

```go
select {
case v := <-result:
	return v, nil
case <-time.After(2 * time.Second):
	return "", fmt.Errorf("timeout")
}
```

这个写法适合单次等待。

如果在循环里高频使用 `time.After`，会反复创建 timer：

```go
for {
	select {
	case job := <-jobs:
		handle(job)
	case <-time.After(time.Second):
		report()
	}
}
```

这种场景更适合 `time.NewTicker` 或复用 `Timer`。

## 循环中的周期任务用 ticker

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
	select {
	case job := <-jobs:
		handle(job)
	case <-ticker.C:
		report()
	case <-ctx.Done():
		return ctx.Err()
	}
}
```

注意 `Stop()`，否则 ticker 生命周期会变得不清楚。

## nil channel 动态禁用 case

```go
func merge(a, b <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)

		for a != nil || b != nil {
			select {
			case v, ok := <-a:
				if !ok {
					a = nil
					continue
				}
				out <- v
			case v, ok := <-b:
				if !ok {
					b = nil
					continue
				}
				out <- v
			}
		}
	}()
	return out
}
```

channel 关闭后把它设为 nil，对应 case 就永远不会再被选中。这个技巧适合状态机，但要控制复杂度。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `select` 在多个 case 同时 ready 时如何选择？
- `default` 有什么作用，为什么可能导致忙轮询？
- 单次超时和循环超时分别应该怎么写？
- 为什么循环里反复 `time.After` 要谨慎？
- nil channel 在 `select` 里有什么用途？
- 如何保证取消信号不会被阻塞点漏掉？
