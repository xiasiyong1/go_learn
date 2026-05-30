# 034. sync.Once

## 问题

`sync.Once` 适合解决什么问题？

## 核心答案

`sync.Once` 保证某段初始化逻辑在并发环境下只执行一次，常用于懒加载单例、初始化配置或建立共享资源。

如果 `Do` 中的函数 panic，Once 也会认为这次调用已经完成，后续不会再次执行。

## 代码示例

```go
var once sync.Once
var client *Client

func GetClient() *Client {
	once.Do(func() {
		client = NewClient()
	})
	return client
}
```

## 面试追问

- sync.Once 和 init 函数有什么区别？
- 初始化失败应该如何处理？
