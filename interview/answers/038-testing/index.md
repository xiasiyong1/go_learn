# 038. 测试

## 问题

Go 表格驱动测试、子测试和 mock 应该怎么写？

## 先给结论

Go 测试优先关注可观察行为。表格驱动测试用数据表覆盖多个输入输出；子测试用 `t.Run` 给每个 case 独立名称；mock/fake 用来替换外部依赖；benchmark 用来验证性能和分配。

不要为了形式写测试。好测试应该：

- 覆盖正常路径和错误路径。
- 输入、期望、断言清楚。
- 不依赖全局状态和执行顺序。
- mock 行为不过度绑定内部实现。

## 表格驱动测试

```go
func TestAdd(t *testing.T) {
	tests := []struct {
		name string
		a    int
		b    int
		want int
	}{
		{name: "positive", a: 1, b: 2, want: 3},
		{name: "negative", a: -1, b: -2, want: -3},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			got := Add(tt.a, tt.b)
			if got != tt.want {
				t.Fatalf("got %d, want %d", got, tt.want)
			}
		})
	}
}
```

可以单独运行某个子测试：

```sh
go test -run 'TestAdd/positive'
```

## 并行子测试注意变量绑定

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

`tt := tt` 让每个子测试使用自己的 case。Go 新版本改善了常见 range 变量问题，但显式绑定仍然让代码更清楚，也照顾旧代码和不同写法。

## `t.Helper()`

```go
func assertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Fatalf("got %v, want %v", got, want)
	}
}
```

`t.Helper()` 让失败行号指向调用 helper 的测试位置，而不是 helper 内部。

## fake 比复杂 mock 更常用

定义消费方需要的小接口：

```go
type UserStore interface {
	Find(ctx context.Context, id int64) (User, error)
}

type Service struct {
	store UserStore
}
```

测试 fake：

```go
type fakeStore struct {
	user User
	err  error
}

func (f fakeStore) Find(ctx context.Context, id int64) (User, error) {
	return f.user, f.err
}
```

测试：

```go
func TestServiceGet(t *testing.T) {
	svc := Service{
		store: fakeStore{user: User{ID: 1, Name: "Tom"}},
	}

	got, err := svc.Get(context.Background(), 1)
	if err != nil {
		t.Fatal(err)
	}
	if got.Name != "Tom" {
		t.Fatalf("got %q", got.Name)
	}
}
```

mock 不要过度断言“内部调用了几次、按什么顺序调用”，除非这就是业务契约。否则重构实现时测试会无意义地碎掉。

## 外部依赖测试工具

HTTP：

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, `{"name":"Tom"}`)
}))
defer server.Close()
```

临时目录：

```go
dir := t.TempDir()
path := filepath.Join(dir, "config.yaml")
```

环境变量：

```go
t.Setenv("APP_ENV", "test")
```

这些工具能减少测试之间的全局状态污染。

## Benchmark

```go
func BenchmarkBuild(b *testing.B) {
	for i := 0; i < b.N; i++ {
		_ = Build("go", "test")
	}
}
```

运行：

```sh
go test -bench . -benchmem
```

如果结果可能被编译器优化掉，用包级变量接住：

```go
var sink string

func BenchmarkBuild(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sink = Build("go", "test")
	}
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 表格驱动测试解决什么问题？
- 子测试有什么好处？
- 并行子测试为什么要注意循环变量？
- `t.Helper()` 有什么作用？
- mock、fake、httptest 分别适合什么场景？
- benchmark 为什么要配合 `-benchmem`，如何避免结果被优化？
