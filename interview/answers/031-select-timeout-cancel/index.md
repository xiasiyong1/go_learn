# 031. select、超时和取消

## 问题

`select` 的执行规则是什么？如何处理超时和取消？

## 核心答案

`select` 会等待多个 channel 操作中可执行的一个。如果多个 case 同时可执行，会伪随机选择一个。`default` 会让 select 非阻塞。

超时通常用 `context` 或 `time.After`，长期循环中更建议复用 `Timer`，避免频繁创建临时定时器。

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

- nil channel 在 select 中有什么效果？
- `default` 使用不当会造成什么问题？
