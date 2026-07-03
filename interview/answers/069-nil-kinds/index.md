# 069. nil

## 问题

Go 中哪些类型可以是 nil？不同 nil 值的行为有什么区别？

## 先给结论

Go 的 nil 不是所有类型都有的零值，只有指针、slice、map、channel、func、interface 等引用类类型可以为 nil。不同 nil 值的可用边界完全不同，不能只用“是 nil”概括。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 int、string、struct 不能和 nil 比较。
- 是否能区分 nil slice、nil map、nil channel 的行为。
- 是否能解释 nil func 调用会 panic。
- 是否理解 interface 的 nil 陷阱。

### 2. 底层机制要讲清楚

- nil 表示某些类型的零引用状态。
- nil slice 可以 len、cap、range、append。
- nil map 可以读和 delete，但写入会 panic。
- nil channel 收发会永久阻塞，nil func 调用会 panic。

### 3. 工程实践怎么取舍

- 用 `len(s) == 0` 判断切片是否为空集合。
- 写 map 前确保已经 `make`。
- select 中可以利用 nil channel 动态禁用 case，但要写得非常清楚。
- 回调函数调用前判断 nil，或者提供空实现。

### 4. 常见误区

- 把所有 nil 类型当成同一种行为。
- 对 nil map 写入。
- select 中某个 channel 变 nil 后所有分支都不再就绪，造成死锁。
- 把 nil 指针装进 interface 后判断接口不为 nil。

## 如何验证理解

- 写表格测试覆盖各种 nil 类型的读、写、调用行为。
- 用 recover 测试 nil map 写和 nil func 调用的 panic。
- 对接口 nil 判断写专门测试，避免 error 返回陷阱。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“nil”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
