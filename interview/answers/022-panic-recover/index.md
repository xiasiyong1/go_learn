# 022. panic 和 recover

## 问题

`panic` 和 `recover` 的使用边界是什么？

## 核心答案

`panic` 表示程序遇到不可继续处理的异常状态，不应该替代普通 error。

`recover` 只能在 defer 函数中捕获当前 goroutine 的 panic。它常用于服务边界兜底，例如 HTTP middleware 防止单个请求导致进程退出。

## 代码示例

```go
defer func() {
	if r := recover(); r != nil {
		log.Println("panic:", r)
	}
}()
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- recover 能捕获其他 goroutine 的 panic 吗？
- panic 后 defer 是否会执行？
