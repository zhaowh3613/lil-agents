# AI CLI 进程通信机制

本文档详细解析 lil agents 如何与 Claude Code、OpenAI Codex、GitHub Copilot 三种 AI CLI 工具进行通信。

## 一、整体架构

```
+--------------------------------------------------+
|              AgentSession 协议                     |
|  start() / send(message:) / terminate()           |
|  回调: onText, onToolUse, onToolResult, ...       |
+--------+-----------------+---------------+--------+
         |                 |               |
    ClaudeSession     CodexSession    CopilotSession
    (长驻进程)         (每轮新进程)     (每轮新进程)
```

三者都用 Foundation 的 `Process` 类启动 CLI 可执行文件，通过 `Pipe`（管道）进行 stdin/stdout/stderr 通信，解析 NDJSON（每行一个 JSON）流式输出。

## 二、查找 CLI 二进制文件

通过 `ShellEnvironment` 类完成（`ShellEnvironment.swift`）：

### 第一步：获取用户 Shell 环境

```swift
// ShellEnvironment.swift:14-16
let proc = Process()
proc.executableURL = URL(fileURLWithPath: "/bin/zsh")
proc.arguments = ["-l", "-i", "-c", "echo '---ENV_START---' && env && echo '---ENV_END---'"]
```

启动 `/bin/zsh -l -i` 获取完整的登录 shell 环境变量。这是必要的，因为 macOS GUI 应用的 `ProcessInfo.processInfo.environment` **不包含**用户在 `.zshrc` / `.zprofile` 中配置的 PATH，直接搜索会找不到通过 npm/brew 安装的 CLI 工具。

### 第二步：搜索二进制文件

```swift
// ShellEnvironment.swift:46-68
static func findBinary(name: String, fallbackPaths: [String], completion: (String?) -> Void) {
    // 1. 从 Shell $PATH 逐目录搜索
    // 2. 如果 PATH 中找不到，尝试硬编码的后备路径
}
```

各 CLI 的后备搜索路径：

| CLI | 后备路径 |
|-----|----------|
| Claude | `~/.local/bin/claude`、`~/.claude/local/bin/claude`、`/usr/local/bin/claude`、`/opt/homebrew/bin/claude` |
| Codex | `~/.local/bin/codex`、`~/.npm-global/bin/codex`、`/usr/local/bin/codex`、`/opt/homebrew/bin/codex` |
| Copilot | `~/.local/bin/copilot`、`~/.npm-global/bin/copilot`、`/usr/local/bin/copilot`、`/opt/homebrew/bin/copilot` |

找到后缓存路径（`static var binaryPath`），后续调用不再搜索。

### 第三步：构建进程环境变量

```swift
// ShellEnvironment.swift:72-88
static func processEnvironment(extraPaths: [String] = []) -> [String: String] {
    var env = cachedEnvironment ?? ProcessInfo.processInfo.environment
    // 确保 PATH 包含: ~/.local/bin, /usr/local/bin, /opt/homebrew/bin 等
    env["TERM"] = "dumb"  // 禁用 CLI 的终端转义序列
    return env
}
```

## 三、进程启动模式

### Claude：长驻进程（双向流式通信）

```swift
// ClaudeSession.swift:50-58
proc.arguments = [
    "-p",                              // 非交互模式（print mode）
    "--output-format", "stream-json",  // 输出为流式 JSON
    "--input-format", "stream-json",   // 输入也是 JSON
    "--verbose",                       // 详细输出（包含工具调用信息）
    "--dangerously-skip-permissions"   // 跳过权限确认（GUI 无法交互）
]
```

通信模型：

```
App <-- stdout (NDJSON 流) ---- claude 进程 (长驻运行)
App ---- stdin (JSON) -------->
App <-- stderr -----------------
```

- 进程**启动一次，持续运行**，通过 stdin/stdout 持续双向通信
- `terminationHandler` 处理进程异常退出

### Codex：每轮新进程

