# 桌面动画与交互实现原理

本文档详细解析 lil agents 如何在 macOS 桌面上显示动画角色并实现交互。

## 一、窗口悬浮显示原理

应用的核心技巧是创建**无边框透明窗口**（`WalkerCharacter.swift:85-96`）：

```swift
window = NSWindow(
    contentRect: contentRect,
    styleMask: .borderless,       // 无标题栏、无边框
    backing: .buffered, defer: false
)
window.isOpaque = false           // 窗口不透明度关闭
window.backgroundColor = .clear   // 背景全透明
window.hasShadow = false          // 无系统阴影
window.level = .statusBar         // 层级在状态栏之上，始终悬浮
window.collectionBehavior = [.canJoinAllSpaces, .stationary]  // 所有桌面空间可见且固定
```

关键点：

- `styleMask: .borderless` — 没有标题栏，窗口完全由内容决定外观
- `backgroundColor = .clear` — 窗口本身透明，只有角色动画可见
- `level = .statusBar` — 窗口层级极高，显示在几乎所有窗口之上（包括 Dock）
- `canJoinAllSpaces + stationary` — 切换桌面空间时角色不会消失

## 二、角色动画：AVQueuePlayer 循环播放视频

角色不是用帧动画或 SpriteKit 绘制的，而是直接**循环播放预录制的 HEVC 透明通道视频**（`walk-bruce-01.mov`、`walk-jazz-01.mov`）：

```swift
// WalkerCharacter.swift:70-76
let asset = AVAsset(url: videoURL)
queuePlayer = AVQueuePlayer()
looper = AVPlayerLooper(player: queuePlayer, templateItem: AVPlayerItem(asset: asset))
playerLayer = AVPlayerLayer(player: queuePlayer)
playerLayer.videoGravity = .resizeAspect
playerLayer.backgroundColor = NSColor.clear.cgColor
```

- `AVPlayerLooper` 实现无缝循环播放
- `.mov` 视频带 alpha 透明通道，背景透明，只有角色可见
- 视频尺寸 1080x1920，缩放为 200pt 高度显示

## 三、行走状态机与运动曲线

每个角色有完整的**行走状态机**：暂停(pause) -> 行走(walk) -> 暂停 循环。

### 行走阶段（`movementPosition` 方法，第 684-707 行）

```
视频时间轴 (10秒一个周期):
|--静止--|--加速--|--匀速--|--减速--|--静止--|
0       3.0    3.75    8.0    8.5    10.0
```

运动曲线使用**梯形速度曲线**（加速-匀速-减速），模拟自然行走：

- **加速阶段**：二次方缓入（`v * t^2 / (2 * dIn)`）
- **匀速阶段**：线性移动
- **减速阶段**：二次方缓出（`v * (t - t^2 / (2 * dOut))`）

`positionProgress`（0.0-1.0）代表角色在 Dock 上方的水平位置比例。

### 行走启动与终止

```swift
func startWalk() {
    // 1. 决定方向：靠近边缘时朝中间走，否则随机
    // 2. 计算行走距离：200-325 像素的随机距离
    // 3. 避让其他角色：保持 12% 的最小间距
    // 4. 翻转角色朝向（CATransform3DMakeScale(-1,1,1)）
    // 5. 播放视频
}

func enterPause() {
    // 停止行走，暂停视频，回到第一帧
    // 设定随机等待时间 5-12 秒后再次行走
}
```

## 四、CVDisplayLink 驱动的帧循环

动画不用 `Timer`，而是用 **CVDisplayLink** 与显示器刷新率同步（`LilAgentsController.swift:132-147`）：

```swift
CVDisplayLinkCreateWithActiveCGDisplays(&displayLink)
CVDisplayLinkSetOutputCallback(displayLink, callback, ...)
CVDisplayLinkStart(displayLink)
```

每帧回调 `tick()` 方法执行以下操作：

1. 获取当前活跃屏幕
2. 计算该屏幕的 Dock 位置和宽度
3. 对每个角色调用 `update(dockX:dockWidth:dockTopY:)` 更新位置
4. 按位置排序设置窗口 z-order（前方角色遮挡后方）

## 五、Dock 位置计算

`getDockIconArea` 方法（第 92-128 行）精确计算 Dock 图标区域：

```
数据来源: UserDefaults(suiteName: "com.apple.dock")

读取参数:
  tilesize        → 图标大小（默认 48pt）
  persistent-apps → 固定应用数量
  persistent-others → 固定文件夹/文档数量
  recent-apps     → 最近使用应用数量
  show-recents    → 是否显示最近应用
  magnification   → 是否开启放大效果

计算公式:
  slotWidth = tileSize * 1.25          (每个图标槽宽度)
  dockWidth = slotWidth * 总图标数 + 分隔符数 * 12pt
  dockWidth *= 1.1                      (边缘 padding 修正)
  dockX = (screenWidth - dockWidth) / 2 (居中对齐)
```

角色在 `dockX` 到 `dockX + dockWidth` 范围内来回走动。

## 六、透明像素点击穿透

`CharacterContentView`（第 11-55 行）实现了 **alpha 通道点击检测** — 点击透明区域不触发响应，穿透到下层窗口：

```swift
override func hitTest(_ point: NSPoint) -> NSView? {
    // 1. 将点击坐标转换为屏幕坐标
    // 2. 用 CGWindowListCreateImage 截取该窗口在点击位置的 1x1 像素
    // 3. 读取 alpha 值
    if pixel[3] > 30 {  // alpha > 30/255 才算命中
        return self      // 有内容，接受点击
    }
    return nil           // 透明，点击穿透
}
```

关键实现细节：

- 因为 `AVPlayerLayer` 是 GPU 渲染的，`layer.render(in:)` 无法捕获视频像素
- 必须用 `CGWindowListCreateImage` 对实际屏幕内容采样
- 坐标需要从 NSScreen（左下原点）翻转到 CG（左上原点）
- 降级方案：如果截图失败，回退到中心 60% 区域的矩形命中测试

## 七、思考气泡与完成提示

当 AI 正在处理时，角色头顶显示思考气泡：

### 思考气泡

- 也是独立的无边框透明窗口，`ignoresMouseEvents = true` 不拦截鼠标
- 从 18 个短语中随机选取（"hmm...", "thinking...", "on it!" 等）
- 每 3-5 秒切换一次，带淡入淡出动画（`NSAnimationContext`）
- 气泡宽度根据文字长度动态计算

### 完成气泡

- AI 回复完成时显示（"done!", "ta-da!" 等随机短语）
- 持续 3 秒后自动消失
- 同时播放随机提示音（9 个 ping 音效中随机选一个）

## 八、技术架构总览

```
CVDisplayLink (每帧回调，与显示器刷新率同步)
    |
    v
LilAgentsController.tick()
    |
    |-- 读取 com.apple.dock UserDefaults 计算 Dock 几何位置
    |
    |-- WalkerCharacter.update() x2 (Bruce & Jazz)
    |   |-- 状态机: 暂停 <-> 行走
    |   |-- 梯形速度曲线计算位置
    |   |-- 移动 NSWindow (setFrameOrigin)
    |   |-- 更新思考/完成气泡
    |   +-- 更新 Popover 位置（如果打开）
    |
    +-- Z-order 排序（按 positionProgress 前后排列）

用户点击角色 (alpha 通道检测命中)
    |
    v
打开 Popover 聊天窗口 + 创建 AgentSession
```
