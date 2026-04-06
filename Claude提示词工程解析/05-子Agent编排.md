# 技巧五：子 Agent 编排

> 核心思想：一个 AI 不够用时，让它派生专业化的"小弟"

---

## Agent 架构总览

Claude Code 有**两种 Agent 模式**：

```
┌─────────────────────────────────────────┐
│              主 Agent (Claude)           │
│                                         │
│  ┌──────────────┐  ┌──────────────┐     │
│  │ Fork Agent   │  │ Spawn Agent  │     │
│  │ 继承上下文    │  │ 零上下文      │     │
│  │ 共享缓存     │  │ 独立会话      │     │
│  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────┘
```

### Fork vs Spawn

| 特性 | Fork（分叉） | Spawn（生成） |
|------|------------|-------------|
| 上下文 | 继承主 Agent 全部上下文 | 从零开始 |
| 缓存 | 共享前缀缓存 | 独立缓存 |
| 用途 | 需要上下文的子任务 | 完全独立的探索/执行 |
| 通信 | 通过 output_file | 通过 output_file |

---

## 六大内置 Agent

### 1. Explore Agent（探索代理）

**定位**: 只读代码探索专家

**核心约束**:
```
STRICTLY PROHIBITED from creating, modifying, or deleting files.
```

**能力**:
- Glob 模式匹配
- 正则搜索（Grep）
- 并行工具调用
- 多种深度级别：quick / medium / very thorough

**模型选择**: 外部用户使用 `haiku`（更快、更便宜），Anthropic 内部用户使用完整模型。

**Why**: 探索不需要创造力，需要的是速度和覆盖面。Haiku 足够。

### 2. Verification Agent（验证代理）

**定位**: 对抗性验证——"Your job is not to confirm, it's to try to break it"

**核心理念**:
```
Two documented failure patterns:
1. Verification avoidance — "I tested it, trust me"
2. Seduced by first 80% — looks good at first glance, breaks on edge cases
```

**自我反制**:
```
Recognize your own rationalizations:
- "It looks like it should work"
- "The code is similar to what worked before"
- "I don't need to test that edge case"
```

**输出格式强制**:
```
Required format for each test:
- Command run
- Output observed
- PASS / FAIL / PARTIAL

Before PASS: must include at least one adversarial probe.
```

**工具限制**: 禁止使用 Agent, PlanMode, Edit, Write, NotebookEdit——确保它是纯验证角色。

### 3. General Purpose Agent（通用代理）

**定位**: 什么都能干的全能代理

**工具权限**: `['*']`（所有工具）

**输出约束**:
```
Respond with a "concise report covering what was done" since the caller
relays it to the user.
```

**共享前缀**: 使用 `SHARED_PREFIX` + `SHARED_GUIDELINES`，与其他 Agent 共享基础指令。

**反膨胀规则**:
```
"Complete the task fully — don't gold-plate, but don't leave it half-done."
"NEVER create files unless absolutely necessary"
"NEVER proactively create documentation files"
```

### 4. Plan Agent（规划代理）

**定位**: 只读的架构规划师

**硬约束**:
```
CRITICAL: You MUST NOT create, modify, delete, or move any files.
This includes temporary files and using redirect operators.
```

**输出格式**: 必须以 "Critical Files for Implementation" 节结尾，列出 3-5 个关键文件。

**工具限制（防御纵深）**:
```typescript
disallowedTools: ['Agent', 'ExitPlanMode', 'FileEdit', 'FileWrite', 'NotebookEdit']
```

不只是 prompt 说"别改文件"，还在工具层面**物理禁止**了改文件的能力。

**Token 优化**: `omitClaudeMd: true`——跳过加载 CLAUDE.md，因为只读代理可以自己 Read 它。

### 5. Claude Code Guide Agent（文档专家）

**定位**: Claude Code / Agent SDK / Claude API 三域文档专家

**动态上下文注入**:
```typescript
// 在系统 prompt 中注入用户的实际配置
- 自定义 skills 列表
- 自定义 agents 列表
- MCP 服务器列表
- 插件命令
- settings.json 内容
```

**模型**: 使用 `haiku` + `dontAsk` 权限模式——轻量级且不需要交互。

**复用优先**:
```
"Check for an existing claude-code-guide agent before spawning a new one"
```

### 6. Statusline Setup Agent（状态栏配置代理）

**定位**: PS1 到 Claude Code statusLine 的转换专家

**技术细节**:
- 嵌入完整的 statusLine stdin JSON schema
- 提供 PS1 转义序列表（`\u` → `$(whoami)`, `\w` → `$(pwd)`）
- 包含 ANSI 颜色处理指导
- 终端颜色感知：状态栏使用暗色，避免过亮

---

## Agent 编排的提示词工程技巧

### 技巧 1：共享前缀模式

```typescript
const SHARED_PREFIX = `You are a specialized agent...`
const SHARED_GUIDELINES = `1. Be concise... 2. Focus on the task...`

// 多个 Agent 复用
GENERAL_PURPOSE_AGENT.systemPrompt = SHARED_PREFIX + SHARED_GUIDELINES + specific
EXPLORE_AGENT.systemPrompt = SHARED_PREFIX + SHARED_GUIDELINES + specific
```

**Why**: 避免在多个 Agent prompt 中重复相同的通用指令，改一处全局生效。

### 技巧 2：防御纵深——工具层 + Prompt 层双重限制

Plan Agent 的约束同时在两层生效：
1. **Prompt 层**: "You MUST NOT create or modify files"
2. **工具层**: `disallowedTools` 物理移除写入工具

**Why**: 只靠 prompt 可能被 jailbreak 绕过，工具层限制是物理保障。

### 技巧 3：输出格式强制

Verification Agent 强制要求：
```
- Command run: ...
- Output observed: ...
- PASS / FAIL / PARTIAL
```

**Why**: 结构化输出让主 Agent 能可靠地解析子 Agent 的结果，而不是从自然语言中猜测。

### 技巧 4：给 Agent 的 Prompt 写作指南

`src/tools/AgentTool/prompt.ts` 中对主 Agent 的指导：

```
"Brief the agent like a smart colleague who just walked into the room"
"Never delegate understanding — don't write 'based on your findings, fix the bug'"
"Launch multiple agents in parallel when possible"
```

这实际上是**meta-prompting**——用 prompt 教 AI 如何写更好的 prompt。

---

## 提示词工程技巧总结

| 技巧 | 具体做法 | 效果 |
|------|---------|------|
| 共享前缀 | 提取通用指令为常量 | 减少重复，统一修改 |
| 防御纵深 | prompt 限制 + 工具物理限制 | 双保险 |
| 结构化输出 | 强制特定格式 | 主 Agent 可靠解析 |
| Meta-prompting | 用 prompt 教 AI 写 prompt | 自举能力 |
| 模型分级 | 探索用 haiku，执行用 opus | 成本/速度优化 |
| 复用优先 | "先检查是否存在，再创建" | 避免资源浪费 |

---

## 给抖音文案的启发

> "Claude 不是一个人在战斗。它会派出专门的'探子'去侦察代码，派出'审计员'去挑毛病，派出'军师'去做规划。每个小弟都有自己的规矩——有的只能看不能动，有的必须尝试打破一切。"
