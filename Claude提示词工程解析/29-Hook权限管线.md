# 技巧二十九：分布式授权——安全决策从提示词到外部流程

> 核心思想：确定性安全 > 概率性安全——把安全从"AI 建议"升级为"代码强制"

---

## 问题本质

在传统的提示词安全模型中，安全规则是这样工作的：

```
系统提示词："不要删除 .git 目录"
模型：读到了这条规则
用户："帮我清理一下 .git"
模型：拒绝（概率性）
```

**问题在于"概率性"这三个字。** 模型大概率会遵守规则，但不是 100%。面对巧妙的 prompt injection、复杂的多步操作、或者边缘场景，模型可能"忘记"安全规则。

更深层的问题是：**安全规则写在提示词里，每次对话都消耗 token。** 一条"不要删除 .git"的规则，每次对话都要花 token 传递给模型——即使用户从来没要求删除 .git。

**Hook 权限管线**（Hook Permission Pipeline）的解法是：**把安全决策从提示词层迁移到代码层。** 不靠 AI 自觉，靠确定性代码。

---

## 权限管线架构

### 四阶段管线

工具调用的完整生命周期中，hooks 在四个阶段可以介入安全决策：

```
PreToolUse Hook ──→ 权限检查 ──→ 工具执行 ──→ PostToolUse Hook
      │                │                          │
      │                ▼                          ▼
      │         PermissionRequest Hook      PermissionDenied Hook
      │                │                          │
      ▼                ▼                          ▼
  可以拒绝       可以修改输入              可以触发重试
  可以放行       可以追加规则
  可以延迟
```

**源码**：cli.js ~line 81750（事件联合类型），~line 81864（PermissionSync）

### 阶段详解

| 阶段 | 时机 | 能做什么 |
|------|------|---------|
| PreToolUse | 工具执行前 | 拒绝、放行、延迟、修改输入、追加规则 |
| PermissionRequest | 权限弹窗触发时 | 允许（可修改输入）、拒绝 |
| PostToolUse | 工具执行后 | 验证结果、触发副作用 |
| PermissionDenied | 权限被拒绝后 | 决定是否重试 |

---

## Hook 能做的四件事

PreToolUse hooks 的权限决策不是一个简单的"允许/拒绝"二选一——它是一个**四态决策系统**：

### 决策 1：permissionDecision

```typescript
// PreToolUse hook 返回值
{
  permissionDecision: 'allow'   // 直接放行，不问用户
  permissionDecision: 'deny'    // 直接拒绝，不问用户
  permissionDecision: 'ask'     // 问用户（弹出确认框）
  permissionDecision: 'defer'   // 延迟到后续 hook 或默认逻辑
}
```

| 决策 | 行为 | 使用场景 |
|------|------|---------|
| `allow` | 静默放行 | 符合企业安全策略的操作 |
| `deny` | 静默拒绝 | 明确违规的操作（如删除关键文件） |
| `ask` | 弹出确认框 | 不确定是否安全的操作 |
| `defer` | 跳过，让下一个 hook 或默认逻辑处理 | 只关心特定操作的 hook |

### 决策 2：输入修改

```typescript
// PermissionRequest hook 可以修改工具输入
{
  behavior: 'allow',
  updatedInput: {
    // 修改后的工具参数
    // 例如：将绝对路径转换为沙箱路径
  },
  updatedPermissions: [
    // 追加新的权限规则
  ]
}
```

**效果**：Hook 不只是门卫（让不让过），还是翻译官（修改请求内容后再过）。

### 决策 3：权限规则累积

```typescript
// Hook 可以动态追加权限规则
{
  updatedPermissions: [
    { tool: 'Bash', pattern: 'npm test', action: 'allow' },
    { tool: 'Bash', pattern: 'rm -rf', action: 'deny' },
  ]
}
```

这意味着权限规则不是静态配置的——**Hook 可以根据运行时状态动态生成规则**。比如：检测到用户在测试目录中操作时，自动允许所有测试相关命令。

### 决策 4：重试机制

```typescript
// PermissionDenied hook
{
  retry: true  // 通知系统重试被拒绝的操作
}
```

**场景**：操作被拒绝后，Hook 可以先做一些准备工作（比如修改文件权限、切换目录），然后告诉系统"现在可以重试了"。

### 完整决策矩阵

| 能力 | PreToolUse | PermissionRequest | PostToolUse | PermissionDenied |
|------|-----------|------------------|-------------|-----------------|
| 允许/拒绝 | ✅ | ✅ | — | — |
| 修改输入 | — | ✅ | — | — |
| 追加规则 | — | ✅ | — | — |
| 触发重试 | — | — | — | ✅ |
| 验证结果 | — | — | ✅ | — |

---

## 确定性安全 vs 概率性安全

