# Go 面试题索引

这个仓库用于整理 Go 面试题。`README.md` 只做索引，每道题一个目录：`index.md` 放主答案和追问题目，`follow-ups.md` 放追问参考答案。

每道题的答案按“先给结论 -> 深入理解 -> 工程取舍 -> 常见误区 -> 验证方式 -> 面试追问”组织。目标不是让初学者背答案，而是帮助读者沿着语言语义、底层机制和真实工程场景逐层理解，回答时能讲清楚“是什么、为什么、怎么验证、规模变大后会怎样”。

难度排序按学习和面试深度从简单到困难排列。难度不是绝对标准，同一个问题在不同公司可能会继续追问到底层实现。

## 简单

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 001 | 数组 / 切片 | Go 中数组和切片有什么区别？切片扩容时需要注意什么？ | [答案](interview/answers/001-array-and-slice/index.md) |
| 007 | 字符串 | `string`、`byte` 和 `rune` 有什么区别？ | [答案](interview/answers/007-string-byte-rune/index.md) |
| 008 | 零值 | Go 常见类型的零值是什么？零值可用有什么意义？ | [答案](interview/answers/008-zero-value/index.md) |
| 009 | nil | nil 切片和空切片有什么区别？ | [答案](interview/answers/009-nil-slice-empty-slice/index.md) |
| 010 | make / new | `make` 和 `new` 有什么区别？ | [答案](interview/answers/010-make-and-new/index.md) |
| 011 | 常量 | `const` 和 `iota` 怎么用？有哪些常见坑？ | [答案](interview/answers/011-const-iota/index.md) |
| 012 | 包 | Go 的包导入、可见性和 init 执行顺序是怎样的？ | [答案](interview/answers/012-package-init-visibility/index.md) |
| 013 | 函数 | Go 中函数、方法、闭包和可变参数分别有什么特点？ | [答案](interview/answers/013-function-method-closure/index.md) |
| 014 | 指针 | Go 指针能做什么？为什么不能做指针运算？ | [答案](interview/answers/014-pointer-basics/index.md) |
| 015 | 结构体 | 结构体值拷贝、匿名字段和 tag 分别怎么理解？ | [答案](interview/answers/015-struct-basics/index.md) |
| 016 | 方法接收者 | 值接收者和指针接收者应该怎么选？ | [答案](interview/answers/016-method-receiver/index.md) |
| 017 | 类型 | 类型别名和新类型有什么区别？ | [答案](interview/answers/017-type-alias-defined-type/index.md) |
| 018 | 比较 | Go 中哪些类型可以比较？`comparable` 表示什么？ | [答案](interview/answers/018-comparable/index.md) |
| 019 | for range | `for range` 有哪些常见坑？ | [答案](interview/answers/019-for-range-pitfalls/index.md) |
| 020 | JSON | Go 结构体 JSON 序列化有哪些常见规则和坑？ | [答案](interview/answers/020-json-struct-tag/index.md) |
| 061 | 变量 | Go 中 `var`、短变量声明和变量遮蔽应该怎么理解？ | [答案](interview/answers/061-variable-declaration-shadowing/index.md) |
| 062 | 基础类型 | Go 的基础数值类型、类型转换和溢出规则有哪些需要注意？ | [答案](interview/answers/062-basic-types-conversion-overflow/index.md) |
| 063 | 控制流 | Go 的 `if`、`switch` 和条件初始化语句有哪些特点？ | [答案](interview/answers/063-if-switch-control-flow/index.md) |
| 064 | 循环 | Go 为什么只有 `for` 循环？`break`、`continue` 和标签怎么用？ | [答案](interview/answers/064-for-break-continue-label/index.md) |
| 065 | 切片操作 | Go 中切片的追加、删除、插入和过滤应该怎么写？ | [答案](interview/answers/065-slice-common-operations/index.md) |
| 066 | map 操作 | Go map 的查找、删除、零值返回和 `comma ok` 应该怎么理解？ | [答案](interview/answers/066-map-common-operations/index.md) |
| 067 | 结构体嵌入 | Go 结构体嵌入、字段提升和方法提升应该怎么理解？ | [答案](interview/answers/067-struct-embedding-promotion/index.md) |
| 068 | 接口 | Go 的接口为什么是隐式实现？接口应该由谁来定义？ | [答案](interview/answers/068-implicit-interface-satisfaction/index.md) |
| 069 | nil | Go 中哪些类型可以是 nil？不同 nil 值的行为有什么区别？ | [答案](interview/answers/069-nil-kinds/index.md) |
| 070 | 时间 | Go 中 `time.Time`、`time.Duration`、时区和单调时间应该怎么理解？ | [答案](interview/answers/070-time-duration-timezone/index.md) |
| 071 | 格式化 | `fmt` 常用格式化占位符和 `Stringer` 接口应该怎么用？ | [答案](interview/answers/071-fmt-stringer-formatting/index.md) |
| 072 | I/O | `io.Reader`、`io.Writer` 和 `io.EOF` 应该怎么理解？ | [答案](interview/answers/072-io-reader-writer-eof/index.md) |

