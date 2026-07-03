# 062. 基础类型

## 问题

Go 的基础数值类型、类型转换和溢出规则有哪些需要注意？

## 先给结论

Go 不会在不同数值类型之间做隐式转换。`int`、`int64`、`uint`、`float64` 是不同类型，混用时必须显式转换。显式转换不等于安全转换：整数可能截断或回绕，浮点数可能丢精度，无类型常量虽然灵活，但一旦落到具体类型也要遵守范围限制。

## 1. Go 为什么不做隐式数值转换

Go 要求你把转换写出来，避免隐藏的精度损失和符号错误。

```go
var a int = 10
var b int64 = 20

// fmt.Println(a + b) // 编译失败：mismatched types int and int64

fmt.Println(int64(a) + b) // 明确把 a 转成 int64
```

这种设计让代码更啰嗦一点，但好处是边界更清楚。尤其是协议、数据库、JSON 里的字段宽度不同，隐式转换很容易制造线上数据问题。

## 2. 无类型常量为什么能直接赋值

常量在没有指定类型前，可以保持“无类型”状态，由上下文决定最终类型。

```go
const n = 100

var a int = n
var b int64 = n
var c float64 = n

fmt.Printf("%T %T %T\n", a, b, c)
```

但常量也不能超出目标类型范围。

```go
const big = 300

var x uint8 = 255
// var y uint8 = big // 编译失败：300 overflows uint8

_ = x
```

注意：常量赋值越界会编译失败，但变量转换越界可能在运行时得到截断后的值。

## 3. 显式转换可能截断或回绕

从大整数转小整数时，Go 会按目标类型宽度保留低位。

```go
func main() {
	var x int16 = 256
	y := int8(x)

	fmt.Println(y) // 0
}
```

有符号和无符号转换也要小心。

```go
func main() {
	var x int = -1
	y := uint(x)

	fmt.Println(y) // 在 64 位平台通常是 18446744073709551615
}
```

所以“代码里写了类型转换”只表示你允许转换，不表示这个转换符合业务范围。

更安全的写法是先检查范围：

```go
func toUint8(x int) (uint8, error) {
	if x < 0 || x > 255 {
		return 0, fmt.Errorf("out of uint8 range: %d", x)
	}
	return uint8(x), nil
}
```

## 4. 浮点数不是精确小数

浮点数适合测量值和近似计算，不适合金额、库存、积分这类要求精确的数据。

```go
func main() {
	x := 0.1 + 0.2
	fmt.Println(x)          // 0.30000000000000004
	fmt.Println(x == 0.3)   // false
}
```

金额通常用整数表示最小单位：

```go
type MoneyCents int64

func add(a, b MoneyCents) MoneyCents {
	return a + b
}

func main() {
	total := add(1999, 299) // 19.99 + 2.99
	fmt.Println(total)     // 2298
}
```

## 5. `byte` 和 `rune` 只是别名，但语义不同

`byte` 是 `uint8` 的别名，通常表示原始字节。`rune` 是 `int32` 的别名，通常表示 Unicode 码点。

```go
var b byte = 'A'
var r rune = '你'

fmt.Printf("%T %v\n", b, b) // uint8 65
fmt.Printf("%T %U\n", r, r) // int32 U+4F60
```

它们底层是整数，但代码里使用 `byte` 或 `rune` 能传达意图：你处理的是字节，还是字符编码层面的码点。

## 6. 面试时怎么答

可以这样组织：

- Go 不做隐式数值转换，混用数值类型必须显式转换。
- 无类型常量在落到具体上下文前很灵活，但仍要满足目标类型范围。
- 显式整数转换可能截断或回绕，所以跨边界转换前要检查范围。
- 浮点数是近似值，不适合金额等精确小数。
- `byte` 和 `rune` 底层是整数别名，但表达的是不同语义。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 `int` 和 `int64` 不能直接相加？
- 常量越界和变量转换越界有什么区别？
- `uint(-1)` 为什么容易出问题？
- 为什么金额不能用 `float64`？
- `byte`、`rune`、`string` 混在一起时应该怎么判断用哪个？
