# 020. JSON 和结构体 tag - 面试追问

## 1. 未导出字段加 JSON tag 为什么没有效果？

`encoding/json` 只处理导出字段。tag 不会改变字段可见性。

```go
type User struct {
	ID   int64  `json:"id"`
	name string `json:"name"`
}

u := User{ID: 1, name: "Tom"}
b, _ := json.Marshal(u)

fmt.Println(string(b)) // {"id":1}
```

`name` 小写开头，包外不可见，反射也不能按普通方式设置它，所以不会参与 JSON。

正确写法：

```go
type User struct {
	ID   int64  `json:"id"`
	Name string `json:"name"`
}
```

## 2. `omitempty` 会省略哪些值？为什么 `false` 和 `0` 容易出问题？

`omitempty` 会省略空值，包括：

- `false`
- `0`
- `""`
- nil 指针、nil interface
- 长度为 0 的 slice、map、string

```go
type Config struct {
	Enabled bool `json:"enabled,omitempty"`
	Limit   int  `json:"limit,omitempty"`
}

b, _ := json.Marshal(Config{Enabled: false, Limit: 0})
fmt.Println(string(b)) // {}
```

问题是：`false` 和 `0` 可能是业务上明确传递的值。

如果需要表达“传了 false”，用指针：

```go
type Config struct {
	Enabled *bool `json:"enabled,omitempty"`
}

enabled := false
b, _ := json.Marshal(Config{Enabled: &enabled})
fmt.Println(string(b)) // {"enabled":false}
```

## 3. 解码时字段缺失、字段为 `null`、字段为零值有什么区别？

普通值字段：

```go
type User struct {
	Age int `json:"age"`
}
```

缺失字段解到新结构体是零值：

```go
var u User
_ = json.Unmarshal([]byte(`{}`), &u)
fmt.Println(u.Age) // 0
```

传零值也是 0：

```go
_ = json.Unmarshal([]byte(`{"age":0}`), &u)
fmt.Println(u.Age) // 0
```

所以普通 `int` 区分不了缺失和传 0。

指针字段可以区分缺失和具体值：

```go
type PatchUser struct {
	Age *int `json:"age"`
}

var p PatchUser
_ = json.Unmarshal([]byte(`{"age":0}`), &p)
fmt.Println(p.Age == nil) // false
```

但缺失和 `null` 都会得到 nil：

```go
_ = json.Unmarshal([]byte(`{}`), &p)
fmt.Println(p.Age == nil) // true

_ = json.Unmarshal([]byte(`{"age":null}`), &p)
fmt.Println(p.Age == nil) // true
```

要区分三态，需要自定义类型或 `json.RawMessage`。

## 4. 如何区分 PATCH 请求里的“未传”和“传了零值”？

常见做法是使用指针字段：

```go
type PatchUser struct {
	Name *string `json:"name"`
	Age  *int    `json:"age"`
}

func Apply(u *User, p PatchUser) {
	if p.Name != nil {
		u.Name = *p.Name
	}
	if p.Age != nil {
		u.Age = *p.Age
	}
}
```

客户端传：

```json
{"age":0}
```

`Age != nil`，且 `*Age == 0`。

如果业务还要区分 `null` 表示清空，可以先解码成 raw message：

```go
var raw map[string]json.RawMessage
_ = json.Unmarshal(data, &raw)

ageRaw, exists := raw["age"]
if !exists {
	// 未传
} else if string(ageRaw) == "null" {
	// 传了 null
} else {
	// 传了具体值
}
```

## 5. nil slice 和 empty slice 的 JSON 输出有什么区别？

nil slice 输出 `null`，empty slice 输出 `[]`。

```go
type Response struct {
	Items []string `json:"items"`
}

a, _ := json.Marshal(Response{Items: nil})
b, _ := json.Marshal(Response{Items: []string{}})

fmt.Println(string(a)) // {"items":null}
fmt.Println(string(b)) // {"items":[]}
```

如果字段带 `omitempty`，两者都会省略：

```go
type Response struct {
	Items []string `json:"items,omitempty"`
}

json.Marshal(Response{Items: nil})        // {}
json.Marshal(Response{Items: []string{}}) // {}
```

对外 API 如果承诺数组字段一直存在，就不要加 `omitempty`，并在输出前把 nil 归一化成 empty slice。

## 6. 自定义 `UnmarshalJSON` 为什么容易递归？

因为在 `UnmarshalJSON` 方法里对同一个类型调用 `json.Unmarshal`，会再次调用这个方法。

错误示例：

```go
func (u *User) UnmarshalJSON(data []byte) error {
	return json.Unmarshal(data, u)
}
```

正确做法是定义一个没有该方法的新类型：

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

`Alias` 是新类型，没有 `UnmarshalJSON` 方法，所以不会递归。
