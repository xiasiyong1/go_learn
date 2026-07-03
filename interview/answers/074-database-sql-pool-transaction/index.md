# 074. database/sql

## 问题

`database/sql` 的连接池、事务和 Rows 关闭应该怎么理解？

## 先给结论

`*sql.DB` 不是一个数据库连接，而是一个并发安全的连接池句柄，应该长期复用。`Rows` 持有连接资源，必须关闭；事务 `*sql.Tx` 绑定单个连接，必须 `Commit` 或 `Rollback`；`QueryRow` 的错误会延迟到 `Scan` 时返回。

## 1. `sql.DB` 应该复用

错误写法：每个请求都 `sql.Open`。

```go
func handler(w http.ResponseWriter, r *http.Request) {
	db, err := sql.Open("mysql", dsn)
	if err != nil {
		http.Error(w, err.Error(), 500)
		return
	}
	defer db.Close()

	_ = db
}
```

`sql.Open` 返回的是连接池句柄，不一定立即建立连接。通常在应用启动时创建一次：

```go
db, err := sql.Open("mysql", dsn)
if err != nil {
	return err
}

ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

if err := db.PingContext(ctx); err != nil {
	return err
}
```

之后把 `db` 注入 repository 或 service，应用退出时再关闭。

## 2. 连接池参数是保护数据库的关键

```go
db.SetMaxOpenConns(50)
db.SetMaxIdleConns(10)
db.SetConnMaxLifetime(30 * time.Minute)
db.SetConnMaxIdleTime(5 * time.Minute)
```

含义：

- `MaxOpenConns`：最多同时打开多少连接。
- `MaxIdleConns`：最多保留多少空闲连接。
- `ConnMaxLifetime`：连接最长存活时间。
- `ConnMaxIdleTime`：连接最大空闲时间。

连接数不是越大越好。太大可能把数据库打满，太小会让应用侧排队。

可以观察：

```go
stats := db.Stats()
fmt.Println(stats.OpenConnections, stats.InUse, stats.Idle, stats.WaitCount)
```

## 3. `Rows` 必须关闭，并检查 `rows.Err()`

```go
rows, err := db.QueryContext(ctx, "select id, name from users")
if err != nil {
	return err
}
defer rows.Close()

for rows.Next() {
	var id int64
	var name string
	if err := rows.Scan(&id, &name); err != nil {
		return err
	}
	fmt.Println(id, name)
}

if err := rows.Err(); err != nil {
	return err
}
```

`Rows` 不关闭会占用连接，最终可能导致连接池耗尽。`rows.Err()` 用于检查迭代过程中发生的错误，不能省略。

## 4. `QueryRow` 的错误在 `Scan` 时返回

`QueryRowContext` 本身不返回 error。

```go
var name string
err := db.QueryRowContext(ctx, "select name from users where id = ?", id).Scan(&name)
if errors.Is(err, sql.ErrNoRows) {
	return "", nil
}
if err != nil {
	return "", err
}
return name, nil
```

很多初学者以为 `QueryRow` 会立刻返回查询错误，但它会把错误延迟到 `Scan`。

## 5. 事务必须结束

标准写法：

```go
tx, err := db.BeginTx(ctx, nil)
if err != nil {
	return err
}
defer tx.Rollback()

if _, err := tx.ExecContext(ctx, "update accounts set balance = balance - ? where id = ?", amount, from); err != nil {
	return err
}
if _, err := tx.ExecContext(ctx, "update accounts set balance = balance + ? where id = ?", amount, to); err != nil {
	return err
}

if err := tx.Commit(); err != nil {
	return err
}
```

`defer tx.Rollback()` 是兜底。事务成功提交后，再执行 Rollback 通常会返回错误，但可以忽略；关键是失败路径不会忘记回滚。

## 6. 事务里不要做慢操作

事务占用连接，也可能持有数据库锁。不要在事务中做 RPC、HTTP 调用或复杂计算。

不推荐：

```go
tx, _ := db.BeginTx(ctx, nil)
defer tx.Rollback()

callRemoteService() // 慢操作，事务一直开着

_, _ = tx.ExecContext(ctx, "update orders set status = ?", "paid")
_ = tx.Commit()
```

更好的做法是把事务范围缩到最小，只包住必须原子提交的 SQL。

## 7. 面试时怎么答

可以这样回答：

- `sql.DB` 是连接池句柄，应长期复用，不是每次请求创建。
- 要配置连接池上限，保护数据库和应用。
- `Rows` 必须 `Close`，遍历后要检查 `rows.Err()`。
- `QueryRow` 的错误在 `Scan` 时返回，空结果是 `sql.ErrNoRows`。
- `Tx` 绑定单个连接，必须 Commit 或 Rollback。
- 事务范围要小，不要持事务做慢 I/O。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 为什么 `sql.DB` 不是一个连接？
- `Rows` 不关闭会造成什么问题？
- `QueryRow` 为什么要在 `Scan` 时处理错误？
- 事务中 `defer tx.Rollback()` 会不会影响 Commit？
- 连接池参数应该怎么调？
