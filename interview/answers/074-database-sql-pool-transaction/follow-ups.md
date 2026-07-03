# 074. database/sql - 面试追问

## 1. 为什么 `sql.DB` 不是一个连接？

`sql.DB` 是连接池管理器。它内部按需打开、复用、关闭多个底层连接。

```go
db, err := sql.Open("mysql", dsn)
if err != nil {
	return err
}

// sql.Open 通常不立即验证数据库可用
if err := db.PingContext(ctx); err != nil {
	return err
}
```

所以它应该作为长生命周期对象复用：

```go
type Repo struct {
	db *sql.DB
}

func NewRepo(db *sql.DB) *Repo {
	return &Repo{db: db}
}
```

每个请求都 `Open`/`Close` 会破坏连接复用，也可能造成连接风暴。

## 2. `Rows` 不关闭会造成什么问题？

`Rows` 持有数据库连接资源。不关闭时，连接可能迟迟不能回到连接池。

错误写法：

```go
rows, err := db.QueryContext(ctx, "select id from users")
if err != nil {
	return err
}

for rows.Next() {
	// ...
}
// 忘记 rows.Close()
```

正确写法：

```go
rows, err := db.QueryContext(ctx, "select id from users")
if err != nil {
	return err
}
defer rows.Close()

for rows.Next() {
	var id int64
	if err := rows.Scan(&id); err != nil {
		return err
	}
}

if err := rows.Err(); err != nil {
	return err
}
```

连接池耗尽时，表面现象可能是请求变慢、goroutine 堆积、`DB.Stats().WaitCount` 增长。

## 3. `QueryRow` 为什么要在 `Scan` 时处理错误？

`QueryRowContext` 返回的是 `*Row`，它没有单独的 error 返回值。查询错误、空结果、扫描错误都会在 `Scan` 暴露。

```go
var name string
err := db.QueryRowContext(ctx, "select name from users where id = ?", id).Scan(&name)
switch {
case errors.Is(err, sql.ErrNoRows):
	return "", nil
case err != nil:
	return "", err
default:
	return name, nil
}
```

如果你忘记 `Scan`，查询根本没有被完整消费，错误也不会被处理。

## 4. 事务中 `defer tx.Rollback()` 会不会影响 Commit？

常见写法是：

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback()

// 执行 SQL...

return tx.Commit()
```

如果 `Commit` 成功，defer 中的 `Rollback` 会在已提交事务上执行，通常返回 `sql.ErrTxDone`，可以忽略。它的价值是兜底失败路径。

如果你想更显式，也可以这样写：

```go
committed := false
defer func() {
	if !committed {
		_ = tx.Rollback()
	}
}()

if err := tx.Commit(); err != nil {
	return err
}
committed = true
return nil
```

多数场景第一种更简洁，团队风格统一即可。

## 5. 连接池参数应该怎么调？

先理解参数，再用指标调。

```go
db.SetMaxOpenConns(50)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

观测连接池：

```go
stats := db.Stats()
fmt.Printf("open=%d inuse=%d idle=%d wait=%d waitDur=%s\n",
	stats.OpenConnections,
	stats.InUse,
	stats.Idle,
	stats.WaitCount,
	stats.WaitDuration,
)
```

经验判断：

- `WaitCount` 和 `WaitDuration` 持续增长，说明应用侧在等连接。
- `MaxOpenConns` 太大，可能把数据库连接打满。
- `MaxIdleConns` 太小，高峰时会频繁建连。
- 调整前要看数据库 CPU、连接数、慢查询和应用延迟，不能只看 Go 侧。
