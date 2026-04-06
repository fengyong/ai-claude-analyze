# 技巧二十：A/B 测试提示词

> 核心思想：提示词改了之后，别猜效果好不好——用数据说话

---

## 问题本质

提示词工程最大的痛点：**你永远不知道你的改动到底是变好了还是变差了**。

传统软件开发有单元测试、集成测试、回归测试。但提示词的"测试"是什么？——开发者改完提示词，自己跑两三个案例，"感觉还行"，就上线了。

这就像一个厨师调完菜只自己尝一口，从不给顾客试吃。

Claude Code 的解法：**在产品里嵌入 GrowthBook 实验平台，对提示词变更进行 A/B 测试**。

---

## GrowthBook 集成架构

```
┌─────────────────────────────────────────────────┐
│               GrowthBook SDK                     │
│                                                  │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │ Feature Flags │    │ A/B Experiments      │   │
│  │ 开/关控制      │    │ 多变体对比            │   │
│  └──────┬───────┘    └──────────┬───────────┘   │
│         │                       │                │
│         ▼                       ▼                │
│  ┌──────────────────────────────────────────┐   │
│  │         S8() 评估函数                      │   │
│  │  输入: flag 名称                           │   │
│  │  输出: 变体值 + 是否已分桶                  │   │
│  └──────────────────────────────────────────┘   │
│         │                                        │
│         ▼                                        │
│  ┌──────────────────────────────────────────┐   │
│  │    trackedFeatureUsage / trackedExperiments│   │
│  │    数据回传 → 分析平台                      │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**源码**：`cli.js ~line 9411` — GrowthBook 客户端初始化

GrowthBook 的数据可以加密传输，运行时解密：

```typescript
// 加密的特性配置在 bundle 中
const encryptedFeatures = process.env.GROWTHBOOK_FEATURES_ENCRYPTED
// 运行时解密
const features = decryptFeatures(encryptedFeatures, decryptionKey)
```

这意味着实验配置可以嵌入到 CLI 的发布包中，不需要每次启动都请求服务器。

---

## 实验可以测什么

Anthropic 可以通过 GrowthBook 对以下维度进行 A/B 测试：

### 1. 行为指令对比

```
变体 A："Be concise. Use short sentences."
变体 B："Be thorough. Explain your reasoning."
变体 C：（无额外指令，使用默认）
```

### 2. 工具优先级排序

```
变体 A：优先使用 Grep 搜索 → 再用 Read 读取
变体 B：优先使用 Agent 探索 → 再定向读取
变体 C：优先使用 Glob 定位 → 再精准读取
```

### 3. 安全策略松紧

```
变体 A：严格模式 — 每个 Bash 命令都弹窗
变体 B：宽松模式 — 常见命令自动批准
变体 C：沙箱模式 — OS 级约束 + 自动批准
```

### 4. 反讨好机制效果

```
变体 A：加入反讨好提示词
变体 B：不加入反讨好提示词
测量指标：用户满意度、任务完成率、诚实度评分
```

| 实验类型 | 可测量指标 | 典型周期 |
|---------|-----------|---------|
| 行为指令 | 响应长度、任务完成时间 | 1-2 周 |
| 工具排序 | 工具调用次数、错误率 | 2-4 周 |
| 安全策略 | 阻断误报率、用户取消率 | 2-4 周 |
| 反讨好 | 用户满意度、任务重试率 | 4-8 周 |

---

## Sticky Bucketing：确保用户在同一实验组

### 问题

如果用户今天被分到"变体 A"，明天被分到"变体 B"，体验会极度不一致——用户会困惑"为什么今天的行为和昨天不一样"。

### 解决：Sticky Bucketing

```typescript
// 基于用户 ID 的确定性哈希分桶
function assignBucket(userId: string, experimentId: string): string {
  const hash = stableHash(`${userId}:${experimentId}`)
  return hash % 100 < 50 ? 'control' : 'variant'
}
```

一旦用户被分配到某个变体，在整个实验周期内保持不变。

| 场景 | 无 Sticky | 有 Sticky |
|------|----------|----------|
| 周一使用 | 变体 A | 变体 A |
| 周二使用 | 变体 B | 变体 A |
| 周三使用 | 变体 A | 变体 A |
| 用户体验 | 混乱、不可预测 | 稳定、一致 |

---

## tengu 代号揭秘：从 Flag 命名看内部实验

GrowthBook 的 flag 命名透露了 Anthropic 的内部代号文化：

```
tengu_surreal_dali               — "天狗"+"达利超现实"（可能是创意生成实验）
tengu_ccr_bundle_seed_enabled    — "CCR bundle 种子"（可能是上下文压缩相关）
tengu_prompt_suggestion_init     — "提示词建议初始化"（可能是主动建议功能）
```

**命名规律**：
- `tengu` 前缀 — 项目代号（天狗是日本传说中的妖怪）
- 中间部分 — 功能描述
- 尾部 — 状态（`enabled`、`init`、`seed`）

**源码**：`cli.js ~line 3628` — `S8()` 函数检查 flag 状态

```typescript
// S8() 是 GrowthBook flag 检查的核心函数
function S8(flagName: string): boolean {
  const feature = growthbook.getFeature(flagName)
  if (!feature.on) return false
  // ...sticky bucketing 逻辑...
  return feature.value
}

// 使用示例
if (S8('tengu_prompt_suggestion_init')) {
  // 启用新的提示词建议功能
}
```

---

## 从经验调优到数据驱动

传统提示词工程的工作流：

```
直觉 → 改提示词 → 自己测试 → "感觉不错" → 上线
```

Claude Code 的工作流：

```
假设 → 设计实验 → GrowthBook 分桶 → 收集数据 → 统计分析 → 决策
```

| 维度 | 经验调优 | 数据驱动 |
|------|---------|---------|
| 决策依据 | 开发者感觉 | 统计显著性 |
| 样本量 | 3-5 个测试用例 | 数万用户 |
| 变量控制 | 无（混杂变量多） | 严格（单一变量） |
| 回滚速度 | 重新发版 | 关闭 flag |
| 发现意外 | 几乎不可能 | 经常发现反直觉的结果 |

---

## 关键洞察

提示词工程从"我改了感觉挺好"进化到"A/B 测试证明变好了"——这是工程学科成熟的标志。
