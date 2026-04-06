# 技巧十六：Hooks——外置的提示词

> 核心思想：与其在提示词里反复强调规则，不如在事件触发时精确注入——一次 token，处处生效

---

## 问题本质

提示词的 token 预算有限。"每次编辑文件后运行 linter"、"如果修改了 TypeScript 文件必须通过类型检查"——这些规则如果写在系统提示词里，意味着**每一轮对话都要为它们付费**，即使用户只是在问天气。

更糟的是，提示词规则是"对模型说话"，而模型可能会漏掉、误解或选择性忽略。你需要一种**机制化**的方式：不是告诉模型"你应该做 X"，而是"当 X 事件发生时，自动执行 Y"。

这就是 Hooks 的设计哲学——**把约束从提示词中剥离，外挂到事件系统里**。

---

## 第一层：30 个生命周期事件全景

Claude Code 定义了 30 个生命周期事件，覆盖了从会话创建到工具执行的每一个关键环节。

**源码**：event union type ~line 81750

### 事件分类表

| 分类 | 事件名 | 触发时机 |
|------|--------|---------|
| **工具生命周期** | `PreToolUse` | 工具调用前（可拦截修改输入/权限） |
| | `PostToolUse` | 工具调用成功后 |
| | `PostToolUseFailure` | 工具调用失败后 |
| **会话管理** | `SessionStart` | 会话初始化 |
| | `SessionEnd` | 会话结束 |
| **模型交互** | `UserPromptSubmit` | 用户提交 prompt 后 |
| | `Stop` | 模型停止生成 |
| | `StopFailure` | 模型停止生成失败 |
| **子 Agent** | `SubagentStart` | 子 Agent 启动 |
| | `SubagentStop` | 子 Agent 停止 |
| **上下文管理** | `PreCompact` | 上下文压缩前 |
| | `PostCompact` | 上下文压缩后 |
| **权限系统** | `PermissionRequest` | 权限请求 |
| | `PermissionDenied` | 权限被拒绝 |
| **通知与交互** | `Notification` | 系统通知 |
| | `Elicitation` | MCP 服务器请求用户输入 |
| | `ElicitationResult` | 用户输入结果 |
| **初始化** | `Setup` | 初始化设置 |
| | `ConfigChange` | 配置变更 |
| | `InstructionsLoaded` | 指令加载完成 |
| **环境** | `CwdChanged` | 工作目录变更 |
| | `FileChanged` | 文件变更 |
| **工作树** | `WorktreeCreate` | 工作树创建 |
| | `WorktreeRemove` | 工作树移除 |
| **团队协作** | `TeammateIdle` | 队友空闲 |
| **任务** | `TaskCreated` | 任务创建 |
| | `TaskCompleted` | 任务完成 |

30 个事件不是随意罗列——它们覆盖了一个 AI 编程助手从"启动"到"执行"的完整生命周期。你可以在任何一个环节插入自定义逻辑。

---

## 第二层：三种拦截器类型

每个事件可以绑定三种不同类型的处理器：

**源码**：hook schema ~line 11857

### 类型对比

| 类型 | 执行方式 | 适用场景 | 返回值能力 |
|------|---------|---------|-----------|
| `command` | 运行 shell 脚本 | 执行 lint、格式化、构建检查 | 通过 stdout JSON 返回决策 |
| `prompt` | 发送给 LLM 评估 | 智能判断、语义分析 | LLM 返回结构化决策 |
| `http` | 发送 webhook POST | 集成外部系统、通知 | HTTP 响应返回决策 |

### Command 拦截器示例

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npx eslint --fix $file",
            "async": true
          }
        ]
      }
    ]
  }
}
```

**关键特性**：
- `matcher`：只在特定工具触发时执行（如只匹配 `Edit`、`Write`）
- `async: true`：后台运行，不阻塞主流程
- exit code 2 = `asyncRewake`：异步完成时唤醒模型继续处理

### Prompt 拦截器示例

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "审查用户的请求是否包含敏感信息。如果是，返回 {\"blocked\": true, \"reason\": \"...\"}"
          }
        ]
      }
    ]
  }
}
```

适合需要**语义理解**的场景——规则太复杂无法写成 shell 脚本时，交给小模型判断。

