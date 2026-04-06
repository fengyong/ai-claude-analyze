# 技巧二十三：数据飞轮——提示词工程的可观测性

> 核心思想：没有数据的提示词优化是盲人摸象，OpenTelemetry 让每一次交互都变成优化信号

---

## 问题本质

你改了一段系统提示，模型的行为似乎变好了——但"似乎"是多少？

- 把"请简洁回答"改成"回答不超过 100 字"，效果提升了还是下降了？
- 加了 `DANGEROUS_uncachedSystemPromptSection` 之后，缓存命中率降了多少？
- 压缩后的 9 段式摘要，哪一段对后续任务成功率影响最大？

**没有可观测性，这些全是猜的。**

Claude Code 内置了完整的 OpenTelemetry 遥测体系，把每一次工具调用、每一次 token 消耗、每一次代码编辑决策都变成可查询的数据点。

---

## 第一层：全链路指标一览

**源码**：Meter ~line 697, OTel schema ~line 35302

### 核心计数器

| 指标名 | 类型 | 含义 | 提示词工程价值 |
|--------|------|------|---------------|
| `claude_code.session.count` | Counter | 会话总数 | 衡量用户活跃度 |
| `lines_of_code.count` | Counter | 生成的代码行数 | 衡量产出效率 |
| `pull_request.count` | Counter | 创建的 PR 数 | 衡量实际交付 |
| `commit.count` | Counter | 提交次数 | 衡量代码变更频率 |
| `cost.usage` | Counter | API 费用消耗 | 成本优化基准 |
| `token.usage` | Counter | Token 消耗量 | 上下文效率度量 |
| `code_edit_tool.decision` | Counter | 编辑工具决策（接受/拒绝） | **衡量模型编辑质量** |
| `active_time.total` | Counter | 用户活跃时间 | 区分等待 vs 工作 |

### 自定义事件

```typescript
// 工具结果配对修复事件
tengu_tool_result_pairing_repaired
  → 当 tool_use/tool_result 不匹配时自动修复
  → 追踪"会话自愈"频率

// Agent 记忆加载事件
tengu_agent_memory_loaded
  → 当 Agent 加载 MEMORY.md 等记忆文件时触发
  → 追踪记忆系统的使用效果

// 工具决策事件
tool_decision
  → 记录每次工具调用的决策（批准/拒绝/自动）
  → 追踪权限系统的安全性
```

---

## 第二层：数据飞轮——从观测到优化的循环

```
                    ┌──────────────┐
                    │   观测       │
                    │  (Observe)   │
                    └──────┬───────┘
                           │
              OpenTelemetry 指标采集
                           │
                           ▼
                    ┌──────────────┐
                    │   分析       │
                    │  (Analyze)   │
                    └──────┬───────┘
                           │
           哪些 prompt section → 更高的完成率？
           哪些工具决策 → 更高的拒绝率？
           哪些上下文压缩策略 → 更少的信息丢失？
                           │
                           ▼
                    ┌──────────────┐
                    │   优化       │
                    │  (Optimize)  │
                    └──────┬───────┘
                           │
           调整 prompt 内容
           修改工具描述
           调整压缩阈值
                           │
                           ▼
                    ┌──────────────┐
                    │   再观测     │
                    │  (Re-observe)│
                    └──────┬───────┘
                           │
                           └──────→ 循环
```

### 关联分析示例

```typescript
// 追踪：哪些 prompt section 的修改带来了更高的 code_edit_tool 成功率？
correlate(
  systemPromptSectionChanges,    // prompt 修改历史
  code_edit_tool.decision,       // 编辑决策结果
  timeWindow: "7d"               // 7 天窗口
)

// 追踪：上下文压缩后，工具调用的成功率是否下降？
correlate(
  autocompact_events,            // 压缩事件
  tool_error_rate,               // 工具错误率
  lag: 3                         // 压缩后 3 轮内的错误率
)
```

---

## 第三层：上下文利用率分析

