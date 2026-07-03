# 062. 基础类型 - 面试追问

## 1. 为什么 `int` 和 `int64` 不能直接相加？

因为它们是两个不同类型。Go 不做隐式数值转换，哪怕它们都表示整数。

```go
var a int = 1
var b int64 = 2

// _ = a + b // 编译失败
_ = int64(a) + b
```

面试官追这个问题，通常不是想听“语法规定”，而是看你能不能解释设计取舍：隐式转换会隐藏截断、符号变化和精度损失。Go 选择让这些转换显式出现在代码里。

## 2. 常量越界和变量转换越界有什么区别？

常量赋值给具体类型时，如果编译器能判断越界，会直接编译失败。

```go
const n = 256

// var x uint8 = n // 编译失败：256 overflows uint8
```

变量转换发生在运行时语义里，Go 会执行转换并得到目标类型范围内的结果。

```go
func main() {
	var n int = 256
	x := uint8(n)
	fmt.Println(x) // 0
}
```

所以不能把“显式转换能编译”理解成“业务上安全”。如果数据来自用户、数据库或网络，转换前应该做范围检查。

```go
func mustUint8(n int) (uint8, error) {
	if n < 0 || n > 255 {
		return 0, fmt.Errorf("invalid uint8: %d", n)
	}
	return uint8(n), nil
}
```

## 3. `uint(-1)` 为什么容易出问题？

严格说，`uint(-1)` 这种常量转换本身会因为负常量不能表示为 uint 而编译失败。

```go
// _ = uint(-1) // constant -1 overflows uint
```

真正常见的问题是负数变量转无符号整数。

```go
func main() {
	x := -1
	y := uint(x)

	fmt.Println(y) // 64 位平台上通常是最大 uint 值
}
```

这在分页、长度、容量、ID 转换里很危险。

```go
func makeBuffer(n int) []byte {
	if n < 0 {
		return nil
	}
	return make([]byte, n)
}
```

先判断业务范围，再转换类型。

## 4. 为什么金额不能用 `float64`？

浮点数用二进制近似表示小数，很多十进制小数无法精确表达。

```go
func main() {
	price := 0.1 + 0.2
	fmt.Println(price)        // 0.30000000000000004
	fmt.Println(price == 0.3) // false
}
```

金额常见做法是用整数保存最小单位。

```go
type Cent int64

func main() {
	var a Cent = 10 // 0.10 元
	var b Cent = 20 // 0.20 元

	fmt.Println(a + b) // 30
}
```

如果业务必须处理小数位、舍入规则、汇率等复杂场景，应使用 decimal 类型，并把舍入规则写进测试。

## 5. `byte`、`rune`、`string` 混在一起时怎么判断用哪个？

看你处理的是哪一层语义。

处理协议、文件、网络包、哈希时，用 byte：

```go
data := []byte{0x48, 0x69}
fmt.Println(string(data)) // Hi
```

处理 Unicode 码点时，用 rune：

```go
s := "你好"
for i, r := range s {
	fmt.Printf("byte index=%d rune=%U\n", i, r)
}
```

处理不可变文本整体时，用 string：

```go
func hasPrefixName(name string) bool {
	return strings.HasPrefix(name, "go")
}
```

不要用 `len(s)` 统计用户看到的字符数：

```go
s := "你好"
fmt.Println(len(s))         // 6，字节数
fmt.Println(len([]rune(s))) // 2，码点数
```

如果涉及 emoji、组合字符、显示宽度，`rune` 数也不一定等于用户感知字符数，需要专门库处理。