### HTTP 拦截器示例

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://audit.example.com/log",
            "headers": { "Authorization": "Bearer xxx" }
          }
        ]
      }
    ]
  }
}
```

适合**外部系统集成**——每次工具执行后通知审计系统、更新 dashboard。

---

## 第三层：Hooks 能做什么

Hooks 不只是"通知"，它们有真正的**干预能力**。

### 覆盖权限决策

这是最强的能力。`PreToolUse` 钩子可以通过返回值直接决定工具调用的命运：

```json
{
  "permissionDecision": "allow"   // 直接放行，不弹确认
}
```
```json
{
  "permissionDecision": "deny"    // 直接拒绝，无需用户确认
}
```
```json
{
  "permissionDecision": "ask"     // 弹出确认框（即使默认规则允许）
}
```
```json
{
  "permissionDecision": "defer"   // 交给下一个处理器判断
}
```

**实际用途**：
- `BashTool` 调用 `rm -rf /` → hook 直接返回 `deny`
- `EditTool` 修改 `.env` 文件 → hook 返回 `ask`（要求确认）
- `BashTool` 运行 `npm test` → hook 返回 `allow`（跳过确认）

### 修改工具输入（updatedInput）

```json
{
  "updatedInput": {
    "file_path": "/safe/path/file.ts",
    "new_string": "sanitized content"
  }
}
```

**实际用途**：
- 给文件路径加上项目前缀（防止越权）
- 在 Bash 命令中注入环境变量
- 自动添加 `--safe` 标志给危险命令

### 修改工具输出（updatedMCPToolOutput）

```json
{
  "updatedMCPToolOutput": {
    "content": "filtered output without secrets",
    "isError": false
  }
}
```

**实际用途**：
- 从工具输出中剥离 API key、密码
- 截断过长的输出（保护上下文预算）
- 将错误信息重写为人类友好的格式

### 注入额外上下文

```json
{
  "additionalContext": "⚠️ 注意：这个文件刚刚被另一个同事修改过，详见 git log -1 --follow"
}
```

`PostToolUse` 钩子可以注入上下文，这些内容会在下一轮模型调用时进入 prompt。

### 自动响应 MCP Elicitation

当 MCP 服务器通过 `Elicitation` 请求用户输入时，钩子可以自动代替用户回答：

```json
{
  "response": "approved",
  "reason": "自动批准已知的白名单操作"
}
```

---

## 第四层：异步执行与模型唤醒

```
用户请求 → 模型调用 Edit → PreToolUse hook（允许）
                                    ↓
                           Edit 执行成功
                                    ↓
                        PostToolUse hook（async: true）
                                    ↓
                           [后台运行 eslint]
                                    ↓
                         exit code = 2
                                    ↓
                          asyncRewake → 唤醒模型
                                    ↓
                      模型收到 eslint 的结果，继续处理
```

**关键设计**：
- `async: true` → hook 后台运行，不阻塞主对话
- exit code 0 → 正常完成，不唤醒模型
- exit code 1 → 失败，不唤醒模型
- exit code 2 → `asyncRewake`，将输出注入上下文并唤醒模型继续

这意味着：**耗时的检查（完整构建、集成测试）可以异步运行，只在发现问题时才打断主流程**。

---

## 提示词工程对比

| 方式 | Token 成本 | 可靠性 | 延迟影响 |
|------|-----------|--------|---------|
| 提示词规则 | 每轮消耗 | 依赖模型遵循 | 每轮增加思考时间 |
| Command hook | 0（触发时才执行） | 100% 程序化 | 可异步，零延迟 |
| Prompt hook | 触发时才消耗 | 小模型专用判断 | 可异步 |
| HTTP hook | 0（网络调用） | 外部系统保证 | 可异步 |

**提示词方式**：
```
You are a careful developer. After every file edit, you MUST run the linter.
Do not forget. This is mandatory. Run: npx eslint <file>
```
→ 每轮对话 ~50 tokens，模型可能"忘记"，需要反复强调。

**Hook 方式**：
```json
{
  "PostToolUse": {
    "matcher": "Edit",
    "hooks": [{ "type": "command", "command": "npx eslint $file", "async": true }]
  }
}
```
→ 零 token 成本，100% 触发，模型甚至不需要"知道"规则存在。

---

## 关键洞察

> **Hooks 的本质是把"提示词应该做的事"变成"系统一定会做的事"——token 省了，可靠性反而更高了。**

提示词管"意图"，Hooks 管"行为"。当你发现自己在反复强调同一条规则时，问自己：这条规则能不能变成一个事件钩子？如果能，就把它从提示词中移出去——模型会感谢你省下的 token 预算。

---

## 给抖音文案的启发

> "Claude 有一套 30 个事件的'外挂系统'。你可以告诉它：'每次改完文件，自动跑 lint'、'每次要删文件，先让我确认'、'每次运行 bash，自动过滤掉输出里的密码'。这些不是提示词，是代码。提示词是'建议'，Hooks 是'法律'——一条都不用消耗 token，一条都不会被忽视。"
