# 题目
有玩家在使用 Farmers Delight 与 Immortalers Delight 相关模组时，单人可以正常进入，但多人联机时客户端会断开连接，无法正常进入服务器。请解决该兼容性问题，并说明：

1. 表面报错是什么？
2. 真正原因在哪里？
3. 为什么日志里一开始看不到真正原因？
4. 你是如何把隐藏异常显现到日志中的？
5. 最终如何修复？

# 已知现象
- 服务端握手看似成功，玩家短暂进入后立刻断开。
- 客户端界面可能只显示：
    - `Internal Exception`
    - `Can't serialize unregistered packet`
    - `DecoderException`
    - `UnsupportedOperationException`
- 服务端日志可能只显示玩家加入后立刻离开，没有完整根因。

# 截图
[picture](bugpicture.png)

# 提示
该错误的第一现场不会直接出现在普通日志中。你需要想办法在网络异常捕获点或编码/解码阶段加入诊断日志，把真正的异常栈打印出来。

# 目标
- 找到导致多人断开的真正异常。
- 修复模组兼容问题。
- 不要只修复表面上的 `Can't serialize unregistered packet`。
- 说明为什么后续会出现大量看似无关的网络包解析错误。