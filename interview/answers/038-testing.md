# 038. 测试

## 问题

Go 表格驱动测试、子测试和 mock 应该怎么写？

## 核心答案

表格驱动测试适合覆盖多组输入输出，减少重复测试代码。子测试用 `t.Run` 表达不同用例，便于单独运行和定位失败。

mock 应该优先围绕接口边界做，不要为了 mock 把所有东西都抽象成接口。接口最好由使用方定义。

## 代码示例

```go
for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		got := Add(tt.a, tt.b)
		if got != tt.want {
			t.Fatalf("got %d, want %d", got, tt.want)
		}
	})
}
```

## 面试追问

- `t.Helper()` 有什么作用？
- 单元测试和集成测试如何划分？