## 中等

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 002 | map | Go 的 map 是否并发安全？如果多个 goroutine 读写 map 应该怎么处理？ | [答案](interview/answers/002-map-concurrency/index.md) |
| 003 | defer | defer 的执行顺序是什么？defer、return 和具名返回值之间是什么关系？ | [答案](interview/answers/003-defer-return/index.md) |
| 004 | interface | 为什么 interface 可能出现“看起来不是 nil，但底层值是 nil”的情况？ | [答案](interview/answers/004-interface-nil/index.md) |
| 021 | error | Go 为什么推荐显式返回 error？`errors.Is/As` 和 wrapping 怎么用？ | [答案](interview/answers/021-error-handling/index.md) |
| 022 | panic / recover | `panic` 和 `recover` 的使用边界是什么？ | [答案](interview/answers/022-panic-recover/index.md) |
| 023 | interface | 空接口、类型断言和 type switch 分别适合什么场景？ | [答案](interview/answers/023-interface-assertion-switch/index.md) |
| 024 | 泛型 | Go 泛型解决什么问题？什么时候不应该用泛型？ | [答案](interview/answers/024-generics/index.md) |
| 025 | 切片 | 大切片截取小切片为什么可能导致内存不释放？ | [答案](interview/answers/025-slice-memory-retention/index.md) |
| 026 | map | map 的 key 为什么必须可比较？遍历顺序为什么是随机的？ | [答案](interview/answers/026-map-key-iteration/index.md) |
| 027 | context | `context.Context` 应该怎么用？常见误用有哪些？ | [答案](interview/answers/027-context/index.md) |
| 028 | goroutine / channel | goroutine 和 channel 的基本通信模型是什么？ | [答案](interview/answers/028-goroutine-channel-basics/index.md) |
| 029 | channel | 无缓冲 channel 和有缓冲 channel 有什么区别？ | [答案](interview/answers/029-buffered-unbuffered-channel/index.md) |
| 030 | channel | channel 应该由谁关闭？向已关闭 channel 发送会怎样？ | [答案](interview/answers/030-channel-close/index.md) |
| 031 | select | `select` 的执行规则是什么？如何处理超时和取消？ | [答案](interview/answers/031-select-timeout-cancel/index.md) |
| 032 | sync | `Mutex` 和 `RWMutex` 有什么区别？如何避免锁误用？ | [答案](interview/answers/032-mutex-rwmutex/index.md) |
| 033 | sync | `WaitGroup` 怎么用？常见错误有哪些？ | [答案](interview/answers/033-waitgroup/index.md) |
| 034 | sync | `sync.Once` 适合解决什么问题？ | [答案](interview/answers/034-sync-once/index.md) |
| 035 | atomic | 原子操作和互斥锁如何选择？ | [答案](interview/answers/035-atomic-vs-mutex/index.md) |
| 036 | 并发模式 | worker pool 和 pipeline 怎么设计？ | [答案](interview/answers/036-worker-pool-pipeline/index.md) |
| 005 | goroutine | goroutine 泄漏通常是怎么产生的？如何排查和避免？ | [答案](interview/answers/005-goroutine-leak/index.md) |
| 037 | timer | `time.Timer` 和 `time.Ticker` 有哪些资源释放问题？ | [答案](interview/answers/037-timer-ticker/index.md) |
| 038 | 测试 | Go 表格驱动测试、子测试和 mock 应该怎么写？ | [答案](interview/answers/038-testing/index.md) |
| 039 | race | Go race detector 能发现什么？不能发现什么？ | [答案](interview/answers/039-race-detector/index.md) |
| 040 | module | Go Module 的版本、replace 和 vendor 怎么理解？ | [答案](interview/answers/040-go-module/index.md) |
| 073 | HTTP client | Go HTTP client 的超时、连接复用和响应体关闭有哪些坑？ | [答案](interview/answers/073-http-client-timeout-reuse/index.md) |
| 074 | database/sql | `database/sql` 的连接池、事务和 Rows 关闭应该怎么理解？ | [答案](interview/answers/074-database-sql-pool-transaction/index.md) |
| 075 | 并发错误 | 多个 goroutine 并发执行时，错误收集和取消传播应该怎么做？ | [答案](interview/answers/075-errgroup-error-propagation/index.md) |
| 076 | 并发模式 | fan-in、fan-out 和背压在 Go 并发里应该怎么设计？ | [答案](interview/answers/076-fan-in-fan-out-backpressure/index.md) |
| 077 | 并发限制 | Go 中如何用信号量限制并发？和 worker pool 有什么区别？ | [答案](interview/answers/077-semaphore-concurrency-limit/index.md) |
| 078 | HTTP server | Go HTTP server 中 middleware、request context 和超时应该怎么设计？ | [答案](interview/answers/078-http-server-middleware-context/index.md) |
| 079 | 反射 | Go 反射能做什么？为什么说反射要谨慎使用？ | [答案](interview/answers/079-reflection-basics/index.md) |
| 080 | 构建 | Go build tags 和按平台条件编译应该怎么用？ | [答案](interview/answers/080-build-tags-conditional-compilation/index.md) |
| 081 | 排序 | Go 中 slice 排序、map 稳定输出和自定义比较应该怎么做？ | [答案](interview/answers/081-sorting-slices-maps/index.md) |
| 082 | 配置 | Go 服务中的配置、环境变量和命令行参数应该怎么管理？ | [答案](interview/answers/082-configuration-env-flags/index.md) |
| 083 | 日志 | Go 服务日志应该怎么打？结构化日志和 context 有什么关系？ | [答案](interview/answers/083-logging-observability-basics/index.md) |
| 084 | 可测试性 | Go 中依赖注入应该怎么做？如何让代码更容易测试？ | [答案](interview/answers/084-dependency-injection-testability/index.md) |

