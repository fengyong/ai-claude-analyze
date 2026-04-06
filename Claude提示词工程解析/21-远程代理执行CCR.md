# 技巧二十一：云端代理——同一份提示词，不同的执行环境

> 核心思想：系统提示词是可移植的——同一份"咒语"，不同的"法场"

---

## 问题本质

有些任务本地跑不动：

- **太重**：全量构建、大规模测试套件、代码迁移
- **太慢**：需要跑 30 分钟的 CI pipeline
- **太占资源**：需要 64GB 内存的编译任务
- **需要隔离**：在干净环境中验证，不受本地配置影响

传统方案：手动 SSH 到远程机器，手动操作。但这样就失去了 Claude Code 的整个工具链——提示词、工具、权限系统全部失效。

CCR（Claude Code Remote）的解法：**把本地的完整执行环境（提示词 + 工具 + 权限 + 代码状态）打包，发送到云端执行，结果拉回本地审查**。

类比：你有一份菜谱（提示词）和一套厨具（工具），但你的厨房太小（本地资源不够）。CCR 让你把菜谱和厨具搬到大厨房（云端），做完菜再端回来。

---

## CCR 架构：本地触发 → 云端执行 → 本地审查

```
┌──────────────────────┐         ┌──────────────────────┐
│      本地 Claude       │         │      云端 Claude      │
│                      │  Bundle  │                      │
│  ┌────────────────┐  │ ──────► │  ┌────────────────┐  │
│  │ 系统提示词      │  │         │  │ 同一份提示词     │  │
│  │ 工具定义        │  │         │  │ 同一套工具      │  │
│  │ 权限配置        │  │         │  │ 同样权限        │  │
│  │ 代码快照        │  │         │  │ 代码副本        │  │
│  └────────────────┘  │         │  └────────────────┘  │
│                      │         │                      │
│  1. 触发任务         │         │  2. 云端执行         │
│  4. 本地审查         │ ◄────── │  3. 结果回传         │
└──────────────────────┘         └──────────────────────┘
```

---

## API 端点

CCR 通过 claude.ai API 进行通信：

```
GET  /v1/code/triggers              — 列出所有远程触发器
POST /v1/code/triggers/{id}/run     — 执行指定远程任务
```

**源码**：`cli.js ~line 3628` — RemoteTrigger 定义

```typescript
// 列出可用的远程触发器
const triggers = await fetch('/v1/code/triggers', {
  headers: { 'Authorization': `Bearer ${token}` }
})

// 执行远程任务
const result = await fetch(`/v1/code/triggers/${triggerId}/run`, {
  method: 'POST',
  body: JSON.stringify({
    bundle: localBundle,
    branch: 'feature/remote-task',
    instructions: '运行完整测试套件并修复所有失败的测试'
  })
})
```

---

## Bundle 传输：如何把本地状态搬到云端

Bundle 是 CCR 的核心数据结构——它打包了云端执行所需的一切：

```
bundle/
├── .claude/
│   ├── settings.json       — 权限配置
│   └── memory.md           — 项目记忆
├── src/                    — 代码快照
├── package.json            — 依赖声明
├── CLAUDE.md               — 项目指令
└── .git/                   — Git 状态（当前分支、HEAD）
```

**关键点**：Bundle 不是简单的文件压缩包，它包含完整的 Git 状态——云端可以根据 HEAD 检出相同的代码版本，然后应用本地的未提交变更。

```typescript
// Bundle 创建（简化伪码）
async function createBundle(): Promise<Bundle> {
  return {
    gitState: {
      remote: getRemoteUrl(),
      branch: getCurrentBranch(),
      head: getHeadCommit(),
      patches: getUncommittedPatches()  // 未提交的变更
    },
    config: {
      settings: readSettings('.claude/settings.json'),
      memory: readFile('CLAUDE.md'),
      permissions: getPermissionRules()
    },
    context: {
      // 不传输完整代码，云端从 Git 拉取
      // 只传输未提交的补丁
    }
  }
}
```

