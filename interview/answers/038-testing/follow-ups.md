# 038. 测试 - 面试追问

## 1. 表格驱动测试解决什么问题？

解决重复测试逻辑和边界覆盖问题。

```go
func TestParseAge(t *testing.T) {
	tests := []struct {
		name    string
		input   string
		want    int
		wantErr bool
	}{
		{name: "valid", input: "18", want: 18},
		{name: "empty", input: "", wantErr: true},
		{name: "invalid", input: "abc", wantErr: true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got, err := ParseAge(tt.input)
			if (err != nil) != tt.wantErr {
				t.Fatalf("err=%v, wantErr=%v", err, tt.wantErr)
			}
			if got != tt.want {
				t.Fatalf("got %d, want %d", got, tt.want)
			}
		})
	}
}
```

新增 case 只需要加一行数据，不需要复制整段测试。

## 2. 子测试有什么好处？

每个 case 有独立名字，失败时更容易定位，也能单独运行。

```sh
go test -run 'TestParseAge/invalid'
```

子测试还能分组：

```go
t.Run("invalid input", func(t *testing.T) {
	// cases
})
```

复杂行为可以按场景组织，而不是一个巨大测试函数从头跑到尾。

## 3. 并行子测试为什么要注意循环变量？

并行子测试会延后执行，必须确保每个子测试拿到自己的 case。

```go
for _, tt := range tests {
	tt := tt
	t.Run(tt.name, func(t *testing.T) {
		t.Parallel()
		got := Add(tt.a, tt.b)
		if got != tt.want {
			t.Fatalf("got %d, want %d", got, tt.want)
		}
	})
}
```

显式 `tt := tt` 能让意图清楚，也避免旧代码或外部变量复用带来的问题。

## 4. `t.Helper()` 有什么作用？

它告诉测试框架：这个函数是 helper。失败时行号应该指向调用 helper 的地方。

```go
func requireNoError(t *testing.T, err error) {
	t.Helper()
	if err != nil {
		t.Fatal(err)
	}
}
```

没有 `t.Helper()` 时，失败位置可能总是 helper 内部那一行，定位不方便。

## 5. mock、fake、httptest 分别适合什么场景？

fake 适合简单替换接口：

```go
type fakeStore struct {
	user User
	err  error
}

func (f fakeStore) Find(context.Context, int64) (User, error) {
	return f.user, f.err
}
```

mock 适合需要验证交互契约，比如必须调用某个外部接口一次。但不要过度 mock 内部实现细节。

`httptest` 适合测试 HTTP 客户端或 handler：

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}))
defer srv.Close()
```

优先选择最贴近真实边界、又能保持测试稳定的方式。

## 6. benchmark 为什么要配合 `-benchmem`，如何避免结果被优化？

`-benchmem` 能看到分配次数和分配字节数：

```sh
go test -bench . -benchmem
```

避免编译器优化：

```go
var sink string

func BenchmarkBuild(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sink = Build("a", "b")
	}
}
```

如果 benchmark 涉及准备数据，循环外准备；如果每轮都要重置状态，用 `b.ResetTimer()` 排除准备成本。
