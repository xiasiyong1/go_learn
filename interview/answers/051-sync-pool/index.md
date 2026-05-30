# 051. sync.Pool

## 问题

`sync.Pool` 适合什么场景？为什么不能当普通对象池？

## 核心答案

`sync.Pool` 适合缓存临时对象，减少频繁分配和 GC 压力，例如临时 buffer。

Pool 中对象可能在任意 GC 后被清理，不能依赖它保存状态，也不能用它实现连接池、资源池这类需要确定生命周期的结构。

## 代码示例

```go
var bufPool = sync.Pool{
	New: func() any {
		return new(bytes.Buffer)
	},
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- Put 回 Pool 前为什么通常要 Reset？
- sync.Pool 在高并发下为什么性能较好？
