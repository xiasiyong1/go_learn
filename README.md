# Go 面试题索引

这个仓库用于整理 Go 面试题。`README.md` 只做索引，具体答案、代码示例和追问放在独立 Markdown 文件中。

难度排序按学习和面试深度从简单到困难排列。难度不是绝对标准，同一个问题在不同公司可能会继续追问到底层实现。

## 简单

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 001 | 数组 / 切片 | Go 中数组和切片有什么区别？切片扩容时需要注意什么？ | [答案](interview/answers/001-array-and-slice.md) |
| 007 | 字符串 | `string`、`byte` 和 `rune` 有什么区别？ | [答案](interview/answers/007-string-byte-rune.md) |
| 008 | 零值 | Go 常见类型的零值是什么？零值可用有什么意义？ | [答案](interview/answers/008-zero-value.md) |
| 009 | nil | nil 切片和空切片有什么区别？ | [答案](interview/answers/009-nil-slice-empty-slice.md) |
| 010 | make / new | `make` 和 `new` 有什么区别？ | [答案](interview/answers/010-make-and-new.md) |
| 011 | 常量 | `const` 和 `iota` 怎么用？有哪些常见坑？ | [答案](interview/answers/011-const-iota.md) |
| 012 | 包 | Go 的包导入、可见性和 init 执行顺序是怎样的？ | [答案](interview/answers/012-package-init-visibility.md) |
| 013 | 函数 | Go 中函数、方法、闭包和可变参数分别有什么特点？ | [答案](interview/answers/013-function-method-closure.md) |
| 014 | 指针 | Go 指针能做什么？为什么不能做指针运算？ | [答案](interview/answers/014-pointer-basics.md) |
| 015 | 结构体 | 结构体值拷贝、匿名字段和 tag 分别怎么理解？ | [答案](interview/answers/015-struct-basics.md) |
| 016 | 方法接收者 | 值接收者和指针接收者应该怎么选？ | [答案](interview/answers/016-method-receiver.md) |
| 017 | 类型 | 类型别名和新类型有什么区别？ | [答案](interview/answers/017-type-alias-defined-type.md) |
| 018 | 比较 | Go 中哪些类型可以比较？`comparable` 表示什么？ | [答案](interview/answers/018-comparable.md) |
| 019 | for range | `for range` 有哪些常见坑？ | [答案](interview/answers/019-for-range-pitfalls.md) |
| 020 | JSON | Go 结构体 JSON 序列化有哪些常见规则和坑？ | [答案](interview/answers/020-json-struct-tag.md) |

## 中等

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 002 | map | Go 的 map 是否并发安全？如果多个 goroutine 读写 map 应该怎么处理？ | [答案](interview/answers/002-map-concurrency.md) |
| 003 | defer | defer 的执行顺序是什么？defer、return 和具名返回值之间是什么关系？ | [答案](interview/answers/003-defer-return.md) |
| 004 | interface | 为什么 interface 可能出现“看起来不是 nil，但底层值是 nil”的情况？ | [答案](interview/answers/004-interface-nil.md) |
| 021 | error | Go 为什么推荐显式返回 error？`errors.Is/As` 和 wrapping 怎么用？ | [答案](interview/answers/021-error-handling.md) |
| 022 | panic / recover | `panic` 和 `recover` 的使用边界是什么？ | [答案](interview/answers/022-panic-recover.md) |
| 023 | interface | 空接口、类型断言和 type switch 分别适合什么场景？ | [答案](interview/answers/023-interface-assertion-switch.md) |
| 024 | 泛型 | Go 泛型解决什么问题？什么时候不应该用泛型？ | [答案](interview/answers/024-generics.md) |
| 025 | 切片 | 大切片截取小切片为什么可能导致内存不释放？ | [答案](interview/answers/025-slice-memory-retention.md) |
| 026 | map | map 的 key 为什么必须可比较？遍历顺序为什么是随机的？ | [答案](interview/answers/026-map-key-iteration.md) |
| 027 | context | `context.Context` 应该怎么用？常见误用有哪些？ | [答案](interview/answers/027-context.md) |
| 028 | goroutine / channel | goroutine 和 channel 的基本通信模型是什么？ | [答案](interview/answers/028-goroutine-channel-basics.md) |
| 029 | channel | 无缓冲 channel 和有缓冲 channel 有什么区别？ | [答案](interview/answers/029-buffered-unbuffered-channel.md) |
| 030 | channel | channel 应该由谁关闭？向已关闭 channel 发送会怎样？ | [答案](interview/answers/030-channel-close.md) |
| 031 | select | `select` 的执行规则是什么？如何处理超时和取消？ | [答案](interview/answers/031-select-timeout-cancel.md) |
| 032 | sync | `Mutex` 和 `RWMutex` 有什么区别？如何避免锁误用？ | [答案](interview/answers/032-mutex-rwmutex.md) |
| 033 | sync | `WaitGroup` 怎么用？常见错误有哪些？ | [答案](interview/answers/033-waitgroup.md) |
| 034 | sync | `sync.Once` 适合解决什么问题？ | [答案](interview/answers/034-sync-once.md) |
| 035 | atomic | 原子操作和互斥锁如何选择？ | [答案](interview/answers/035-atomic-vs-mutex.md) |
| 036 | 并发模式 | worker pool 和 pipeline 怎么设计？ | [答案](interview/answers/036-worker-pool-pipeline.md) |
| 005 | goroutine | goroutine 泄漏通常是怎么产生的？如何排查和避免？ | [答案](interview/answers/005-goroutine-leak.md) |
| 037 | timer | `time.Timer` 和 `time.Ticker` 有哪些资源释放问题？ | [答案](interview/answers/037-timer-ticker.md) |
| 038 | 测试 | Go 表格驱动测试、子测试和 mock 应该怎么写？ | [答案](interview/answers/038-testing.md) |
| 039 | race | Go race detector 能发现什么？不能发现什么？ | [答案](interview/answers/039-race-detector.md) |
| 040 | module | Go Module 的版本、replace 和 vendor 怎么理解？ | [答案](interview/answers/040-go-module.md) |

