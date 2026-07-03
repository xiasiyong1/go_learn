# 003. defer、return 和具名返回值 - 面试追问

## 1. `return expr` 和 `defer` 的精确执行顺序是什么？

可以记成四步：

1. 计算 `expr`。
2. 把 `expr` 的值赋给返回值。
3. 执行 `defer`。
4. 函数返回。

用代码推演最清楚：

```go
package main

import "fmt"

func f() (n int) {
	defer func() {
		fmt.Println("defer before:", n)
		n += 10
		fmt.Println("defer after:", n)
	}()

	fmt.Println("return expression")
	return 1
}

func main() {
	fmt.Println("result:", f())
}
```

输出：

```text
return expression
defer before: 1
defer after: 11
result: 11
```

`defer before` 看到的是 `1`，说明 `return 1` 已经先把 `1` 赋给了返回值；最终结果是 `11`，说明 `defer` 在真正返回前还能修改具名返回值。

## 2. `defer f(x)` 和 `defer func(){ f(x) }()` 为什么输出可能不同？

因为 `defer f(x)` 的参数在注册 `defer` 时求值；闭包里的 `x` 则在闭包执行时读取。

```go
package main

import "fmt"

func print(label string, n int) {
	fmt.Println(label, n)
}

func main() {
	x := 1

	defer print("argument:", x)
	defer func() {
		print("closure:", x)
	}()

	x = 2
}
```

输出：

```text
closure: 2
argument: 1
```

第二个 `defer` 后注册，所以先执行。它是闭包，执行时读取到 `x == 2`。第一个 `defer` 在注册时已经把参数保存成 `1`。

如果想让闭包也固定住注册时的值，把值作为参数传进去：

```go
func main() {
	x := 1
	defer func(v int) {
		fmt.Println(v)
	}(x)

	x = 2
}
```

这里输出 `1`。

## 3. 为什么具名返回值可以被 `defer` 修改，匿名返回值通常不行？

具名返回值是函数内可见的变量：

```go
func named() (n int) {
	defer func() {
		n *= 2
	}()
	return 5
}
```

它大致等价于：

```go
func named() (n int) {
	n = 5
	func() {
		n *= 2
	}()
	return
}
```

所以返回 `10`。

匿名返回值没有一个可以被 `defer` 按名字访问的返回变量。下面的 `defer` 改的是局部变量 `n`：

```go
func unnamed() int {
	n := 5
	defer func() {
		n *= 2
	}()
	return n
}
```

它大致等价于：

```go
func unnamed() int {
	n := 5
	ret := n
	func() {
		n *= 2
	}()
	return ret
}
```

所以返回 `5`。

## 4. 循环中使用 `defer` 关闭资源有什么风险？

风险是资源释放延后到整个函数结束，而不是本次循环结束。

错误示例：

```go
func CountLines(paths []string) (int, error) {
	total := 0
	for _, path := range paths {
		f, err := os.Open(path)
		if err != nil {
			return 0, err
		}
		defer f.Close()

		n, err := countLines(f)
		if err != nil {
			return 0, err
		}
		total += n
	}
	return total, nil
}
```

如果路径很多，文件会一直开着，直到 `CountLines` 返回。

推荐把单次处理拆成函数：

```go
func CountLines(paths []string) (int, error) {
	total := 0
	for _, path := range paths {
		n, err := countOne(path)
		if err != nil {
			return 0, err
		}
		total += n
	}
	return total, nil
}

func countOne(path string) (int, error) {
	f, err := os.Open(path)
	if err != nil {
		return 0, err
	}
	defer f.Close()

	return countLines(f)
}
```

这样每次 `countOne` 返回时都会关闭对应文件。

## 5. `defer` 解锁什么时候合适，什么时候应该显式解锁？

短临界区、函数逻辑简单时，`defer` 解锁非常合适：

```go
func (s *Store) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	v, ok := s.m[key]
	return v, ok
}
```

它能避免多个 `return` 分支忘记解锁。

但如果锁只需要保护一小段代码，后面还有慢操作，就应该缩短持锁时间：

```go
func (s *Store) Send(key string) error {
	s.mu.RLock()
	value, ok := s.m[key]
	s.mu.RUnlock()

	if !ok {
		return fmt.Errorf("missing key %s", key)
	}
	return sendToRemote(value)
}
```

不要写成：

```go
func (s *Store) BadSend(key string) error {
	s.mu.RLock()
	defer s.mu.RUnlock()

	value, ok := s.m[key]
	if !ok {
		return fmt.Errorf("missing key %s", key)
	}
	return sendToRemote(value) // 持锁期间做远程调用
}
```

这里的问题不是 `defer` 本身，而是临界区设计太大。

## 6. `defer`、`panic`、`recover` 三者之间是什么关系？

发生 `panic` 后，当前 goroutine 会开始退出当前函数，并执行已经注册的 `defer`。`recover` 只有在 `defer` 函数中直接调用时，才能捕获这个 panic。

```go
package main

import "fmt"

func safeRun() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()

	panic("boom")
}

func main() {
	safeRun()
	fmt.Println("continue")
}
```

输出：

```text
recovered: boom
continue
```

常见错误是把 `recover` 放在普通函数里，然后从 `defer` 调用这个普通函数的外层。下面这样不能恢复 panic：

```go
func badRecover() {
	if r := recover(); r != nil {
		fmt.Println(r)
	}
}

func run() {
	defer func() {
		badRecover() // recover 没有在 defer 函数中直接调用
	}()
	panic("boom")
}
```

实际工程里不要用 `recover` 掩盖普通错误。它更适合在 goroutine 边界、框架入口、任务调度器入口做兜底，防止单个任务把整个进程打崩。
