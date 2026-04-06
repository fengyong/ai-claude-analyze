# 技巧二十五：服务端兜底——API 也管上下文

> 核心思想：客户端压缩再精巧也可能漏掉边界情况，API 层面的 context_management 是最后的安全网

---

## 问题本质

前面的文章（技巧七、十三）详细分析了 Claude Code 客户端的三层压缩体系：微压缩、API 微压缩、全压缩。这套系统很精巧，但它有一个根本性的假设——**客户端总是能正确判断何时需要压缩**。

现实中的反例：

```
反例 1：客户端 token 估算不准
  → roughTokenCountEstimation() 用 content.length / 4 估算
  → 对于非英语内容（如中文、日文），这个估算偏差很大
  → 客户端以为还有 30K tokens 的余量
  → 实际已经接近上限

反例 2：并行工具调用的突发膨胀
  → 模型一次性并行调用 8 个 Read 工具
  → 每个 Read 返回 10K 字符
  → 80K 字符突然涌入上下文
  → 客户端来不及压缩

反例 3：外部工具的不可控输出
  → MCP 工具返回了意外大的结果
  → Bash 命令输出了 50K 行日志
  → 客户端的截断机制可能没有覆盖到
```

**如果只靠客户端管理上下文，这些边界情况可能导致请求直接失败。**

---

## 第二层：服务端 context_management 字段

**源码**：context_management ~line 1261

API 响应中包含 `context_management` 字段，这是服务端对上下文状态的主动管理：

```typescript
// API 响应结构（message_delta handler）
{
  type: "message_delta",
  delta: {
    stop_reason: "end_turn",
    text: "模型的回复内容"
  },

  // 服务端上下文管理指令
  context_management: {
    // 策略 1：清除旧的工具调用
    clear_tool_uses: {
      enabled: true,
      trigger_threshold: 180000,    // 接近上限时触发
      target_tokens: 40000          // 清除到这个大小
    },

    // 策略 2：清除旧的 thinking 块
    clear_thinking: {
      enabled: true,
      idle_threshold: 3600          // 空闲 1 小时后清除
    }
  }
}
```

### 服务端能做什么

```
服务端能力：

  1. 截断（Truncate）
     → 从消息头开始丢弃最旧的消息
     → 客户端收到截断指令后重建消息列表

  2. 摘要（Summarize）
     → 对旧消息生成摘要替代原文
     → 类似客户端的全压缩，但在服务端执行

  3. 清除（Clear）
     → 移除工具调用/结果的具体内容
     → 保留消息骨架，释放 token 空间
```

---

## 第三层：客户端-服务端协同——谁先压缩、谁兜底

### 协同架构

```
客户端压缩链：

  Level 0: 微压缩（每轮自动）
       │ 清除旧工具结果
       ▼
  Level 1: API 微压缩（接近 180K）
       │ 服务端清除旧调用和 thinking
       ▼
  Level 2: 全压缩（超过 187K）
       │ 9 段式摘要 + 保留最近 5 个文件
       ▼
  如果全压缩请求本身 too long...
       │
       ▼
  服务端兜底：
    → context_management 指令生效
    → 截断或摘要旧消息
    → 确保响应不超限
```

### 责任划分

| 场景 | 谁负责 | 机制 |
|------|--------|------|
| 每轮对话后清理旧工具结果 | 客户端 | 微压缩 |
| 上下文接近 180K 时清理 | 服务端 | API 微压缩策略 |
| 上下文超过 187K 时全量压缩 | 客户端 | 全压缩（9 段式摘要） |
| 全压缩请求本身超长 | 服务端 | PTL 截断 + 重试 |
| 客户端估算偏差导致漏判 | 服务端 | context_management 兜底 |
| 极端情况（所有压缩失败） | 服务端 | 断路器 + 错误响应 |

**核心原则**：客户端是"第一道防线"（主动、精细），服务端是"最后的安全网"（被动、粗暴但有效）。

---

## 第四层：autoCompactWindow 的可调窗口

**源码**：autoCompactWindow ~line 12032

`autoCompactWindow` 是一个关键的可配置参数，决定了"多大的上下文算大"：

```typescript
// 默认值：200K tokens（匹配 Claude Opus 的上下文窗口）
const MODEL_CONTEXT_WINDOW_DEFAULT = 200_000

// 用户可配置范围
autoCompactWindow: number  // 100K ~ 1M

// 不同配置下的行为
```

### 配置对比

| autoCompactWindow | 触发阈值 | 适用场景 |
|-------------------|---------|---------|
| 100K | ~87K | 快速对话、短任务 |
| 200K（默认） | ~187K | 标准使用 |
| 500K | ~487K | 大型代码库探索 |
| 1M | ~987K | 超长自主任务 |

### 上下文窗口信息回传

API 在 usage 数据中回传上下文窗口信息，帮助客户端做出正确决策：

```typescript
// API usage 响应
{
  usage: {
    input_tokens: 145000,
    output_tokens: 4000,
    // 上下文窗口元数据
    contextWindow: 200000,       // 模型的实际上下文窗口
    maxOutputTokens: 8192        // 模型的最大输出限制
  }
}

// 客户端据此计算
effectiveWindow = contextWindow - min(maxOutputTokens, 200000)
                 = 200000 - 8192
                 = 191808

triggerThreshold = effectiveWindow - AUTOCOMPACT_BUFFER_TOKENS (13000)
                  = 191808 - 13000
                  = 178808

// 当 input_tokens > 178808 时，触发全压缩
```

**注意**：`maxOutputTokens` 被从有效窗口中扣除——因为输出也要占上下文空间。如果模型要生成 8K tokens 的回复，那你输入就不能用满 200K。

---

## 第五层：安全网的价值——不是替代，是补充

### 类比：飞机的多重安全系统

```
客户端压缩 = 飞行员的主动操作
  → 正常情况下，飞行员（客户端）控制一切
  → 精细、高效、可预测

服务端兜底 = 自动驾驶仪 + 地面碰撞避免系统
  → 飞行员失误或没注意到时介入
  → 粗暴但有效，保命优先
  → 触发时说明已经接近危险边界
```

### 实际效果

```
没有服务端兜底的世界：
  客户端估算偏差 → 请求超限 → API 返回 400 错误
  → 用户看到 "Request too long" 错误
  → 对话可能需要重新开始

有服务端兜底的世界：
  客户端估算偏差 → 服务端 context_management 介入
  → 自动截断/摘要旧消息 → 请求成功处理
  → 用户无感知，对话继续
  → 遥测记录事件，用于后续优化
```

---

## 提示词工程技巧总结

| 技巧 | 具体做法 | 效果 |
|------|---------|------|
| 服务端管理 | context_management 字段 | 被动兜底，防止请求超限 |
| 可调窗口 | autoCompactWindow 100K~1M | 适配不同任务规模 |
| 协同设计 | 客户端先压缩，服务端兜底 | 双重安全网 |
| 元数据回传 | contextWindow + maxOutputTokens | 客户端精确计算触发阈值 |
| 粗暴保底 | 截断 > 报错 | 用户体验优先于完美 |

---

## 关键洞察

> **上下文管理不是纯客户端责任——API 是最后的安全网。客户端的三层压缩负责精细化管理，服务端的 context_management 负责兜底。两者的协同确保了：即使客户端的 token 估算偏差、即使并行工具调用突发膨胀、即使全压缩请求本身超长——对话都不会因为"太长"而中断。这不是冗余设计，而是分布式系统的标准安全模式：每个层级都应该假设上游可能失败。**
