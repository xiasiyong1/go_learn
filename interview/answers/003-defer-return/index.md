# 003. defer、return 和具名返回值

## 问题

defer 的执行顺序是什么？defer、return 和具名返回值之间是什么关系？

## 先给结论

`defer` 会把函数调用延迟到当前函数返回前执行，多个 `defer` 按后进先出顺序执行。

和 `return` 放在一起时，可以按这个顺序理解：

1. 计算 `return` 后面的表达式。
2. 把结果赋给返回值。
3. 执行已经注册的 `defer`。
4. 函数真正返回给调用方。

所以具名返回值能被 `defer` 修改；匿名返回值一般不能被 `defer` 直接修改，因为它已经被复制到返回槽里了。

## defer 的执行顺序：后进先出

```go
package main

import "fmt"

func main() {
	defer fmt.Println("first")
	defer fmt.Println("second")
	defer fmt.Println("third")

	fmt.Println("body")
}
```

输出：

```text
body
third
second
first
```

可以把 `defer` 理解成当前函数维护了一组延迟调用，函数退出前按栈的顺序弹出执行。

## defer 的参数在注册时求值

这是初学者最容易错的点之一。

```go
package main

import "fmt"

func main() {
	x := 1
	defer fmt.Println("defer arg:", x)

	x = 2
	fmt.Println("body:", x)
}
```

输出：

```text
body: 2
defer arg: 1
```

原因是 `defer fmt.Println("defer arg:", x)` 注册时，`x` 作为实参已经求值为 `1`。后面把 `x` 改成 `2`，不会影响这个已保存的实参。

但闭包捕获变量时不一样：

```go
package main

import "fmt"

func main() {
	x := 1
	defer func() {
		fmt.Println("defer closure:", x)
	}()

	x = 2
}
```

输出：

```text
defer closure: 2
```

闭包里读取的是变量 `x` 本身，而不是注册 defer 那一刻的值。

如果你希望闭包拿到注册时的值，可以显式传参：

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

## 具名返回值为什么会被 defer 修改

具名返回值是函数作用域内的真实变量，`defer` 闭包可以在函数返回前修改它。

```go
package main

import "fmt"

func named() (n int) {
	defer func() {
		n++
	}()
	return 10
}

func main() {
	fmt.Println(named()) // 11
}
```

这段代码的执行过程可以拆成：

```go
func named() (n int) {
	n = 10
	func() {
		n++
	}()
	return
}
```

所以最终返回 `11`。

## 匿名返回值为什么不会被同样修改

对比下面的例子：

```go
package main

import "fmt"

func unnamed() int {
	n := 10
	defer func() {
		n++
	}()
	return n
}

func main() {
	fmt.Println(unnamed()) // 10
}
```

`return n` 会先把 `n` 的当前值复制到匿名返回值里，然后执行 `defer`。`defer` 修改的是局部变量 `n`，不是那个已经准备好的返回值。

可以粗略理解成：

```go
func unnamed() int {
	n := 10
	ret := n
	func() {
		n++
	}()
	return ret
}
```

所以最终返回 `10`。

## defer 包装错误要克制

具名返回值经常和 `defer` 一起用来包装错误：

```go
package service

import "fmt"

func LoadUser(id string) (err error) {
	defer func() {
		if err != nil {
			err = fmt.Errorf("load user %s: %w", id, err)
		}
	}()

	if id == "" {
		return fmt.Errorf("empty id")
	}
	return nil
}
```

这个写法可以，但要注意两点：

- 只在 `err != nil` 时包装，不要把成功路径改成失败。
- 不要在多个 `defer` 里反复改同一个 `err`，否则错误来源会变得难追踪。

更危险的写法是无条件覆盖错误：

```go
func Bad() (err error) {
	defer func() {
		err = nil // 错误：可能吞掉真实失败
	}()

	return fmt.Errorf("disk failed")
}
```

面试时如果能主动说出“具名返回值可以改，但不要滥用来隐藏控制流”，答案会更像真实工程经验。

## 循环里的 defer

`defer` 在当前函数返回时执行，不是在当前循环迭代结束时执行。

错误示例：

```go
func ReadAll(paths []string) error {
	for _, path := range paths {
		f, err := os.Open(path)
		if err != nil {
			return err
		}
		defer f.Close() // 文件会等到 ReadAll 返回时才关闭

		// 读取文件...
	}
	return nil
}
```

如果 `paths` 很多，文件描述符可能长时间不释放。

一种修正方式是把每次迭代拆到小函数里，让 `defer` 跟随小函数及时执行：

```go
func ReadAll(paths []string) error {
	for _, path := range paths {
		if err := readOne(path); err != nil {
			return err
		}
	}
	return nil
}

func readOne(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()

	// 读取文件...
	return nil
}
```

如果逻辑非常简单，也可以显式关闭，但要确保所有分支都能关闭资源。

## defer 和锁

锁和 `defer` 是常见组合：

```go
func (s *Store) Get(key string) (int, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	v, ok := s.m[key]
	return v, ok
}
```

它的优点是不会因为中途 `return` 忘记解锁。

但如果临界区里包含慢操作，`defer` 不能替你解决持锁时间过长的问题：

```go
func (s *Store) BadHandle(key string) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	value := s.m[key]
	return callRemoteAPI(value) // 错误：持锁期间做慢 IO
}
```

更好的做法是只在锁内拿到需要的数据：

```go
func (s *Store) Handle(key string) error {
	s.mu.Lock()
	value := s.m[key]
	s.mu.Unlock()

	return callRemoteAPI(value)
}
```

这里显式 `Unlock` 不是为了追求语法短，而是为了缩短临界区。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `return expr` 和 `defer` 的精确执行顺序是什么？
- `defer f(x)` 和 `defer func(){ f(x) }()` 为什么输出可能不同？
- 为什么具名返回值可以被 `defer` 修改，匿名返回值通常不行？
- 循环中使用 `defer` 关闭资源有什么风险？
- `defer` 解锁什么时候合适，什么时候应该显式解锁？
- `defer`、`panic`、`recover` 三者之间是什么关系？
