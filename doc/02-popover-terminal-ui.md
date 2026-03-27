# 弹窗终端界面实现

本文档解析点击角色后打开的聊天终端窗口的实现细节。

## 一、弹窗窗口结构

点击角色后打开的是一个 420x310 的无边框主题化聊天窗口，从上到下分三层：

```
+------------------------------+
|  标题栏 (28pt)                |  <-- 显示 "claude >" 等提供者名称
+------------------------------+  <-- 1px 分隔线
|                              |
|  输出区域 (NSScrollView)      |  <-- NSTextView，只读，可选中，支持滚动
|  显示 AI 回复、工具调用、       |
|  错误信息等                    |
|                              |
+------------------------------+
|  输入框 (NSTextField, 30pt)   |  <-- 用户在这里输入消息，回车发送
+------------------------------+
```

### 窗口创建（`WalkerCharacter.swift:281-337`）

```swift
let win = KeyableWindow(
    contentRect: CGRect(x: 0, y: 0, width: 420, height: 310),
    styleMask: .borderless,
    backing: .buffered, defer: false
)
win.isOpaque = false
win.backgroundColor = .clear
win.hasShadow = true
win.level = NSWindow.Level(rawValue: NSWindow.Level.statusBar.rawValue + 10)
```

- 使用自定义 `KeyableWindow`（重写 `canBecomeKey` 为 true）以支持键盘输入
- 层级比角色窗口更高（statusBar + 10）
- 根据主题背景亮度自动设置 `darkAqua` 或 `aqua` 外观

## 二、输出区域能显示的内容类型

| 类型 | 方法 | 样式 |
|------|------|------|
| **用户输入** | `appendUser()` | `> 消息` — 粗体 + 主题强调色前缀 |
| **AI 流式回复** | `appendStreamingText()` | 支持 Markdown 渲染，实时追加 |
| **工具调用** | `appendToolUse()` | `  TOOLNAME 摘要` — 强调色标签 + 灰色摘要 |
| **工具结果** | `appendToolResult()` | `  DONE/FAIL 摘要` — 绿色/红色 |
| **错误信息** | `appendError()` | 红色文字 |

## 三、内置 Markdown 渲染器

`renderMarkdown()`（`TerminalView.swift:277-341`）手写了一个轻量 Markdown 渲染器，支持：

### 块级元素

| Markdown 语法 | 渲染效果 |
|---|---|
| `# 标题` | 字号 +2，加粗，强调色 |
| `## 标题` | 字号 +1，加粗，强调色 |
| `### 标题` | 正常字号，加粗，强调色 |
| `- 列表项` / `* 列表项` | 圆点符号 `  * ` + 内容 |
| ` ```代码块``` ` | 等宽字体，字号 -1，带背景色 |

### 行内元素（`renderInlineMarkdown`，第 343-426 行）

| Markdown 语法 | 渲染效果 |
|---|---|
| `` `code` `` | 等宽字体 + 强调色 + 背景色 |
| `**bold**` | 粗体 |
| `[text](url)` | 可点击的下划线链接 |
| `https://...` 裸 URL | 自动识别为可点击链接 |

渲染原理：逐字符扫描，遇到特殊标记（`` ` ``、`**`、`[`、`http`）时切换到对应的富文本属性，最终输出 `NSAttributedString`。

## 四、输入与消息发送

```swift
// TerminalView.swift:149-158
@objc private func inputSubmitted() {
    let text = inputField.stringValue.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !text.isEmpty else { return }
    inputField.stringValue = ""      // 清空输入框

    appendUser(text)                  // 在输出区显示用户消息
    isStreaming = true
    currentAssistantText = ""
    onSendMessage?(text)              // 回调给 WalkerCharacter -> AgentSession
}
```

- 输入框使用自定义 `PaddedTextFieldCell`，提供内边距和自定义圆角
- 按回车触发 `inputSubmitted`
- 焦点环关闭（`focusRingType: .none`），光标颜色跟随主题

## 五、弹窗打开与关闭流程

### 打开（`WalkerCharacter.openPopover()`）

```
1. 关闭其他角色的弹窗（同一时间只能有一个弹窗）
2. 角色停止行走 → 暂停视频 → 回到第一帧（站立姿势）
3. 如果没有会话，创建 AgentSession 并启动 CLI 进程
4. 创建弹窗窗口（如果不存在）
5. 恢复历史消息（replayHistory）
6. 定位弹窗到角色头顶（水平居中，屏幕边缘夹紧）
7. 设置全局事件监听器：
   - 点击弹窗和角色外部 → 关闭弹窗
   - 按 ESC 键 → 关闭弹窗
8. 聚焦输入框
```

### 关闭（`WalkerCharacter.closePopover()`）

```
1. 隐藏弹窗窗口
2. 移除事件监听器
3. 如果 AI 仍在工作 → 显示思考气泡
4. 如果 AI 刚完成 → 显示完成气泡（重置 3 秒计时）
5. 角色恢复行走（随机 2-5 秒后开始）
```

注意：关闭弹窗**不会终止 AI 会话**，会话在后台继续运行，重新打开时通过 `replayHistory` 恢复。

## 六、主题化系统

所有颜色、字体都来自 `PopoverTheme`，四套预设主题：

| 主题 | 风格 |
|------|------|
| **Peach** | 暖色调桃色 |
| **Midnight** | 深色调夜间 |
| **Cloud** | 浅色调白云 |
| **Moss** | 绿色调苔藓 |

每个主题定义以下属性：

- `popoverBg` / `popoverBorder` — 弹窗背景和边框
- `titleBarBg` / `titleText` — 标题栏样式
- `textPrimary` / `textDim` / `errorColor` / `successColor` — 文字颜色
- `accentColor` — 强调色（用于用户输入前缀、代码高亮、链接等）
- `font` / `fontBold` / `bubbleFont` — 字体
- `popoverCornerRadius` / `bubbleCornerRadius` — 圆角
- `inputBg` — 代码块背景色

主题保存在 `UserDefaults` 中，切换后立即生效。

## 七、首次引导（Onboarding）

首次启动时，Bruce 角色会显示一个特殊的欢迎弹窗：

```
hey! we're bruce and jazz -- your lil dock agents.

click either of us to open a Claude AI chat. we'll walk around
while you work and let you know when Claude's thinking.

check the menu bar icon (top right) for themes, sounds, and more options.

click anywhere outside to dismiss, then click us again to start chatting.
```

- 输入框不可编辑
- 点击外部关闭后标记 onboarding 完成（写入 UserDefaults）
- 后续点击直接打开正常聊天界面

## 八、交互流程图

```
用户点击角色
    |
    v
hitTest() alpha 检测 --> 透明区域? --> 穿透，无响应
    |
    v (命中)
handleClick()
    |
    +-- 首次引导? --> openOnboardingPopover() --> 显示欢迎信息
    |
    +-- 已打开弹窗? --> closePopover() --> 恢复行走
    |
    +-- 未打开弹窗? --> openPopover()
            |
            +-- 关闭其他弹窗
            +-- 角色停止 + 视频暂停
            +-- 创建/恢复 AgentSession
            +-- 显示弹窗 + 聚焦输入框
            |
            v
    用户输入消息并按回车
            |
            v
    inputSubmitted() --> onSendMessage --> session.send()
            |
            v
    AI 流式回复 --> appendStreamingText() (Markdown 渲染)
    工具调用   --> appendToolUse() + appendToolResult()
    完成       --> endStreaming() + 提示音 + 完成气泡
```