## 困难

| 编号 | 主题 | 面试题 | 答案 |
| --- | --- | --- | --- |
| 006 | GC | Go 的垃圾回收大致怎么工作？写代码时哪些行为会增加 GC 压力？ | [答案](interview/answers/006-gc-basics.md) |
| 041 | 逃逸分析 | 什么是逃逸分析？如何判断变量分配在栈上还是堆上？ | [答案](interview/answers/041-escape-analysis.md) |
| 042 | 调度器 | Go 调度器 GMP 模型是什么？ | [答案](interview/answers/042-gmp-scheduler.md) |
| 043 | 内存模型 | Go 内存模型中的 happens-before 怎么理解？ | [答案](interview/answers/043-memory-model.md) |
| 044 | map 实现 | Go map 底层大致如何实现？扩容时发生了什么？ | [答案](interview/answers/044-map-internals.md) |
| 045 | channel 实现 | channel 底层大致如何实现？阻塞和唤醒是怎么发生的？ | [答案](interview/answers/045-channel-internals.md) |
| 046 | interface 实现 | interface 底层表示是什么？空接口和非空接口有什么区别？ | [答案](interview/answers/046-interface-internals.md) |
| 047 | 内存分配 | Go 内存分配器大致怎么工作？小对象和大对象有什么区别？ | [答案](interview/answers/047-memory-allocator.md) |
| 048 | 栈 | goroutine 栈如何增长？为什么 goroutine 可以很轻量？ | [答案](interview/answers/048-goroutine-stack.md) |
| 049 | GC | 三色标记、写屏障和 STW 分别是什么？ | [答案](interview/answers/049-gc-tricolor-write-barrier.md) |
| 050 | 性能 | 如何用 benchmark、pprof 和 trace 定位性能问题？ | [答案](interview/answers/050-pprof-benchmark-trace.md) |
| 051 | sync.Pool | `sync.Pool` 适合什么场景？为什么不能当普通对象池？ | [答案](interview/answers/051-sync-pool.md) |
| 052 | 调优 | 如何减少 Go 程序的内存分配？ | [答案](interview/answers/052-reduce-allocations.md) |
| 053 | 网络 | Go netpoller 的作用是什么？为什么大量连接不需要大量线程？ | [答案](interview/answers/053-netpoller.md) |
| 054 | HTTP | 如何设计一个可优雅关闭的 Go HTTP 服务？ | [答案](interview/answers/054-http-graceful-shutdown.md) |
| 055 | 并发控制 | 如何设计限流、超时和熔断？ | [答案](interview/answers/055-rate-limit-timeout-circuit-breaker.md) |
| 056 | 锁竞争 | 如何定位和优化锁竞争？ | [答案](interview/answers/056-lock-contention.md) |
| 057 | 单飞 | `singleflight` 解决什么问题？适合哪些场景？ | [答案](interview/answers/057-singleflight.md) |
| 058 | unsafe | `unsafe` 能做什么？使用边界是什么？ | [答案](interview/answers/058-unsafe.md) |
| 059 | cgo | cgo 的成本和使用风险是什么？ | [答案](interview/answers/059-cgo.md) |
| 060 | 工程设计 | 如何设计高并发 Go 服务的整体稳定性方案？ | [答案](interview/answers/060-high-concurrency-service.md) |

## 新增约定

新增题目时继续按难度分组追加，答案文件保持编号前缀：

```markdown
| 061 | 主题 | 问题 | [答案](interview/answers/061-topic.md) |
```
