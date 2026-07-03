# 072. I/O

## 问题

`io.Reader`、`io.Writer` 和 `io.EOF` 应该怎么理解？

## 先给结论

`io.Reader` 和 `io.Writer` 是 Go 标准库最核心的小接口。Reader 的关键语义是：读取到的字节数 `n` 和错误 `err` 要一起看，`n > 0` 时必须先处理数据，再处理错误。`io.EOF` 表示流结束，通常不是异常错误。

## 1. Reader 和 Writer 的接口很小

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}
```

接口小，所以组合能力强。文件、网络连接、字符串、bytes.Buffer、压缩流都能接入同一套函数。

```go
func printAll(r io.Reader) error {
	_, err := io.Copy(os.Stdout, r)
	return err
}

printAll(strings.NewReader("hello\n"))
```

## 2. Reader 把数据读入调用方提供的 buffer

```go
r := strings.NewReader("hello")
buf := make([]byte, 2)

for {
	n, err := r.Read(buf)
	if n > 0 {
		fmt.Printf("read: %q\n", buf[:n])
	}
	if err == io.EOF {
		break
	}
	if err != nil {
		return err
	}
}
```

输出类似：

```text
read: "he"
read: "ll"
read: "o"
```

重点是 `buf[:n]`，不能直接使用整个 `buf`，因为最后一次读取可能没有填满。

## 3. `n > 0` 和 `err != nil` 可能同时出现

Reader 允许返回部分数据和错误。正确循环应该先处理 `n`，再判断 `err`。

错误写法：

```go
n, err := r.Read(buf)
if err != nil {
	return err // 可能丢掉 buf[:n] 中已经读到的数据
}
use(buf[:n])
```

正确写法：

```go
n, err := r.Read(buf)
if n > 0 {
	use(buf[:n])
}
if err != nil {
	return err
}
```

这点是 Reader 面试题里最核心的追问之一。

## 4. EOF 表示正常结束

读取完整流时，`io.EOF` 通常是结束信号，不应该当成错误日志到处上报。

```go
for {
	n, err := r.Read(buf)
	if n > 0 {
		process(buf[:n])
	}
	if err == io.EOF {
		break
	}
	if err != nil {
		return fmt.Errorf("read: %w", err)
	}
}
```

但如果协议要求必须读满固定长度，EOF 就可能是异常。

```go
buf := make([]byte, 8)
_, err := io.ReadFull(r, buf)
if err == io.ErrUnexpectedEOF {
	return fmt.Errorf("short header")
}
```

## 5. 大数据不要轻易 `io.ReadAll`

`io.ReadAll` 会把全部内容读入内存。

```go
data, err := io.ReadAll(resp.Body)
if err != nil {
	return err
}
fmt.Println(len(data))
```

小配置、小响应可以这样写；大文件、上传下载、代理转发应该流式处理。

```go
_, err := io.Copy(dst, src)
if err != nil {
	return err
}
```

流式处理的好处是内存占用与内容总大小解耦。

## 6. 复用 buffer 时不要长期保存 `buf[:n]`

下面的代码会把多个切片都指向同一个底层 buffer，后续读取会覆盖前面的内容。

```go
var chunks [][]byte
buf := make([]byte, 4)

for {
	n, err := r.Read(buf)
	if n > 0 {
		chunks = append(chunks, buf[:n]) // 错：保存了复用 buffer 的视图
	}
	if err == io.EOF {
		break
	}
}
```

如果要长期保存，必须复制。

```go
chunk := append([]byte(nil), buf[:n]...)
chunks = append(chunks, chunk)
```

## 7. 面试时怎么答

可以这样回答：

- Reader/Writer 是非常小的接口，小接口带来强组合能力。
- Reader 读入调用方提供的 buffer，返回本次读到的字节数和错误。
- `n > 0` 时先处理数据，再处理 err。
- `io.EOF` 表示流结束，通常不是异常。
- 大数据用流式 `io.Copy`，不要随便 `io.ReadAll`。
- 复用 buffer 时，如果要保存数据，需要复制 `buf[:n]`。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 Reader 要返回 `(n int, err error)` 两个值？
- 为什么读取循环要先处理 `n` 再处理 `err`？
- `io.EOF` 和 `io.ErrUnexpectedEOF` 有什么区别？
- `io.ReadAll` 什么时候不合适？
- 为什么保存 `buf[:n]` 之前经常要复制？