**源码**：usage O5K ~line 81750

API 响应中携带的 usage 数据是上下文管理的"体检报告"：

```typescript
// API 响应中的 usage 字段
{
  input_tokens: 45230,
  output_tokens: 1200,
  cache_creation_input_tokens: 0,      // 缓存命中，无需重建
  cache_read_input_tokens: 42000,      // 42K tokens 从缓存读取

  // 上下文窗口元数据
  contextWindow: 200000,               // 模型上下文窗口大小
  maxOutputTokens: 8192                // 最大输出 token 数
}
```

### 上下文利用率计算

```
上下文利用率 = input_tokens / contextWindow
             = 45,230 / 200,000
             = 22.6%

缓存命中率 = cache_read_input_tokens / input_tokens
           = 42,000 / 45,230
           = 92.9% ← 非常高，省钱

边际成本 = cache_read 价格 × 0.1（缓存命中只计 10%）
         vs 完整价格 × 1.0
```

### 追踪自动压缩窗口

```typescript
// autoCompactWindow 可配置范围：100K ~ 1M tokens
autoCompactWindow: 200000   // 默认
autoCompactWindow: 500000   // 大上下文模型
autoCompactWindow: 1000000  // 最大

// 不同窗口的压缩策略对比
| 窗口大小 | 触发阈值 | 压缩频率 | 信息保留度 |
|---------|---------|---------|-----------|
| 100K    | ~87K    | 高      | 低        |
| 200K    | ~187K   | 中      | 中        |
| 500K    | ~487K   | 低      | 高        |
| 1M      | ~987K   | 极低    | 极高      |
```

---

## 第四层：企业级可观测——otelHeadersHelper

对于企业用户，Claude Code 支持将遥测数据路由到自有的后端：

```typescript
// 企业配置：将遥测数据发送到自己的 OpenTelemetry collector
{
  "otel": {
    "endpoint": "https://telemetry.mycompany.com/v1/metrics",
    "headers": {
      "Authorization": "Bearer <enterprise-token>",
      "X-Tenant-ID": "engineering-team"
    }
  }
}
```

### 企业遥测架构

```
Claude Code 实例（数百个开发者）
       │
       │  OpenTelemetry 协议
       ▼
企业 OTel Collector
       │
       ├──→ Prometheus/Grafana（实时仪表盘）
       │     - 团队级别的 token 消耗
       │     - 代码编辑接受率趋势
       │     - 平均任务完成时间
       │
       ├──→ 数据仓库（长期分析）
       │     - prompt 优化的 A/B 测试
       │     - 成本归因（哪个项目烧钱最多）
       │
       └──→ 告警系统
             - 异常高的工具拒绝率 → 可能的 prompt 退化
             - 异常高的压缩频率 → 上下文管理需要调优
```

**为什么这对企业重要**：当 100 个开发者同时使用 Claude Code 时，没有遥测你就不知道：
- 谁在浪费 token（重复读取大文件）
- 哪些团队的 prompt 配置效果最好
- 整体的投入产出比是多少

---

## 提示词工程技巧总结

| 技巧 | 具体做法 | 效果 |
|------|---------|------|
| 全链路指标 | OpenTelemetry 覆盖每个操作 | 可量化的提示词效果 |
| 关联分析 | prompt 变更 × 工具决策率 | 找到因果关系而非相关性 |
| 上下文利用率 | usage 数据实时计算 | 知道何时需要压缩 |
| 企业遥测 | otelHeadersHelper 自有后端 | 团队级别的效果对比 |
| 数据飞轮 | 观测→分析→优化→再观测 | 持续改进的正循环 |

---

## 关键洞察

> **没有可观测性的提示词工程是盲人摸象——你改了一段 prompt，然后凭感觉判断"好像变好了"。有了 OpenTelemetry，每一次工具决策、每一行代码生成、每一笔 API 费用都变成可查询的数据点。提示词工程从"手艺活"变成"数据驱动的工程学科"。**
