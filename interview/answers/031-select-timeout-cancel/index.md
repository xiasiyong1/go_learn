# 031. select、超时和取消

## 问题

`select` 的执行规则是什么？如何处理超时和取消？

## 先给结论

select 用于在多个 channel 操作之间等待。它常被用来组合结果、超时、取消和默认分支，关键是理解 ready case 的随机选择、nil channel 和 default 的影响。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道多个 case 同时就绪时会伪随机选择一个。
- 是否知道没有 case 就绪且无 default 时会阻塞。
- 是否知道 nil channel 的 case 永远不会就绪。
- 是否能正确组合 `time.After`、Timer 和 context 取消。

### 2. 底层机制要讲清楚

- select 只关心 channel 操作是否可以立即进行。
- default 会让 select 非阻塞，使用不当会形成忙轮询。
- nil channel 可用来动态关闭某个 case，而不必额外分支。
- `time.After` 每次调用会创建 timer，循环中使用要注意分配和资源。

### 3. 工程实践怎么取舍

- 请求级取消优先使用 context。
- 单次超时可以用 `time.After`，循环或高频路径优先复用 Timer。
- 需要退出循环时，在每个可能阻塞的 select 中监听 done 或 ctx.Done。
- default 只用于明确的非阻塞尝试，不能作为避免阻塞的万能方案。

### 4. 常见误区

- select 加 default 后疯狂空转，占满 CPU。
- 循环里反复 `time.After`，制造大量 timer 分配。
- 忽略 ctx.Done，导致上游取消后 goroutine 仍阻塞。
- 依赖多个 ready case 的选择顺序。

## 如何验证理解

- 写测试用短 timeout 验证函数是否按时返回。
- 用 benchmark 比较循环中 `time.After` 和复用 Timer 的分配。
- 用 goroutine profile 检查取消后是否还有阻塞在 select 的 goroutine。

## 代码示例

```go
select {
case v := <-ch:
	_ = v
case <-ctx.Done():
	return ctx.Err()
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“select、超时和取消”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
