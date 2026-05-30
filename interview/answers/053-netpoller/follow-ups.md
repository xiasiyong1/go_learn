# 053. netpoller - 面试追问

## 追问与参考答案

### 1. 阻塞网络 I/O 为什么不会阻塞整个线程池？

Go 的网络 fd 通常以非阻塞方式配合 netpoller 使用。goroutine 等待网络事件时会挂起，M 可以继续执行其他 goroutine，所以不会因为一个连接等待 I/O 就占住一个线程。

### 2. netpoller 和调度器如何协作？

netpoller 负责监听 fd 就绪事件，调度器负责把对应 goroutine 放回可运行队列。两者协作后，大量网络等待可以用较少线程承载。