```swift
// CodexSession.swift:57-64
// 第一轮:
proc.arguments = ["exec", "--json", "--full-auto", "--skip-git-repo-check", message]
// 后续轮次 (恢复上下文):
proc.arguments = ["exec", "resume", "--last", "--json", "--full-auto", "--skip-git-repo-check", message]
```

- 每次发送消息都启动一个新进程，消息作为命令行参数
- 后续轮次通过 `resume --last` 恢复对话上下文
- 进程完成任务后自动退出

### Copilot：每轮新进程

```swift
// CopilotSession.swift:62-72
var args = ["-p", message]
if !isFirstTurn { args.insert("--continue", at: 0) }  // 后续轮次恢复上下文
args.append(contentsOf: ["--output-format", "json"])
args.append("--allow-all")                             // 自动授权所有操作
```

- 与 Codex 类似，每轮新进程
- 后续轮次通过 `--continue` 恢复上下文
- 有降级逻辑：如果 JSON 解析失败，自动切换到纯文本模式

## 四、NDJSON 流式解析

三者共用相同的**行缓冲**解析模式：

```swift
// ClaudeSession.swift:136-145
private func processOutput(_ text: String) {
    lineBuffer += text                                      // 追加到缓冲区
    while let newlineRange = lineBuffer.range(of: "\n") {
        let line = String(lineBuffer[...newlineRange.lowerBound])  // 截取完整一行
        lineBuffer = String(lineBuffer[newlineRange.upperBound...]) // 剩余留缓冲
        parseLine(line)                                     // 解析单行 JSON
    }
}
```

为什么需要行缓冲？因为 stdout 数据可能在任意字节处断开（管道缓冲区刷新），一个 JSON 对象可能被拆成多次 `readabilityHandler` 回调。必须手动按 `\n` 拆行，确保每次只解析完整的 JSON 行。

### 数据接收流程

```swift
// 管道读取回调 (所有 Session 通用模式)
outPipe.fileHandleForReading.readabilityHandler = { [weak self] handle in
    let data = handle.availableData          // 读取可用数据
    guard !data.isEmpty else { return }
    if let text = String(data: data, encoding: .utf8) {
        DispatchQueue.main.async {           // 切换到主线程
            self?.processOutput(text)        // 行缓冲解析
        }
    }
}
```

## 五、事件类型解析

### Claude 事件（`ClaudeSession.swift:147-215`）

```json
// 会话初始化
{"type": "system", "subtype": "init", ...}

// AI 文本回复 + 工具调用
{"type": "assistant", "message": {"content": [
    {"type": "text", "text": "让我来看一下..."},
    {"type": "tool_use", "name": "Bash", "input": {"command": "ls -la"}}
]}}

// 工具执行结果
{"type": "user", "message": {"content": [
    {"type": "tool_result", "is_error": false, "content": "..."}
]}}

// 一轮对话结束
{"type": "result", "result": "完整回复文本"}
```

| 事件 type | 含义 | 触发回调 |
|---|---|---|
| `system` + `init` | 会话初始化完成 | `onSessionReady` |
| `assistant` + `text` | AI 文本回复 | `onText` |
| `assistant` + `tool_use` | AI 调用工具 | `onToolUse` |
| `user` + `tool_result` | 工具执行结果 | `onToolResult` |
| `result` | 一轮对话结束 | `onTurnComplete` |

### Codex 事件（`CodexSession.swift:148-214`）

| 事件 type | 含义 | 触发回调 |
|---|---|---|
| `item.started` + `command_execution` | 开始执行命令 | `onToolUse("Bash", ...)` |
| `item.completed` + `agent_message` | AI 文本回复 | `onText` |
| `item.completed` + `command_execution` | 命令执行完毕 | `onToolResult` |
| `item.completed` + `file_change` | 文件修改 | `onToolUse("FileChange", ...)` |
| `turn.completed` | 一轮结束 | `onTurnComplete` |
| `turn.failed` | 一轮失败 | `onError` + `onTurnComplete` |

