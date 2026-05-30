# 031. select、超时和取消 - 面试追问

## 追问与参考答案

### 1. nil channel 在 select 中有什么效果？

nil channel 在 select 中永远不会就绪，所以对应 case 会被禁用。这个特性常用于动态开启或关闭某个 select 分支。

### 2. `default` 使用不当会造成什么问题？

`default` 会让 select 变成非阻塞，如果放在循环里又没有休眠或其他阻塞点，可能造成 CPU 空转。它适合尝试性收发，不适合无控制地忙等。
