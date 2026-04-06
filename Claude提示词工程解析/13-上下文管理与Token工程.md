# 掌控 200K 上下文：Claude Code 的 Token 工程

> 核心发现：不是"提问者要更精准"，而是系统用三层压缩 + 缓存分层，让你几乎不会真正"用完"上下文

---

## 回答最初的问题

"掌控大模型力量的核心是什么？"

**不是用户端的精准度**。Claude Code 的系统提示本身就占了 10-20K tokens 来"教模型怎么做"。用户输入只是对话的一小部分。

**是系统端的 Token 工程**——在 200K 的上下文窗口里，如何让每一层（系统提示、工具结果、对话历史、模型输出）高效共存。

---

## 第一层：Prompt Cache 分层——静态/动态分区

### 架构

```
getSystemPrompt() 组装系统提示：

┌─ 静态分区（7 段，对话期间不变）────────────────┐
│  1. intro — 简介                                │
│  2. system — 系统身份                           │
│  3. doing_tasks — 任务执行指导                  │
│  4. actions — 行为规范                          │
│  5. using_tools — 工具使用说明                  │
│  6. tone_and_style — 语气风格                   │
│  7. output_efficiency — 输出效率指导            │
│                                                 │
│  → 通过 Anthropic API prompt cache 缓存         │
│  → 每次对话只计费一次                            │
├─────────────────────────────────────────────────┤
│  __SYSTEM_PROMPT_DYNAMIC_BOUNDARY__             │
│  （缓存断点标记）                                │
├─ 动态分区（12 段，按需计算）────────────────────┤
│  session_guidance, memory, env_info,            │
│  language, output_style, mcp_instructions,      │
│  scratchpad, frc, summarize_tool_results,       │
│  numeric_length_anchors, token_budget, brief    │
│                                                 │
│  → 每段有独立 compute 函数                      │
│  → 计算一次后缓存，直到 /clear 或 /compact      │
└─────────────────────────────────────────────────┘
```

### 缓存基础设施

```typescript
// systemPromptSections.ts

// 创建 memoized section——计算一次后缓存
systemPromptSection(name, compute)

// 危险操作：每 turn 重新计算，破坏 prompt cache
// 需要提供理由说明
DANGEROUS_uncachedSystemPromptSection(name, compute, reason)

// 检查缓存，未命中时调用 compute()
resolveSystemPromptSections()

// /clear 和 /compact 时清除所有缓存
clearSystemPromptSections()
```

**关键洞察**：`DANGEROUS_` 前缀——Anthropic 把"破坏缓存"标记为危险操作，需要显式理由。这说明 prompt cache 的命中率是他们非常在意的优化指标。

### 缓存的价值

静态分区估计占 10-20K tokens。因为 prompt cache：
- **首次**：完整计费（写入缓存）
- **后续**：缓存命中部分只计 10% 的费用
- **效果**：200K 上下文窗口中，系统提示的"边际成本"很低

---

## 第二层：Token 预算系统——三级估算

### 估算体系

```
级别 1：countTokensWithAPI()
  → Anthropic countTokens API，精确
  → 最贵，不常用

级别 2：countTokensViaHaikuFallback()
  → 用 Haiku 模型做 token 计数
  → API 不可用时的 fallback

级别 3：roughTokenCountEstimation()
  → content.length / 4（默认 4 字符/token）
  → JSON 文件用 2 字节/token（因为 {, }, : 等密集单字符）
  → 图片固定估算 2000 tokens
  → 最快，最常用
```

### 上下文尺寸权威度量

`tokenCountWithEstimation()` 是所有阈值决策的依据：

```
逻辑：找到最后一个携带 usage 的 API 响应
      + 后续消息的 rough 估算
      = 当前上下文大小

特殊处理：并行工具调用
  → 模型并行调用 4 个工具时
  → streaming 为每个 content block 创建单独的 assistant 记录
  → 函数向后遍历到同 message.id 的第一个 sibling
  → 确保所有交错的 tool_result 都被计入
```

### 用户自定义 Token Budget

用户可以指定 token 目标：

```
"+500k"        → 继续工作直到消耗 500K tokens
"spend 2M"     → 目标 2M tokens
"use 1B"       → 目标 1B tokens
```

系统生成提示：
```
"Stopped at X% of token target (Y / Z). Keep working — do not summarize."
```

### 递减收益检测