---

## 分支隔离：云端代理在专用分支上工作

远程代理不会直接推送到你的分支——它创建一个专用的远程分支：

```
本地分支: feature/auth-redesign
        ↓ 触发远程任务
远程分支: feature/auth-redesign-remote-001
        ↓ 执行完毕
本地审查: git fetch && git diff feature/auth-redesign...feature/auth-redesign-remote-001
```

这保证了：
- 本地分支不受远程执行影响
- 可以选择性合并远程结果
- 多个远程任务可以并行执行

---

## remote_review 工作流

CCR 的审查流程有一个特殊标签——`remote_review`：

```
┌─────────────────────────────────────────┐
│          remote_review 工作流            │
│                                         │
│  云端执行 ──→ 结果回传 ──→ 本地审查      │
│                         │               │
│                         ▼               │
│              ┌────────────────────┐     │
│              │  标记为 remote_review│     │
│              │  "此代码在云端编写"  │     │
│              │  "需本地人工审查"    │     │
│              └────────────────────┘     │
│                         │               │
│                         ▼               │
│              ┌────────────────────┐     │
│              │  逐文件 diff 审查   │     │
│              │  合并 / 修改 / 拒绝  │     │
│              └────────────────────┘     │
└─────────────────────────────────────────┘
```

**源码**：`cli.js ~line 86701` — CCR 会话管理，`~line 85999` — 策略门控

### 策略门控

远程会话有严格的访问控制：

```json
{
  "policy": {
    "allow_remote_sessions": true,     // 是否允许远程会话
    "remote_session_timeout": 3600,    // 超时（秒）
    "allowed_triggers": ["build", "test"],  // 允许的触发器类型
    "require_review": true             // 是否强制本地审查
  }
}
```

---

## 会话复用：连接与断开

远程会话不是一次性的——可以连接、断开、重连：

```
本地: 启动远程任务 → 断开连接 → 继续本地工作
     （30 分钟后）
本地: 重新连接 → 检查进度 → 获取结果
```

```typescript
// 连接远程会话
const session = await connectRemoteSession(sessionId)
console.log(`进度: ${session.progress}%`)
console.log(`已完成: ${session.completedSteps}/${session.totalSteps}`)

// 断开（不取消任务）
await disconnectRemoteSession(sessionId)
```

---

## 同一份提示词，不同的法场

CCR 最深刻的设计哲学：**系统提示词是可移植的**。

```
本地环境:                        云端环境:
┌────────────────────┐          ┌────────────────────┐
│ system prompt:     │   ===    │ system prompt:     │
│ "You are Claude    │   完全   │ "You are Claude    │
│  Code, an expert   │   相同   │  Code, an expert   │
│  coding agent..."  │          │  coding agent..."  │
│                    │          │                    │
│ Tools:             │   ===    │ Tools:             │
│ Bash, Read, Write  │   完全   │ Bash, Read, Write  │
│ Grep, Glob, Edit   │   相同   │ Grep, Glob, Edit   │
│                    │          │                    │
│ Permissions:       │   ===    │ Permissions:       │
│ allow: npm test    │   完全   │ allow: npm test    │
│ deny: rm -rf /     │   相同   │ deny: rm -rf /     │
└────────────────────┘          └────────────────────┘

执行环境:                        执行环境:
┌────────────────────┐          ┌────────────────────┐
│ MacBook Pro        │          │ 云端服务器           │
│ 16GB RAM           │    ≠     │ 64GB RAM           │
│ M2 芯片            │          │ x86_64             │
│ 本地网络           │          │ 高速内网            │
└────────────────────┘          └────────────────────┘
```

这证明了一件事：**好的系统提示词不依赖于特定的执行环境**。就像一份法律，可以在任何法庭生效——法庭的规模不同（本地 vs 云端），但法律条文（提示词）是同一份。

---

## 关键洞察

系统提示词的可移植性证明了"提示词即程序"——同一份指令，不同的机器，相同的逻辑。