### 概率性安全（提示词模式）

```
系统提示词："不要允许删除 .git 目录"
→ 模型大概率遵守
→ 但面对复杂场景可能失误
→ 每次对话消耗 token
→ 无法保证 100% 执行
```

### 确定性安全（Hook 模式）

```typescript
// PreToolUse hook
if (tool === 'Bash' && command.includes('.git')) {
  return { permissionDecision: 'deny' }
}
```

→ **代码决定**，不是 AI 决定
→ **100% 执行**，不存在"模型偶尔忘记"
→ **不消耗 token**，在代码层拦截
→ **不需要模型参与**，直接返回结果

### 对比

| 维度 | 提示词安全 | Hook 安全 |
|------|-----------|----------|
| 执行保证 | 概率性（~95-99%） | 确定性（100%） |
| Token 消耗 | 每次对话都消耗 | 零 token 消耗 |
| 复杂场景 | 可能绕过 | 代码级不可绕过 |
| 可审计性 | 难以审计 | 可日志记录每次决策 |
| 动态规则 | 静态写死在 prompt | 运行时动态生成 |

**类比**：就像交通安全——"请勿超速"是提示词安全（概率性遵守），而限速带是 Hook 安全（确定性强制）。你可以不看路边的限速标志，但你不可能飞过限速带。

---

## PermissionSync：分布式代理的权限集中

当多个代理（Swarm/Team）协同工作时，权限管理变得更加复杂——每个代理都有独立的上下文窗口，安全规则不可能在每个代理的提示词中都完整重复。

**PermissionSync** 解决了这个问题：**将权限审批集中到主会话（Team Lead）**。

### 工作流程

```
Teammate Agent 执行操作
    ↓
触发权限请求
    ↓
PermissionSync 发送消息到主会话（Mailbox）
    ↓
Team Lead 审批（基于完整上下文）
    ↓
决策同步回 Teammate
    ↓
Teammate 执行或取消
```

**源码**：cli.js ~line 81864

### 设计精妙

| 设计选择 | 为什么 |
|---------|--------|
| 集中审批 | Team Lead 有最完整的上下文，最适合做安全决策 |
| Mailbox 通信 | 异步消息传递，不阻塞其他 teammate |
| 决策缓存 | `user_permanent` 决策跨会话持久化，避免重复询问 |
| 信任传递 | Team Lead 的决策自动应用到所有 teammate |

### 决策持久化

用户的权限决策分为三类：

```typescript
user_temporary  // 允许一次，下次还问
user_permanent  // 永久允许，跨会话持久化
user_reject     // 拒绝，记录偏好
```

`user_permanent` 决策被持久化到磁盘——下次会话启动时，已经被永久允许的操作不再弹窗。这在不影响安全性的前提下减少了用户摩擦。

---

## 输入转换：Hook 不只是守门员

Hook 权限系统中一个容易被忽视的能力是**输入转换**（Input Transformation）：

```typescript
// Hook 不只是决定"能不能执行"，还可以修改"执行什么"
{
  behavior: 'allow',
  updatedInput: {
    path: '/sandbox/restricted-path',  // 重定向路径
    content: sanitizedContent,          // 清洗后的内容
  }
}
```

这意味着 Hook 可以作为一个**安全代理**——它检查请求，修改不安全的部分，然后放行安全的版本。模型以为自己在执行原始操作，实际执行的是 Hook 修改后的安全版本。

**类比**：就像给孩子吃药——孩子以为自己吃的是糖（原始请求），实际上妈妈把药包在了糖衣里（Hook 修改后的安全版本）。孩子不知道，AI 也不知道。

---

## 提示词工程技巧总结

| 技巧 | 具体做法 | 效果 |
|------|---------|------|
| 四态决策 | allow/deny/ask/defer | 细粒度权限控制，不是二元判断 |
| 输入转换 | 修改工具参数后放行 | 安全代理，透明修正 |
| 规则累积 | 运行时动态追加权限规则 | 基于上下文的安全策略 |
| 集中审批 | PermissionSync 到 Team Lead | 分布式代理的安全统一 |
| 决策持久化 | user_permanent 跨会话保存 | 减少重复确认摩擦 |
| 确定性安全 | 代码层拦截替代 prompt 层建议 | 100% 执行保证 |

---

## 给抖音文案的启发

> "Claude 的安全规则以前写在提示词里——相当于在墙上贴了张'请勿偷窃'的标语。现在呢？Claude 把安全从'请勿偷窃'升级成了'防盗门'。提示词是标语，Hook 是门锁。标语可能被人忽视，但门锁你绕不过去。"

---

## 关键洞察

确定性安全优于概率性安全——Hook 权限管线把安全从"AI 建议遵守"升级为"代码强制执行"，这是提示词工程中最重要的架构进化之一。
