# 072. I/O - 面试追问

## 1. 为什么 Reader 要返回 `(n int, err error)` 两个值？

因为一次读取可能只读到部分数据，也可能在读到部分数据的同时遇到错误或 EOF。`n` 告诉你本次 buffer 中有多少字节有效，`err` 告诉你读取状态。

```go
buf := make([]byte, 4)
n, err := r.Read(buf)

if n > 0 {
	fmt.Printf("valid data: %q\n", buf[:n])
}
if err != nil {
	fmt.Println("read status:", err)
}
```

如果只返回 error，就无法表达“读到了一部分，但后面结束了”。

## 2. 为什么读取循环要先处理 `n` 再处理 `err`？

因为 `n > 0` 和 `err != nil` 可以同时成立。先 return err 可能丢数据。

错误写法：

```go
for {
	n, err := r.Read(buf)
	if err != nil {
		return err
	}
	process(buf[:n])
}
```

正确写法：

```go
for {
	n, err := r.Read(buf)
	if n > 0 {
		process(buf[:n])
	}
	if err == io.EOF {
		return nil
	}
	if err != nil {
		return err
	}
}
```

这是 Reader 语义里最容易被问深的点。

## 3. `io.EOF` 和 `io.ErrUnexpectedEOF` 有什么区别？

`io.EOF` 表示正常读到流末尾。

```go
r := strings.NewReader("abc")
data, err := io.ReadAll(r)
fmt.Println(string(data), err) // abc <nil>
```

很多高级函数会把最终 EOF 处理掉，返回 nil。

`io.ErrUnexpectedEOF` 表示还没读到预期长度，流就结束了。

```go
r := strings.NewReader("abc")
buf := make([]byte, 5)

_, err := io.ReadFull(r, buf)
fmt.Println(err == io.ErrUnexpectedEOF) // true
```

协议解析里，头部、长度字段、固定帧没读满，通常就是异常 EOF。

## 4. `io.ReadAll` 什么时候不合适？

当输入大小不可控或可能很大时不合适。

```go
data, err := io.ReadAll(req.Body) // 如果 body 很大，内存会暴涨
if err != nil {
	return err
}
```

更稳的做法是限制大小：

```go
limited := io.LimitReader(req.Body, 1<<20) // 1 MiB
data, err := io.ReadAll(limited)
```

或直接流式复制：

```go
_, err := io.Copy(dst, req.Body)
```

如果是 HTTP 服务，还应配合 `http.MaxBytesReader` 限制请求体。

## 5. 为什么保存 `buf[:n]` 之前经常要复制？

因为 `buf[:n]` 只是复用 buffer 的一个视图，下一次 Read 会覆盖同一个底层数组。

错误示例：

```go
var chunks [][]byte
buf := make([]byte, 3)

for {
	n, err := r.Read(buf)
	if n > 0 {
		chunks = append(chunks, buf[:n])
	}
	if err == io.EOF {
		break
	}
}
```

正确写法：

```go
if n > 0 {
	chunk := append([]byte(nil), buf[:n]...)
	chunks = append(chunks, chunk)
}
```

判断标准：如果只是立刻处理，可以不复制；如果要存起来、传给异步 goroutine、放进缓存，就要复制。
