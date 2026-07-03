# 079. 反射 - 面试追问

## 1. `reflect.Type` 和 `reflect.Value` 分别解决什么问题？

`Type` 解决“这个东西是什么类型”，`Value` 解决“这个东西当前是什么值”。

```go
u := User{Name: "alice"}

t := reflect.TypeOf(u)
v := reflect.ValueOf(u)

fmt.Println(t.Name()) // User
fmt.Println(v.FieldByName("Name").String()) // alice
```

如果你要读字段 tag，用 Type：

```go
field, _ := t.FieldByName("Name")
fmt.Println(field.Tag.Get("json"))
```

如果你要读或改字段值，用 Value。

## 2. 为什么 `reflect.ValueOf(u).FieldByName(...).Set...` 会 panic？

因为 `ValueOf(u)` 得到的是传入接口里的副本，不是可设置的原始变量。

```go
u := User{Name: "alice"}
v := reflect.ValueOf(u)
f := v.FieldByName("Name")

fmt.Println(f.CanSet()) // false
// f.SetString("bob")   // panic
```

要修改原值，传指针：

```go
v := reflect.ValueOf(&u).Elem()
f := v.FieldByName("Name")

fmt.Println(f.CanSet()) // true
f.SetString("bob")
```

面试时不要只说“要传指针”，还要说明为什么：反射必须拿到可寻址、可设置的原值。

## 3. 反射如何安全处理指针和 nil？

先判断 Kind，再判断 IsNil，最后 Elem。

```go
func asStructValue(x any) (reflect.Value, error) {
	v := reflect.ValueOf(x)
	if !v.IsValid() {
		return reflect.Value{}, fmt.Errorf("nil value")
	}

	if v.Kind() == reflect.Pointer {
		if v.IsNil() {
			return reflect.Value{}, fmt.Errorf("nil pointer")
		}
		v = v.Elem()
	}

	if v.Kind() != reflect.Struct {
		return reflect.Value{}, fmt.Errorf("expected struct")
	}
	return v, nil
}
```

不要对不确定的值直接 `Elem()`：

```go
// reflect.ValueOf(123).Elem() // panic
```

## 4. struct tag 是谁解释的？

tag 是结构体字段上的字符串元数据，编译器只保留它，不理解业务语义。

```go
type User struct {
	Name string `json:"name" db:"user_name"`
}

t := reflect.TypeOf(User{})
f, _ := t.FieldByName("Name")

fmt.Println(f.Tag.Get("json")) // name
fmt.Println(f.Tag.Get("db"))   // user_name
```

`encoding/json` 会解释 `json` tag，ORM 可能解释 `db` tag，校验库可能解释 `validate` tag。tag 写错通常不会编译失败，只会运行行为不符合预期。

## 5. 反射和泛型分别适合什么场景？

泛型适合“编译期知道类型集合，只是逻辑重复”的场景。

```go
func First[T any](s []T) (T, bool) {
	if len(s) == 0 {
		var zero T
		return zero, false
	}
	return s[0], true
}
```

反射适合“运行时才知道结构”的场景，比如读取任意结构体 tag。

```go
func PrintFields(x any) {
	t := reflect.TypeOf(x)
	for i := 0; i < t.NumField(); i++ {
		fmt.Println(t.Field(i).Name)
	}
}
```

业务代码能用泛型或接口表达时，优先不用反射。反射更适合框架边界，不适合到处散落在业务逻辑里。