```typescript
// query/tokenBudget.ts
// 连续 3 次且 delta < 500 tokens 时停止

if (consecutiveLowDelta >= 3 && delta < 500) {
  // 模型在空转，停止
}
```

---

## 第三层：三级压缩——从微压缩到全压缩

### 压缩层级总览

```
┌─────────────────────────────────────────────────┐
│  Level 0：微压缩（Microcompact）                  │
│  每轮对话自动执行                                 │
│  清除旧的工具返回结果                             │
│  → 释放 token，延迟全压缩                        │
├─────────────────────────────────────────────────┤
│  Level 1：API 微压缩（API Microcompact）          │
│  服务端执行                                       │
│  清除旧的工具调用和 thinking 块                   │
│  → 节省更多 token，进一步延迟全压缩              │
├─────────────────────────────────────────────────┤
│  Level 2：全压缩（Autocompact）                   │
│  触发阈值：187K tokens                           │
│  9 段式摘要 + 保留最近 5 个文件                   │
│  → 187K → 20-50K，重置上下文                     │
└─────────────────────────────────────────────────┘
```

### Level 0：微压缩

```
可压缩的工具：Read, Bash, Grep, Glob, WebSearch, WebFetch, Edit, Write

时间触发：距上一条助手消息超过阈值时
  → 将旧工具结果替换为 "[Old tool result content cleared]"
  → 保留消息结构，释放 token 空间

缓存触发：使用 API 的缓存编辑功能
  → 移除旧的工具返回结果
  → 但不破坏缓存前缀
```

### Level 1：API 微压缩

两种策略：

```
策略 1：clear_tool_uses_20250919
  → 清除旧的工具调用和结果

策略 2：clear_thinking_20251015
  → 清除 thinking 块

默认阈值：
  MAX_INPUT_TOKENS = 180,000
  TARGET_INPUT_TOKENS = 40,000

Thinking 块特殊策略：
  idle > 1h 时只保留最后 1 个 thinking turn
```

### Level 2：全压缩——触发条件

```
MODEL_CONTEXT_WINDOW_DEFAULT = 200,000 tokens
AUTOCOMPACT_BUFFER_TOKENS = 13,000 tokens

有效上下文窗口 = 200,000 - min(maxOutputTokens, 20,000)
触发阈值 = 有效上下文窗口 - 13,000

当已用 tokens > 187,000 时 → 触发全压缩
```

### 全压缩——9 段式摘要模板

```
<analysis>
  （模型在此组织思路，最终被剥离）
</analysis>

<summary>
  1. Primary Request and Intent
     用户的主要请求和意图

  2. Key Technical Concepts
     关键技术概念

  3. Files and Code Sections
     涉及的文件和代码段

  4. Errors and Fixes
     遇到的错误及修复

  5. Problem Solving
     问题解决过程

  6. All User Messages
     所有用户消息

  7. Pending Tasks
     待处理任务

  8. Current Work
     当前工作

  9. Optional Next Step
     可选的下一步
</summary>
```

### 全压缩——压缩后恢复

```
压缩前：模型读了 20 个文件

压缩后恢复：
  - 最多恢复 5 个最近读取的文件
  - 总 token 预算 50,000
  - 每文件最多 5,000 tokens
  - 排除已被保留消息中的 Read 结果（避免重复，最多节省 25K tokens/次）

Skill 恢复：
  - 每 skill 最多 5,000 tokens
  - 总预算 25,000 tokens
```

### PTL（Prompt Too Long）重试

压缩请求本身如果碰到 prompt-too-long：

```
truncateHeadForPTLRetry()
  → 丢弃最旧的 API round group
  → 最多重试 3 次
  → 3 次都失败 → 断路器，停止压缩
```

### 断路器保护

```
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

连续失败 3 次 → 停止尝试自动压缩
防止系统在极端情况下无限重试
```

---

## 第四层：冗余检测——知道你浪费了多少

### 重复文件读取检测

```typescript
// contextAnalysis.ts
// 当同一文件被多次 Read 时

fileReadStats.forEach((data, path) => {
  if (data.count > 1) {
    const averageTokensPerRead = Math.floor(data.totalTokens / data.count)
    const duplicateTokens = averageTokensPerRead * (data.count - 1)
    // → 记录重复消耗的 tokens
  }
})
```

### 上下文 Token 分布统计

