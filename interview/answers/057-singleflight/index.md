# 057. singleflight

## 问题

`singleflight` 解决什么问题？适合哪些场景？

## 核心答案

`singleflight` 用于合并同一个 key 的并发请求，让只有一个请求真正执行，其他请求等待并共享结果。

它适合缓存击穿、热点 key 加载、重复远程调用合并等场景。它不是缓存本身，也不负责结果持久化。

## 代码示例

```go
var g singleflight.Group
v, err, shared := g.Do(key, func() (any, error) {
	return loadFromDB(key)
})
_ = shared
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- singleflight 和互斥锁有什么区别？
- 如果执行函数很慢或失败会怎样？
