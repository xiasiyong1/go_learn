# 039. race detector

## 问题

Go race detector 能发现什么？不能发现什么？

## 核心答案

race detector 能发现运行过程中实际发生的数据竞争，即多个 goroutine 并发访问同一变量，至少一个是写，并且没有同步。

它不能证明程序没有并发问题，只能覆盖测试或运行路径中触发到的竞争。死锁、业务竞态、原子逻辑错误也不一定能发现。

## 代码示例

```bash
go test -race ./...
go run -race main.go
```

## 面试追问

- 数据竞争和竞态条件有什么区别？
- race 检测为什么会让程序变慢？
