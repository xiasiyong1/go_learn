# 078. HTTP server - 面试追问

## 追问与参考答案

### 1. 如果继续追问底层机制，回答应该深入到什么程度？

不要停在一句结论上，要沿着“语言语义 -> 运行时或编译器机制 -> 工程影响”的顺序回答。

- `http.Handler` 只有 `ServeHTTP(ResponseWriter, *Request)` 一个方法。
- middleware 接收一个 handler，返回一个增强后的 handler。
- 客户端断开、请求超时或服务关闭时，request context 会被取消。
- server 的 ReadTimeout、ReadHeaderTimeout、WriteTimeout、IdleTimeout 控制不同阶段。

面试时可以先用一句话建立主线，再展开关键细节。这样既能让初学者听懂，也能让面试官看到你不是死记硬背。

### 2. 这个知识点在真实项目里怎么取舍？

核心不是知道某个 API 或语法能用，而是知道什么时候该用、什么时候不该用。

- 在最外层 middleware 做 panic recover 和请求日志。
- 认证、限流、trace、metrics 应按清晰顺序组合。
- handler 内所有下游调用都传递 `r.Context()`。
- 为 server 配置读写超时，防止慢客户端耗尽资源。

如果一个选择会影响可读性、性能、并发安全或 API 兼容性，要把这些代价说出来，而不是只给“用 A”或“用 B”的答案。

### 3. 这道题最容易追问哪些坑？

面试官通常会从边界条件和反例继续问，因为这些地方最能区分“会背”和“真懂”。

- handler 启动后台 goroutine 后不监听 request context。
- middleware 写了响应后仍继续调用下游 handler。
- 未设置 ReadHeaderTimeout，暴露慢速头攻击风险。
- 在 WriteTimeout 内做长流式响应，导致连接被意外中断。

回答这类追问时，最好先指出错误直觉，再解释为什么错，最后给出正确写法或规避方式。

### 4. 如何证明你的判断是对的？

Go 很适合用小实验验证语言语义，也适合用工具验证性能和并发问题。

- 用 httptest 测试 middleware 顺序、状态码和响应体。
- 模拟客户端取消，确认下游调用收到 context 取消。
- 压测慢请求和大请求，验证 server timeout 配置。

如果问题涉及并发，优先想到 `go test -race`、goroutine profile、block profile 或 trace；如果涉及性能，优先想到 benchmark、`-benchmem`、pprof 和逃逸分析。

### 5. 当规模变大后，这个问题会如何升级？

很多 Go 基础题在小程序里只是语法点，在服务端工程里会变成资源、稳定性和可维护性问题。

- 小服务可以从简单 handler 开始。
- 服务增多后，统一 middleware 能保证日志、指标、trace 和错误处理一致。
- 网关或长连接服务要单独设计超时策略，不能套用普通短请求配置。

面试中可以主动补一句规模化后的影响，这会让答案从“语言知识”升级成“工程判断”。

### 6. 初学者应该怎么把这个问题学扎实？

建议按三个层次学习：先写最小可运行例子确认语义，再读官方文档或标准库用法，最后用测试、benchmark 或 profile 观察真实行为。不要只背结论；每个结论都要能回答“为什么”和“在哪些条件下不成立”。
