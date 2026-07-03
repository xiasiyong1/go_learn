# 078. HTTP server

## 问题

Go HTTP server 中 middleware、request context 和超时应该怎么设计？

## 先给结论

HTTP server 不只是写 handler。生产代码通常需要 middleware 链处理日志、恢复、认证、超时、限流和 trace，并且每个 handler 都要尊重请求 context。

## 深入理解

### 1. 这道题真正考察什么

- 是否知道 `http.Handler` 和 `http.HandlerFunc` 的关系。
- 是否能解释请求 context 什么时候取消。
- 是否知道 middleware 是对 Handler 的包装。
- 是否能区分 server 级超时和业务 handler 超时。

### 2. 底层机制要讲清楚

- `http.Handler` 只有 `ServeHTTP(ResponseWriter, *Request)` 一个方法。
- middleware 接收一个 handler，返回一个增强后的 handler。
- 客户端断开、请求超时或服务关闭时，request context 会被取消。
- server 的 ReadTimeout、ReadHeaderTimeout、WriteTimeout、IdleTimeout 控制不同阶段。

### 3. 工程实践怎么取舍

- 在最外层 middleware 做 panic recover 和请求日志。
- 认证、限流、trace、metrics 应按清晰顺序组合。
- handler 内所有下游调用都传递 `r.Context()`。
- 为 server 配置读写超时，防止慢客户端耗尽资源。

### 4. 常见误区

- handler 启动后台 goroutine 后不监听 request context。
- middleware 写了响应后仍继续调用下游 handler。
- 未设置 ReadHeaderTimeout，暴露慢速头攻击风险。
- 在 WriteTimeout 内做长流式响应，导致连接被意外中断。

## 如何验证理解

- 用 httptest 测试 middleware 顺序、状态码和响应体。
- 模拟客户端取消，确认下游调用收到 context 取消。
- 压测慢请求和大请求，验证 server timeout 配置。

## 面试追问

追问参考答案：[follow-ups.md](follow-ups.md)

- 如果继续追问“HTTP server”的底层机制，应该讲到哪些层次？
- 这个知识点在真实项目里应该如何取舍？
- 最容易踩的坑是什么？为什么这些坑不是背结论就能避免的？
- 如何用测试、工具或 profiling 验证自己的判断？
- 当数据量、并发量或团队规模变大后，这个问题会怎样升级？
