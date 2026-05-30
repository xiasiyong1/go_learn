# 037. Timer 和 Ticker - 面试追问

## 追问与参考答案

### 1. `Timer.Stop` 返回值表示什么？

`Timer.Stop` 返回 true 表示定时器成功停止，返回 false 表示定时器已经触发或已经停止。返回 false 时，如果后续要复用 timer，通常需要按场景处理 channel 中可能残留的值。

### 2. 重置 Timer 前需要注意什么？

Reset 前要确认 timer 处于停止或已触发且 channel 已正确排空的状态，否则可能读到旧的触发信号。常见安全写法是 Stop 后必要时 drain，再 Reset。
