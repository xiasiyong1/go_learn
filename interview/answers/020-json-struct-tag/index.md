# 020. JSON 和结构体 tag

## 问题

Go 结构体 JSON 序列化有哪些常见规则和坑？

## 先给结论

`encoding/json` 主要通过反射处理结构体的导出字段和 tag。

常见规则：

- 未导出字段不会参与 JSON 编解码，即使写了 tag 也不行。
- `json:"name"` 指定字段名。
- `json:"-"` 忽略字段。
- `omitempty` 会在字段为空值时省略字段。
- nil slice 编码为 `null`，empty slice 编码为 `[]`。
- 解码时缺失字段会保留目标字段原值；解到新结构体时就是零值。
- 自定义 `MarshalJSON` / `UnmarshalJSON` 可以改变行为，但要避免递归调用自己。

## 只有导出字段会被处理

```go
type User struct {
	ID   int64  `json:"id"`
	name string `json:"name"`
}

u := User{ID: 1, name: "Tom"}
b, _ := json.Marshal(u)

fmt.Println(string(b)) // {"id":1}
```

`name` 是未导出字段，小写开头，即使写了 `json:"name"` 也不会被 `encoding/json` 处理。

如果需要对外输出，字段必须导出：

```go
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
}
```

## tag 控制字段名和忽略

```go
type User struct {
	ID       int64  `json:"id"`
	Password string `json:"-"`
}
```

`Password` 不会输出：

```go
u := User{ID: 1, Password: "secret"}
b, _ := json.Marshal(u)

fmt.Println(string(b)) // {"id":1}
```

对外 API 建议显式写 tag，避免 Go 字段名变化影响 JSON 协议。

## omitempty 会省略空值

```go
type Response struct {
	Count int    `json:"count,omitempty"`
	OK    bool   `json:"ok,omitempty"`
	Name  string `json:"name,omitempty"`
}

b, _ := json.Marshal(Response{})
fmt.Println(string(b)) // {}
```

`0`、`false`、`""` 都会被省略。

这经常是坑：

```go
type PatchUser struct {
	Age int `json:"age,omitempty"`
}
```

如果客户端明确传 `{"age":0}`，你输出时也可能省略 `age`。如果业务需要区分“未传”和“传了 0”，不要用普通 `int` 加 `omitempty`。

## 缺失字段、null、零值

解码到新结构体时，缺失字段是零值：

```go
type User struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

var u User
_ = json.Unmarshal([]byte(`{"name":"Tom"}`), &u)

fmt.Println(u.Name) // Tom
fmt.Println(u.Age)  // 0
```

解码到已有结构体时，缺失字段保留原值：

```go
u := User{Name: "old", Age: 18}
_ = json.Unmarshal([]byte(`{"name":"Tom"}`), &u)

fmt.Println(u.Name) // Tom
fmt.Println(u.Age)  // 18
```

这在 PATCH 或复用结构体变量时很重要。

## 用指针表达可选字段

指针字段可以区分“没有值”和“有零值”：

```go
type PatchUser struct {
	Age *int `json:"age"`
}

var p PatchUser
_ = json.Unmarshal([]byte(`{"age":0}`), &p)

fmt.Println(p.Age == nil) // false
fmt.Println(*p.Age)       // 0
```

缺失字段时：

```go
var p PatchUser
_ = json.Unmarshal([]byte(`{}`), &p)

fmt.Println(p.Age == nil) // true
```

但普通指针字段不能区分“缺失”和“传 null”，两者都会得到 nil：

```go
_ = json.Unmarshal([]byte(`{"age":null}`), &p)
fmt.Println(p.Age == nil) // true
```

如果必须区分三态：未传、传 null、传具体值，需要自定义类型或先解码成 `map[string]json.RawMessage`。

## nil slice 和 empty slice

```go
type Response struct {
	Items []string `json:"items"`
}

a, _ := json.Marshal(Response{Items: nil})
b, _ := json.Marshal(Response{Items: []string{}})

fmt.Println(string(a)) // {"items":null}
fmt.Println(string(b)) // {"items":[]}
```

对外 API 通常更倾向空列表输出 `[]`，可以在边界层统一归一化：

```go
func NormalizeItems(items []string) []string {
	if items == nil {
		return []string{}
	}
	return items
}
```

如果字段加了 `omitempty`，nil slice 和 empty slice 都会被省略。

## 严格拒绝未知字段

默认情况下，未知字段会被忽略：

```go
var u User
_ = json.Unmarshal([]byte(`{"name":"Tom","unknown":1}`), &u)
```

如果是对外输入，想严格校验：

```go
dec := json.NewDecoder(r)
dec.DisallowUnknownFields()

if err := dec.Decode(&u); err != nil {
	return err
}
```

这适合管理后台、配置文件、强契约 API。开放型 API 是否拒绝未知字段，要考虑兼容性。

## 自定义 UnmarshalJSON 避免递归

错误写法：

```go
func (u *User) UnmarshalJSON(data []byte) error {
	return json.Unmarshal(data, u) // 递归调用自己
}
```

正确写法是定义别名类型打断方法集：

```go
func (u *User) UnmarshalJSON(data []byte) error {
	type Alias User
	var aux Alias

	if err := json.Unmarshal(data, &aux); err != nil {
		return err
	}
	*u = User(aux)
	return nil
}
```

如果需要额外字段：

```go
func (u *User) UnmarshalJSON(data []byte) error {
	type Alias User
	var aux struct {
		*Alias
		CreatedAt string `json:"created_at"`
	}
	aux.Alias = (*Alias)(u)

	if err := json.Unmarshal(data, &aux); err != nil {
		return err
	}
	return nil
}
```

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 未导出字段加 JSON tag 为什么没有效果？
- `omitempty` 会省略哪些值？为什么 `false` 和 `0` 容易出问题？
- 解码时字段缺失、字段为 `null`、字段为零值有什么区别？
- 如何区分 PATCH 请求里的“未传”和“传了零值”？
- nil slice 和 empty slice 的 JSON 输出有什么区别？
- 自定义 `UnmarshalJSON` 为什么容易递归？
