# 题目
有玩家反馈安装模组后，创建或进入包含 `changede:latex_space` 维度的世界时会无限加载，或者使用命令传送到该维度后服务端线程卡死。开发环境中问题可能不明显，但正式环境中更容易复现。请解决该维度加载卡死问题，并说明：

1. 表面现象是什么？
2. 为什么开发环境和正式环境表现不一致？
3. 真正卡死的位置在哪里？
4. 为什么看起来像是传送门问题，但根因其实在维度地形替换？
5. 最终如何修复液体替换导致的区块递归加载？

# 版本
1.20.1 Forge

参考版本：

- Forge: `47.4.x`
- Minecraft: `1.20.1`
- Changed: `0.15.4`
- Changed Addon Plus: `2.8.2b`
- changedE: `1.1.0`

# 已知现象
- 正式环境中，创建新世界或进入 `changede:latex_space` 后可能长时间卡在加载界面。
- 使用命令传送到 Latex Space 时，服务端线程可能卡死：

```mcfunction
/execute in changede:latex_space run tp @s 0 0 0
```

- 客户端窗口可能仍能响应一部分操作，但集成服务端不再继续 tick。
- 强制崩溃或关闭窗口时，游戏可能无法正常退出，需要从任务管理器结束进程。
- 传送门或传送逻辑可能表现为目标点在地底、目标区块未加载、预览画面为空，但这些不一定是第一根因。
- 普通日志中可能没有明确崩溃报告，因为这是服务端线程等待区块加载时形成的卡死，不一定会主动抛出异常。

# 诊断线索
使用 Spark、Observable 或类似采样工具时，可能看到主线程长时间停在区块加载路径中：

```text
ServerPlayer.teleportTo
Entity.setPosRaw
Level.getChunk
ServerChunkCache.getChunk
BlockableEventLoop.managedBlock
ChunkMap
EventBus.post
LatexSpaceTerrainEvents.onChunkLoad
LatexSpaceTerrainEvents.replaceLatexSpaceTerrain
LevelChunk.setBlockState
LiquidBlock.onPlace
FluidInteractionRegistry.canInteract
Level.getFluidState
Level.getChunk
ServerChunkCache.getChunk
BlockableEventLoop.managedBlock
```

这个调用栈说明：玩家传送需要加载目标区块，目标区块加载时触发模组的地形替换事件，地形替换又放置了液体方块，液体方块的 `onPlace` 为了检查流体交互读取邻近流体状态，进而再次请求区块，最终形成递归加载或主线程等待。

# 推荐解题时间
4-6 小时

# 难度
5

# 难度评价
这个问题属于高难度 bug，原因是它不是普通崩溃，而是区块加载过程中的主线程卡死。日志里通常不会直接告诉你“哪一行代码错了”，需要借助采样工具从调用栈中反推阻塞链路。

难点主要有：

- 问题只在正式环境更稳定复现，开发环境可能因为加载速度、映射环境或调试状态不同而掩盖问题。
- 表面现象像是传送门、传送坐标或维度未加载，但真正原因在 `ChunkEvent.Load` 中修改液体地形。
- 卡死发生在 Minecraft 区块加载的同步等待链路中，不一定生成 crash report。
- `LevelChunk#setBlockState` 看似安全，但替换成 `LiquidBlock` 时会触发 `LiquidBlock#onPlace`，这个回调可能访问邻近区块。
- 不能简单粗暴取消所有 `LiquidBlock#onPlace`，否则会破坏正常流体放置、流动和交互，只能在 Latex Space 地形替换上下文中精准跳过。

# 提示
不要只检查传送门代码。传送门只是触发目标维度区块加载的入口，真正需要排查的是“区块加载时做了什么”。

重点检查：

- 是否在 `ChunkEvent.Load` 中大量调用 `setBlockState`。
- 是否在区块加载期间放置了水、岩浆或自定义 `LiquidBlock`。
- 自定义液体是否继承或复用了 Minecraft/Forge 的液体放置逻辑。
- `LiquidBlock#onPlace` 中是否会读取邻近位置的 `FluidState`。
- `getChunk`、`getChunkAt`、`getFluidState` 是否出现在同一条递归调用栈里。

# 目标
- 找到正式环境无限加载或卡死的真实调用栈。
- 说明为什么 `LiquidBlock#onPlace` 会导致区块递归加载。
- 修复 Latex Space 地形替换中的液体替换逻辑。
- 保留普通流体行为，不要影响玩家正常放置液体或其他维度中的液体更新。
- 说明为什么这个问题不能只通过修改传送坐标、传送门绑定或预加载目标区块解决。

# 参考修复方向
可以在地形替换液体时设置一个仅当前线程有效的标记，然后用 Mixin 在 `LiquidBlock#onPlace` 开头判断该标记：

- 如果是 Latex Space 地形替换过程中的液体放置，则取消这一次 `onPlace`。
- 如果是普通世界行为、玩家放置液体、液体自然更新，则照常执行。

这样可以避免 `LiquidBlock#onPlace -> getFluidState -> getChunk` 在区块加载期间递归请求区块，同时不会全局破坏液体机制。
