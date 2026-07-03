# 073. HTTP client

## 问题

Go HTTP client 的超时、连接复用和响应体关闭有哪些坑？

## 先给结论

Go 的 `http.Client` 应该复用，超时要显式设置，响应体必须关闭。很多线上问题不是不会发请求，而是没有控制超时、没有读完或关闭 body、连接池配置不符合流量模型。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道默认 `http.Client` 没有整体超时。
- 是否知道 Response Body 必须关闭。
- 是否能解释连接复用依赖 Transport 和 body 处理。
- 是否能区分 client timeout、context timeout、dial timeout 和 header timeout。

### 2. 底层机制要讲清楚

- `http.Client` 内部通过 Transport 管理连接池，应该长期复用。
- 如果响应体不关闭，连接和 goroutine 可能无法及时释放。
- 要复用连接，通常需要读完 body 或让 Transport 能安全回收连接。
- 超时可以分层设置：拨号、TLS、响应头、整体请求和业务 context。

### 3. 工程实践怎么取舍

- 创建全局或注入的复用 client，不要每次请求创建新 client。
- 为请求设置 context deadline，并为 client/transport 设置合理超时。
- 始终 `defer resp.Body.Close()`，并根据需要处理读取错误。
- 高并发调用下游时配置连接池上限、空闲连接和每主机连接数。

### 4. 常见误区

- 使用默认 client 调下游，遇到慢请求无限等待。
- 忘记关闭 body，导致连接泄漏。
- 每次请求新建 client，失去连接复用并增加 fd 和握手成本。
- 无脑重试 POST 请求，导致重复写入。

## 如何验证理解

- 用 httptest 构造慢响应、慢 header 和大 body 场景。
- 压测时观察连接数、fd 数、超时率和 goroutine 数。
- 用 httptrace 或日志验证连接是否复用。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“HTTP client”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