```
analyzeContext() 遍历所有消息，统计：
  - toolRequests / toolResults — 按工具名称分类
  - humanMessages / assistantMessages — 人机消息
  - localCommandOutputs — 本地命令输出
  - attachments — 附件
  - duplicateFileReads — 重复文件读取

→ 转换为遥测指标，用于优化
```

---

## 第五层：用户端指导——从源头减少

### 输出效率（外部用户）

```
"Go straight to the point. Try the simplest approach first
 without going in circles. Do not overdo it. Be extra concise."
```

### 输出效率（Anthropic 内部）

```
"Length limits: keep text between tool calls to ≤25 words.
 Keep final responses to ≤100 words unless the task requires more detail."
```

**差异**：内部用户有硬性字数限制，外部用户只有模糊的"简洁"指令。这也是一种门控。

### MEMORY.md 截断

```
MAX_ENTRYPOINT_LINES = 200 行
MAX_ENTRYPOINT_BYTES = 25,000 字节
→ 双限制，先触发者为准
→ 防止记忆系统本身消耗过多上下文
```

---

## 整体工作流

```
用户输入
  │
  ▼
CLI Parser (main.tsx)
  │
  ▼
QueryEngine
  │
  ├──→ getSystemPrompt() 组装
  │    ├─ 静态分区（cached）
  │    ├─ 动态分区（memoized until /clear）
  │    └─ prompt cache: 'global'
  │
  ├──→ Anthropic API
  │    └─ prompt cache 命中 → 边际成本低
  │
  ├──→ Tool Loop 执行
  │    ├─ Read/Bash/Grep/Glob...
  │    ├─ 去重：mtime 检测避免重复读取
  │    └─ 结果大小限制：50K chars / 100K tokens
  │
  ├──→ tokenCountWithEstimation() 持续监测
  │    └─ 三级估算：API → Haiku fallback → rough
  │
  ├──→ 微压缩（每轮）
  │    └─ 清除旧工具结果
  │
  ├──→ API 微压缩（接近 180K）
  │    └─ 服务端清除旧调用和 thinking
  │
  ├──→ 全压缩（超过 187K）
  │    ├─ 9 段式摘要
  │    ├─ 恢复最近 5 个文件
  │    ├─ 断路器保护
  │    └─ PTL 重试机制
  │
  ▼
继续对话（压缩后重新读取需要的文件）
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| `src/constants/prompts.ts` | 系统提示组装、静态/动态分区 |
| `src/constants/systemPromptSections.ts` | 缓存基础设施 |
| `src/services/tokenEstimation.ts` | 三级 token 估算体系 |
| `src/utils/tokens.ts` | 上下文尺寸权威度量 |
| `src/utils/tokenBudget.ts` | 用户 token budget 解析 |
| `src/query/tokenBudget.ts` | Budget 追踪和递减收益检测 |
| `src/utils/context.ts` | 上下文窗口默认值（200K） |
| `src/utils/contextAnalysis.ts` | 上下文分析、重复读取检测 |
| `src/services/compact/autoCompact.ts` | 自动压缩触发、阈值计算、断路器 |
| `src/services/compact/prompt.ts` | 9 段式摘要模板 |
| `src/services/compact/compact.ts` | 核心压缩流程、PTL 重试 |
| `src/services/compact/microCompact.ts` | 客户端工具结果清除 |
| `src/services/compact/apiMicrocompact.ts` | 服务端上下文管理策略 |
| `src/memdir/memdir.ts` | MEMORY.md 截断机制 |

---

## 核心洞察

> **掌控 200K 上下文的关键不是"提问者要更精准"，而是系统用三层压缩 + 缓存分层，让你几乎不会真正"用完"上下文。**
>
> 系统提示本身占了 10-20K tokens（但缓存后边际成本低）。
> 工具结果是 token 消耗的大头（但有去重和微压缩）。
> 对话历史增长最快（但有三级压缩兜底）。
>
> 真正影响效果的不是"你说了多少字"，而是"你让模型读了什么文件"——因为 Read 工具返回的内容是上下文窗口里最大的消费者，而且压缩后只会保留最近 5 个。

---

## 给抖音文案的启发

> "你以为 200K 上下文是给你的？不是。系统自己先占了 10-20K 来教 AI 怎么工作。然后你每读一个文件，它就吃掉几千 tokens。对话太长时，Claude 会偷偷压缩——把你的对话变成 9 段摘要，只保留最近读的 5 个文件。所以你真正能用的上下文，远没有 200K 那么多。"
