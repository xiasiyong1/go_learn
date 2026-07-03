# 079. 反射

## 问题

Go 反射能做什么？为什么说反射要谨慎使用？

## 先给结论

反射让程序在运行时查看类型、字段、方法和值，也可以在满足条件时修改值。它适合框架、序列化、ORM、配置绑定这类“类型在编译期不完全确定”的场景；业务核心逻辑应优先用静态类型、接口或泛型，因为反射会牺牲类型安全、可读性和性能。

## 1. Type 和 Value 分别是什么

`reflect.TypeOf` 看类型，`reflect.ValueOf` 看值。

```go
type User struct {
	Name string
	Age  int
}

u := User{Name: "alice", Age: 18}

t := reflect.TypeOf(u)
v := reflect.ValueOf(u)

fmt.Println(t.Name())  // User
fmt.Println(t.Kind())  // struct
fmt.Println(v.Kind())  // struct
fmt.Println(v.FieldByName("Name").String()) // alice
```

`Type` 适合做元信息分析，例如字段名、字段类型、tag；`Value` 适合读取或修改具体值。

## 2. 不是所有 Value 都能 Set

下面会 panic，因为 `ValueOf(u)` 拿到的是不可设置的副本。

```go
u := User{Name: "alice"}
v := reflect.ValueOf(u)

field := v.FieldByName("Name")
fmt.Println(field.CanSet()) // false

// field.SetString("bob") // panic
```

要修改原值，必须传指针，并取 `Elem()`。

```go
u := User{Name: "alice"}
v := reflect.ValueOf(&u).Elem()

field := v.FieldByName("Name")
fmt.Println(field.CanSet()) // true

field.SetString("bob")
fmt.Println(u.Name) // bob
```

面试里常追问 `CanSet`，因为它能区分“读到值”和“能改原值”。

## 3. 指针和值要分清

很多反射代码会因为没处理指针而失败。

```go
func printKind(x any) {
	v := reflect.ValueOf(x)
	fmt.Println(v.Kind())
}

printKind(User{})  // struct
printKind(&User{}) // ptr
```

如果函数既支持值也支持指针，要显式处理。

```go
func indirect(v reflect.Value) reflect.Value {
	if v.Kind() == reflect.Pointer {
		if v.IsNil() {
			return reflect.Value{}
		}
		return v.Elem()
	}
	return v
}
```

对 nil 指针调用 `Elem()` 也要小心：

```go
var u *User
v := reflect.ValueOf(u)

fmt.Println(v.IsNil()) // true
// v.Elem() 不是可用的 User 值
```

## 4. 读取 struct tag

很多库用反射读取 tag。

```go
type User struct {
	Name string `json:"name" validate:"required"`
}

t := reflect.TypeOf(User{})
field, _ := t.FieldByName("Name")

fmt.Println(field.Tag.Get("json"))     // name
fmt.Println(field.Tag.Get("validate")) // required
```

tag 只是字符串，编译器不理解业务含义。`json`、`validate`、ORM 等库自己解释这些 tag。

## 5. 反射访问未导出字段有限制

```go
type user struct {
	name string
}

u := user{name: "alice"}
v := reflect.ValueOf(u)
f := v.FieldByName("name")

fmt.Println(f.CanInterface()) // false
```

直接 `Interface()` 可能 panic。反射不是让你随意绕过封装的工具。

## 6. 反射代码要把 panic 风险挡在边界

反射里字段不存在、类型不匹配、不可设置都可能导致 panic 或零 Value。

```go
func setStringField(ptr any, name string, value string) error {
	v := reflect.ValueOf(ptr)
	if v.Kind() != reflect.Pointer || v.IsNil() {
		return fmt.Errorf("expected non-nil pointer")
	}

	elem := v.Elem()
	if elem.Kind() != reflect.Struct {
		return fmt.Errorf("expected pointer to struct")
	}

	field := elem.FieldByName(name)
	if !field.IsValid() {
		return fmt.Errorf("field not found: %s", name)
	}
	if !field.CanSet() || field.Kind() != reflect.String {
		return fmt.Errorf("field %s is not settable string", name)
	}

	field.SetString(value)
	return nil
}
```

这比直接 `FieldByName(...).SetString(...)` 更适合真实代码。

## 7. 面试时怎么答

可以这样回答：

- 反射能在运行时查看类型和值，常用于 JSON、ORM、配置、测试工具。
- `TypeOf` 看类型元信息，`ValueOf` 看具体值。
- 要修改值，必须拿到可寻址、可设置的 Value，通常传指针再 `Elem()`。
- 指针、nil、字段不存在、未导出字段、类型不匹配都要处理，否则容易 panic。
- 业务核心逻辑优先用静态类型、接口或泛型；反射放在框架边界。
- 高频路径使用反射要 benchmark 和 profile，不要凭感觉。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- `reflect.Type` 和 `reflect.Value` 分别解决什么问题？
- 为什么 `reflect.ValueOf(u).FieldByName(...).Set...` 会 panic？
- 反射如何安全处理指针和 nil？
- struct tag 是谁解释的？
- 反射和泛型分别适合什么场景？