## 困难

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 006 | GC | Go 的垃圾回收大致怎么工作？写代码时哪些行为会增加 GC 压力？ | [答案](interview/answers/006-gc-basics/index.md) |
| 041 | 逃逸分析 | 什么是逃逸分析？如何判断变量分配在栈上还是堆上？ | [答案](interview/answers/041-escape-analysis/index.md) |
| 042 | 调度器 | Go 调度器 GMP 模型是什么？ | [答案](interview/answers/042-gmp-scheduler/index.md) |
| 043 | 内存模型 | Go 内存模型中的 happens-before 怎么理解？ | [答案](interview/answers/043-memory-model/index.md) |
| 044 | map 实现 | Go map 底层大致如何实现？扩容时发生了什么？ | [答案](interview/answers/044-map-internals/index.md) |
| 045 | channel 实现 | channel 底层大致如何实现？阻塞和唤醒是怎么发生的？ | [答案](interview/answers/045-channel-internals/index.md) |
| 046 | interface 实现 | interface 底层表示是什么？空接口和非空接口有什么区别？ | [答案](interview/answers/046-interface-internals/index.md) |
| 047 | 内存分配 | Go 内存分配器大致怎么工作？小对象和大对象有什么区别？ | [答案](interview/answers/047-memory-allocator/index.md) |
| 048 | 栈 | goroutine 栈如何增长？为什么 goroutine 可以很轻量？ | [答案](interview/answers/048-goroutine-stack/index.md) |
| 049 | GC | 三色标记、写屏障和 STW 分别是什么？ | [答案](interview/answers/049-gc-tricolor-write-barrier/index.md) |
| 050 | 性能 | 如何用 benchmark、pprof 和 trace 定位性能问题？ | [答案](interview/answers/050-pprof-benchmark-trace/index.md) |
| 051 | sync.Pool | `sync.Pool` 适合什么场景？为什么不能当普通对象池？ | [答案](interview/answers/051-sync-pool/index.md) |
| 052 | 调优 | 如何减少 Go 程序的内存分配？ | [答案](interview/answers/052-reduce-allocations/index.md) |
| 053 | 网络 | Go netpoller 的作用是什么？为什么大量连接不需要大量线程？ | [答案](interview/answers/053-netpoller/index.md) |
| 054 | HTTP | 如何设计一个可优雅关闭的 Go HTTP 服务？ | [答案](interview/answers/054-http-graceful-shutdown/index.md) |
| 055 | 并发控制 | 如何设计限流、超时和熔断？ | [答案](interview/answers/055-rate-limit-timeout-circuit-breaker/index.md) |
| 056 | 锁竞争 | 如何定位和优化锁竞争？ | [答案](interview/answers/056-lock-contention/index.md) |
| 057 | 单飞 | `singleflight` 解决什么问题？适合哪些场景？ | [答案](interview/answers/057-singleflight/index.md) |
| 058 | unsafe | `unsafe` 能做什么？使用边界是什么？ | [答案](interview/answers/058-unsafe/index.md) |
| 059 | cgo | cgo 的成本和使用风险是什么？ | [答案](interview/answers/059-cgo/index.md) |
| 060 | 工程设计 | 如何设计高并发 Go 服务的整体稳定性方案？ | [答案](interview/answers/060-high-concurrency-service/index.md) |

## 新增约定

新增题目时继续按难度分组追加，答案文件保持编号前缀：

```markdown
| 085 | 主题 | 问题 | [答案](对应题目目录/index.md) |
```
