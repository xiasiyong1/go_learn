# 007. string、byte 和 rune - 面试追问

## 追问与参考答案

### 1. 如何正确统计字符串中的字符数量？

如果统计 Unicode 码点数量，可以用 `utf8.RuneCountInString` 或 `for range` 计数。如果要统计用户感知的字符，例如组合音标和 emoji，需要按 grapheme cluster 处理，不能只数 rune。

### 2. `[]byte(s)` 和 `string(b)` 是否会复制内存？

`[]byte(s)` 和 `string(b)` 在语义上会生成不可共享的值，通常会复制数据以保证 string 不可变。编译器在少数临时场景可能做优化，但代码不应该依赖这种优化。
