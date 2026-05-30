# 022. panic 和 recover - 面试追问

## 追问与参考答案

### 1. recover 能捕获其他 goroutine 的 panic 吗？

不能。`recover` 只能捕获当前 goroutine 中正在展开的 panic，其他 goroutine 的 panic 必须在对应 goroutine 内部自己 defer recover。

### 2. panic 后 defer 是否会执行？

会执行。panic 发生后当前 goroutine 的调用栈开始展开，每一层已经注册的 defer 会按后进先出执行；如果没有 recover，最终程序崩溃。