### Copilot 事件（`CopilotSession.swift:171-237`）

| 事件 type | 含义 | 触发回调 |
|---|---|---|
| `assistant.message` | AI 完整回复 | 记录到 history |
| `assistant.message_delta`（ephemeral） | 流式增量文本 | `onText(delta)` |
| `assistant.tool_call` | 工具调用 | `onToolUse` |
| `assistant.tool_result` | 工具结果 | `onToolResult` |
| `assistant.turn_end` / `result` | 一轮结束 | `onTurnComplete` |

Copilot 特有的降级逻辑：如果首行 JSON 解析失败，自动切换到纯文本模式（`useJsonOutput = false`），直接将 stdout 内容作为回复文本。

## 六、消息发送方式对比

### Claude：stdin 写入 JSON

```swift
// ClaudeSession.swift:111-127
func send(message: String) {
    let payload: [String: Any] = [
        "type": "user",
        "message": ["role": "user", "content": message]
    ]
    let line = jsonStr + "\n"                         // 一行一个 JSON
    pipe.fileHandleForWriting.write(line.data(using: .utf8)!)
}
```

### Codex / Copilot：命令行参数

```swift
// 消息直接作为进程参数传入
proc.arguments = ["exec", "--json", "--full-auto", message]   // Codex
proc.arguments = ["-p", message, "--output-format", "json"]   // Copilot
```

## 七、工具调用摘要格式化

Claude Session 对常见工具做了专门的摘要提取（`ClaudeSession.swift:218-234`）：

| 工具名 | 摘要来源 |
|--------|----------|
| Bash | `input["command"]` (命令文本) |
| Read | `input["file_path"]` (文件路径) |
| Edit / Write | `input["file_path"]` (文件路径) |
| Glob | `input["pattern"]` (匹配模式) |
| Grep | `input["pattern"]` (搜索模式) |
| 其他 | `input["description"]` 或前 3 个 key |

## 八、完整数据流

```
用户输入 "帮我写个函数"
        |
        v
  +------------------+
  |  send(message:)  |
  +--------+---------+
           |
     +-----+---------------------------------------+
     | Claude: stdin 写入 JSON                       |
     | {"type":"user","message":{"role":"user",      |
     |  "content":"帮我写个函数"}}                   |
     |                                              |
     | Codex/Copilot: 启动新进程                    |
     | codex exec --json "帮我写个函数"              |
     +-----+---------------------------------------+
           |
     stdout 流式输出 (NDJSON)
           |
     +-----+---------------------------+
     | lineBuffer 行缓冲               |
     | 按 \n 拆分 -> 逐行 JSON 解析    |
     +-----+---------------------------+
           |
     +-----+------------------------------------------+
     | type="assistant" -> onText("好的,我来...")       | -> TerminalView 实时显示
     | type="assistant"/tool_use -> onToolUse           | -> 显示 "BASH ls -la"
     | type="user"/tool_result -> onToolResult          | -> 显示 "DONE file.txt"
     | type="result" -> onTurnComplete                  | -> 播放提示音 + 完成气泡
     +------------------------------------------------+
```

## 九、三种 CLI 对比总结

| 特性 | Claude | Codex | Copilot |
|------|--------|-------|---------|
| **进程模型** | 长驻进程 | 每轮新进程 | 每轮新进程 |
| **输入方式** | stdin JSON | 命令行参数 | 命令行参数 |
| **输出格式** | stream-json | json | json / 纯文本降级 |
| **上下文恢复** | 自动（长驻） | `resume --last` | `--continue` |
| **权限模式** | `--dangerously-skip-permissions` | `--full-auto` | `--allow-all` |
| **流式文本** | `assistant.content[].text` | `item.completed.text` | `assistant.message_delta` |
| **二进制路径缓存** | 有 | 有 | 有 |
| **环境变量** | Shell 环境 | Shell 环境 + npm-global | Shell 环境 + npm-global |
