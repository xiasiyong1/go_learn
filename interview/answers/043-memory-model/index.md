# 043. Go 内存模型

## 问题

Go 内存模型中的 happens-before 怎么理解？

## 核心答案

happens-before 描述并发操作之间的可见性顺序。如果 A happens-before B，那么 B 能看到 A 在此之前完成的内存写入。

互斥锁的 Unlock happens-before 后续 Lock；channel 发送 happens-before 对应接收；close channel happens-before 接收方观察到关闭；atomic 操作也提供特定同步语义。

## 代码示例

```go
mu.Lock()
x = 1
mu.Unlock()

mu.Lock()
fmt.Println(x)
mu.Unlock()
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 没有数据竞争是否等于逻辑一定正确？
- channel 如何建立内存可见性？
